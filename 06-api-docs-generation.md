# API Docs Generation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-swagger-ui-bundle"` — Swagger UI served as server endpoints
- `"com.softwaremill.sttp.apispec" %% "openapi-circe-yaml"` — serialise OpenAPI model to YAML (for file generation)

---

## Endpoints for documentation

Tapir endpoint descriptions are data structures. The same description used to implement server logic can also generate OpenAPI documentation. Each API module exposes its endpoints for docs via the `EndpointsForDocs` trait:

```scala
trait EndpointsForDocs:
  def endpointsForDocs: List[AnyEndpoint]
```

Each module tags its endpoints for grouping in the docs:

```scala
object UserApi extends EndpointsForDocs:
  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("user"))

object PasswordResetApi extends EndpointsForDocs:
  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("password-reset"))
```

`wireList` is a MacWire macro that collects all values matching the element type — see [Compile-Time Dependency Injection](07-compile-time-dependency-injection.md).

All endpoint lists are merged in `Dependencies`:

```scala
object Dependencies:
  val endpointsForDocs: List[AnyEndpoint] =
    List(UserApi, PasswordResetApi, VersionApi).flatMap(_.endpointsForDocs)
```

## Serving Swagger UI at runtime

`SwaggerInterpreter` generates server endpoints that serve the Swagger UI and the OpenAPI spec:

```scala
val docsEndpoints = SwaggerInterpreter(
  swaggerUIOptions = SwaggerUIOptions.default.copy(contextPath = List("api", "v1"))
).fromEndpoints[Identity](endpointsForDocs, "Bootzooka", "1.0")
```

This produces server endpoints that serve the interactive Swagger UI at `/api/v1/docs`. The `contextPath` must match the path prefix where the API endpoints are mounted, so that the "Try it out" feature sends requests to the correct URLs.

The docs endpoints are added alongside the API endpoints:

```scala
val apiEndpoints =
  (serverEndpoints ++ docsEndpoints).map(se =>
    se.prependSecurityIn(apiContextPath.foldLeft(emptyInput: EndpointInput[Unit])(_ / _))
  )
```

## Generating OpenAPI YAML at build time

For frontends that need the API spec during their build (e.g., to generate typed HTTP clients), the spec can be written to a file:

```scala
object OpenAPIDescription:
  val Title = "Bootzooka"
  val Version = "1.0"

@main def writeOpenAPIDescription(path: String): Unit =
  val yaml = OpenAPIDocsInterpreter()
    .toOpenAPI(Dependencies.endpointsForDocs, OpenAPIDescription.Title, OpenAPIDescription.Version)
    .toYaml
  Files.writeString(Paths.get(path), yaml)
```

This is a standalone `@main` method — it runs without starting the HTTP server. The `endpointsForDocs` list is the same one used by the runtime Swagger UI, so the generated file is always consistent.

In `build.sbt`, this is wired as an sbt task:

```scala
generateOpenAPIDescription := Def.taskDyn {
  val targetPath = ((Compile / target).value / "openapi.yaml").toString
  Def.task {
    (Compile / runMain).toTask(
      s" com.softwaremill.bootzooka.writeOpenAPIDescription $targetPath"
    ).value
  }
}.value
```

The generated `openapi.yaml` is then consumed by the frontend build to generate typed API stubs.
