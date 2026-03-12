# Resource Management

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `useInScope`, `useCloseableInScope`

---

## Resources tied to scope lifecycle

In a direct-style application, resources (database connections, HTTP clients,
SDK instances) need to be released when the application shuts down. Ox ties
resource lifecycle to concurrency scopes: resources are acquired when the scope
starts and released — in reverse order — when the scope ends.

## useInScope and useCloseableInScope

`useInScope` acquires a resource and registers a custom release function:

```scala
supervised {
  val resource = useInScope(acquire())(release)
  // use resource
}
// release is called automatically
```

`useCloseableInScope` is a shorthand for resources that implement
`AutoCloseable`:

```scala
supervised {
  val db = useCloseableInScope(DB.createTestMigrate(config.db))
  // use db
}
// db.close() is called automatically
```

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

The sttp backend is wrapped with `useInScope` and a custom `_.close()` release
function. The database connection pool uses `useCloseableInScope` since `DB`
extends `AutoCloseable`.

When the application receives SIGTERM:
1. The root scope is interrupted
2. All forks (HTTP server, background processes) are interrupted and awaited
3. Resources are released in reverse order: first the database pool, then the
   sttp backend

## Release ordering

Resources are released in reverse acquisition order. This is important because
later resources may depend on earlier ones. For example, the HTTP server (which
uses the database) is stopped before the database connection pool is closed.

The sequence in the example application:
1. **Acquire:** config → OpenTelemetry SDK → sttp backend → database pool
2. **Start:** background processes → HTTP server (via forks)
3. **Shutdown:** interrupt forks → stop HTTP server → stop background processes
   → close database pool → close sttp backend

## OxApp ensures cleanup

`OxApp` is the entry point that makes this work. It installs signal handlers for
SIGINT/SIGTERM and initiates orderly scope termination:

```scala
object Main extends OxApp.Simple:
  override def run(using Ox): Unit =
    val deps = Dependencies.create  // acquires resources via useInScope
    deps.emailService.startProcesses()
    deps.httpApi.start().discard
    never  // blocks until interrupted
```

Without `OxApp`, a `sys.exit()` or unhandled signal would terminate the JVM
without releasing resources. `OxApp` converts these into scope interruptions,
which trigger the cleanup chain.
