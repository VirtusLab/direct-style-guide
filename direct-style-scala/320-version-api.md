# Version API

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
- sbt plugin: `sbt-buildinfo` — generates a Scala object with build metadata at
  compile time

---

## Build info generation

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

