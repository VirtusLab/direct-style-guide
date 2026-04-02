# Resource Management

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `useInScope`, `useCloseableInScope`

---

## useInScope and useCloseableInScope

`useInScope(acquire)(release)` registers a resource and its release function
within the current scope. `useCloseableInScope` is a shorthand for
`AutoCloseable` resources. Resources are released in reverse acquisition order
when the scope ends.

## Application resource setup

The `Dependencies.create` method acquires all resources within the application's
root scope:

```scala
def create(using Ox): Dependencies =
  val config = Config.read.tap(Config.log)
  val otel = initializeOtel()
  val sttpBackend = useInScope(
    Slf4jLoggingBackend(
      OpenTelemetryMetricsBackend(
        OpenTelemetryTracingBackend(HttpClientSyncBackend(), otel),
        otel
      )
    )
  )(_.close())
  val db: DB = useCloseableInScope(DB.createTestMigrate(config.db))

  create(config, otel, sttpBackend, db, DefaultClock)
```

On scope termination (e.g., SIGTERM via `OxApp` — see [Background
Processes](110-background-processes.md)), resources are released in reverse
acquisition order: first the database pool, then the sttp backend.
