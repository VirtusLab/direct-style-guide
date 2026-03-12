# Compile-Time OpenAPI Generation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-openapi-docs"` — OpenAPI model
  generation from endpoint descriptions
- `"com.softwaremill.sttp.apispec" %% "openapi-circe-yaml"` — serialise OpenAPI
  model to YAML

---

## Why generate at build time

Tapir can generate OpenAPI documentation at runtime (via `SwaggerInterpreter`),
but generating the spec at build time has a distinct advantage: the OpenAPI YAML
file can be consumed by frontend builds to generate typed HTTP clients. This
makes the API contract a build-time dependency rather than a runtime one.

## Collecting endpoints for docs

Tapir endpoint descriptions are data structures. The same description used to
implement server logic can also generate OpenAPI documentation. Each API module
exposes its endpoints for docs via the `EndpointsForDocs` trait:

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

`wireList` is a MacWire macro that collects all values matching the element type
— see [Compile-Time Dependency
Injection](04-compile-time-dependency-injection.md).

All endpoint lists are merged in a single place:

```scala
object Dependencies:
  val endpointsForDocs: List[AnyEndpoint] =
    List(UserApi, PasswordResetApi, VersionApi).flatMap(_.endpointsForDocs)
```

## The generator

A standalone `@main` method converts the endpoint descriptions to YAML and
writes the file:

```scala
object OpenAPIDescription:
  val Title = "My API"
  val Version = "1.0"

@main def writeOpenAPIDescription(path: String): Unit =
  val yaml = OpenAPIDocsInterpreter()
    .toOpenAPI(Dependencies.endpointsForDocs, OpenAPIDescription.Title, OpenAPIDescription.Version)
    .toYaml
  Files.writeString(Paths.get(path), yaml)
```

This runs without starting the HTTP server. It only needs the endpoint
descriptions and the OpenAPI interpreter — no server dependencies, no runtime
configuration.

## Wiring as an sbt task

In `build.sbt`, the generator is exposed as a task that can be called from the
build or CI:

```scala
generateOpenAPIDescription := Def.taskDyn {
  val targetPath = ((Compile / target).value / "openapi.yaml").toString
  Def.task {
    (Compile / runMain).toTask(
      s" com.softwaremill.myapp.writeOpenAPIDescription $targetPath"
    ).value
  }
}.value
```

`Def.taskDyn` is needed because `runMain` returns a dynamic task. The generated
`openapi.yaml` is written to the `target` directory, where the frontend build
can pick it up to generate typed API stubs.

Because the generator uses the same `endpointsForDocs` list as the runtime
Swagger UI (if enabled), the generated spec is always consistent with the
running server.
