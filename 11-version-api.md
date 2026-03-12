# Version API

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
- sbt plugin: `sbt-buildinfo` — generates a Scala object with build metadata at
  compile time

---

## Build info generation

The `sbt-buildinfo` plugin generates a `BuildInfo` object during compilation
that contains project metadata:

```scala
buildInfoKeys := Seq[BuildInfoKey](
  name,
  version,
  scalaVersion,
  sbtVersion,
  action("lastCommitHash") {
    import scala.sys.process._
    Try("git rev-parse HEAD".!!.trim).getOrElse("?")
  }
),
buildInfoOptions += BuildInfoOption.ToJson,
buildInfoOptions += BuildInfoOption.ToMap,
buildInfoPackage := "com.softwaremill.bootzooka.version",
buildInfoObject := "BuildInfo"
```

This generates `com.softwaremill.bootzooka.version.BuildInfo` with fields like
`name`, `version`, `scalaVersion`, `sbtVersion`, and `lastCommitHash`. The
`ToMap` option adds a `toMap` method (used for logging at startup), and `ToJson`
adds a `toJson` method.

The `lastCommitHash` is a custom key that runs `git rev-parse HEAD` at compile
time to capture the current commit SHA.

## The version endpoint

The endpoint is minimal — a GET that returns the commit hash:

```scala
class VersionApi extends ServerEndpoints:
  import VersionApi.*

  private val versionServerEndpoint: ServerEndpoint[Any, Identity] =
    versionEndpoint.handleSuccess { _ =>
      Version_OUT(BuildInfo.lastCommitHash)
    }

  override val endpoints = wireList

object VersionApi extends EndpointsForDocs:
  private val versionEndpoint = baseEndpoint.get
    .in("admin" / "version")
    .out(jsonBody[Version_OUT])

  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("admin"))

  case class Version_OUT(buildSha: String) derives ConfiguredJsonValueCodec, Schema
```

`handleSuccess` is used instead of `handle` because this endpoint cannot fail —
it always returns the build hash. There's no `Either` in the return type.

The endpoint is tagged as `"admin"` for grouping in the generated API docs.
