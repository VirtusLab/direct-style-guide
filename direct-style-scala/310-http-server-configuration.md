# HTTP Server Configuration

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend
- `"com.softwaremill.sttp.tapir" %% "tapir-files"` — serving static files

---

## Security headers

Anti-clickjacking headers added to `baseEndpoint`, inherited by all endpoints:

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint
    .errorOut(failOutput)
    .out(header("X-Frame-Options", "DENY"))
    .out(header("Content-Security-Policy", "frame-ancestors 'none'"))
```

These are Tapir output headers — included in every successful response. Error
responses are handled separately by server options — see [Error Output
Customisation](210-error-output-customisation.md).

## CORS

Tapir provides a built-in CORS interceptor:

```scala
val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .corsInterceptor(CORSInterceptor.default[Identity])
  // ... other interceptors
  .options
```

`CORSInterceptor.default[Identity]` allows all origins and common HTTP methods.
For production, restrict the allowed origins:

```scala
import sttp.tapir.server.interceptor.cors.{CORSConfig, CORSInterceptor}

val corsInterceptor = CORSInterceptor.customOrThrow[Identity](
  CORSConfig.default.allowMatchingOrigins(_ == "https://example.com")
)
```

The CORS interceptor is part of the server options interceptor chain (see
[OpenTelemetry Observability](510-opentelemetry-observability.md) for the full
chain ordering). It handles preflight `OPTIONS` requests automatically.

## Serving static files

For single-page applications, the backend serves the frontend's static assets
(HTML, JS, CSS) and falls back to `index.html` for client-side routes:

```scala
import sttp.tapir.files.{FilesOptions, staticResourcesGetServerEndpoint}

val webappEndpoints = List(
  staticResourcesGetServerEndpoint[Identity](emptyInput: EndpointInput[Unit])(
    classOf[HttpApi].getClassLoader,
    "webapp",
    FilesOptions.default[Identity].defaultFile(List("index.html"))
  )
)
```

`defaultFile(List("index.html"))` provides SPA support: unmatched paths return
`index.html`. These endpoints are added after API endpoints (`apiEndpoints ++
webappEndpoints`) so API routes take priority.

## Request cancellation

When a client disconnects mid-request (e.g., a user navigates away), the server
can either interrupt the in-progress handler or let it finish. By default, Tapir
interrupts the handler. For applications using database connections, this can
cause problems:

```scala
val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  // ... interceptors
  .options
  .copy(interruptServerLogicWhenRequestCancelled = false)
```

> **Warning:** `interruptServerLogicWhenRequestCancelled = false` lets
> cancelled requests run to completion. The default (interrupt) can cause
> HikariCP to mark connections as broken when JDBC calls are interrupted
> mid-flight.

## Starting the server

```scala
def start()(using Ox): NettySyncServerBinding =
  NettySyncServer(serverOptions, NettyConfig.default.host(config.host).port(config.port))
    .addEndpoints(allEndpoints)
    .start()
```

`NettySyncServer.start()` takes `(using Ox)` — it registers the server as a
resource in the current scope, stopping it on scope termination.

