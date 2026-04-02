# Compile-Time OpenAPI Generation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-openapi-docs"` — OpenAPI model
  generation from endpoint descriptions
- `"com.softwaremill.sttp.apispec" %% "openapi-circe-yaml"` — serialise OpenAPI
  model to YAML

---

## Collecting endpoints for docs

Each API module's companion object extends `EndpointsForDocs` (see
[Compile-Time Dependency
Injection](130-compile-time-dependency-injection.md)) and tags its endpoints:

```scala
object UserApi extends EndpointsForDocs:
  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("user"))
```

All endpoint lists are merged in a single place:

```scala
object Dependencies:
  val endpointsForDocs: List[AnyEndpoint] =
    List(UserApi, PasswordResetApi, VersionApi).flatMap(_.endpointsForDocs)
```

## The generator

A standalone `@main` function converts the endpoint descriptions to YAML and
writes the file:

```scala
import sttp.apispec.openapi.circe.yaml.*
import sttp.tapir.docs.openapi.OpenAPIDocsInterpreter
import java.nio.file.*

object OpenAPIDescription:
  val Title = "My API"
  val Version = "1.0"

@main def writeOpenAPIDescription(path: String): Unit =
  val yaml = OpenAPIDocsInterpreter()
    .toOpenAPI(Dependencies.endpointsForDocs, OpenAPIDescription.Title, OpenAPIDescription.Version)
    .toYaml
  Files.writeString(Paths.get(path), yaml)
```

## Wiring as an sbt task

In `build.sbt`, define a task key and wire the generator:

```scala
val generateOpenAPIDescription = taskKey[Unit]("Generates OpenAPI YAML from endpoint descriptions")

generateOpenAPIDescription := Def.taskDyn {
  val targetPath = ((Compile / target).value / "openapi.yaml").toString
  Def.task {
    (Compile / runMain).toTask(
      s" com.softwaremill.myapp.writeOpenAPIDescription $targetPath"
    ).value
  }
}.value
```

