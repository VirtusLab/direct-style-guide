# OpenTelemetry Observability

## Dependencies

- `"io.opentelemetry" % "opentelemetry-sdk-extension-autoconfigure"` —
  auto-configures the OpenTelemetry SDK from environment variables
- `"io.opentelemetry" % "opentelemetry-exporter-otlp"` — exports traces,
  metrics, and logs via OTLP protocol
- `"io.opentelemetry" % "opentelemetry-exporter-sender-jdk"` — uses JDK's
  `HttpClient` for OTLP transport (replaces the default OkHttp sender)
- `"com.softwaremill.sttp.tapir" %% "tapir-opentelemetry-tracing"` — Tapir
  server interceptor that creates spans for incoming HTTP requests
- `"com.softwaremill.sttp.tapir" %% "tapir-opentelemetry-metrics"` — Tapir
  server interceptor that records HTTP request metrics
- `"com.softwaremill.sttp.client4" %% "opentelemetry-backend"` — wraps an sttp
  HTTP client to propagate trace context and record outgoing request metrics
- `"com.softwaremill.ox" %% "otel-context"` — propagates OpenTelemetry context
  across Ox's virtual thread scopes
- `"io.opentelemetry.instrumentation" % "opentelemetry-runtime-telemetry-java8"`
  — JVM runtime metrics (CPU, memory, GC, threads, classes)
- `"io.opentelemetry.instrumentation" % "opentelemetry-logback-appender-1.0"` —
  sends Logback log records to OpenTelemetry
- `"com.softwaremill.ox" %% "mdc-logback"` — MDC values that propagate across
  virtual threads within Ox scopes

---

## SDK initialization

The OpenTelemetry SDK is configured using the auto-configuration module, which
reads settings from environment variables (`OTEL_SERVICE_NAME`,
`OTEL_EXPORTER_OTLP_ENDPOINT`, etc.):

```scala
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk
import io.opentelemetry.instrumentation.runtimemetrics.java8.*
import io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender

private def initializeOtel(): OpenTelemetry =
  AutoConfiguredOpenTelemetrySdk
    .initialize()
    .getOpenTelemetrySdk()
    .tap { otel =>
      Classes.registerObservers(otel)
      Cpu.registerObservers(otel)
      MemoryPools.registerObservers(otel)
      Threads.registerObservers(otel)
      GarbageCollector.registerObservers(otel, false).discard
    }
    .tap(OpenTelemetryAppender.install)
```

This does three things:

1. **SDK setup** — `AutoConfiguredOpenTelemetrySdk.initialize()` creates a fully
   configured `OpenTelemetry` instance. Without `OTEL_*` environment variables,
   it defaults to noop exporters (no data is sent anywhere). In production, you
   set `OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318` to send telemetry to
   a collector.

2. **JVM runtime metrics** — The `opentelemetry-runtime-telemetry-java8` library
   provides pre-built observers for class loading, CPU usage, memory pools,
   thread counts, and garbage collection. These are registered once at startup
   and automatically report metrics at the SDK's configured interval.

3. **Logback integration** — `OpenTelemetryAppender.install` hooks into Logback
   so that log records are sent as OpenTelemetry log signals. This means logs,
   traces, and metrics all flow through the same pipeline and can be correlated
   in a backend like Grafana or Jaeger.

### OTLP transport

The build explicitly excludes the default OkHttp-based OTLP sender and replaces
it with the JDK `HttpClient` sender:

```scala
"io.opentelemetry" % "opentelemetry-exporter-otlp" % otelVersion
  exclude ("io.opentelemetry", "opentelemetry-exporter-sender-okhttp"),
"io.opentelemetry" % "opentelemetry-exporter-sender-jdk" % otelVersion,
```

This avoids pulling in OkHttp as a dependency and instead uses the HTTP client
built into JDK 11+. One fewer dependency, and consistent with the direct-style
philosophy of using JDK facilities.

## Server-side tracing

Tapir's server interceptors form a pipeline that processes every request. The
tracing interceptor creates an OpenTelemetry span for each incoming HTTP
request:

```scala
import sttp.shared.Identity

val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .prependInterceptor(OpenTelemetryTracing(otel))
  .prependInterceptor(SetTraceIdInMDCInterceptor)
  .defaultHandlers(...)
  .corsInterceptor(CORSInterceptor.default[Identity])
  .metricsInterceptor(OpenTelemetryMetrics.default[Identity](otel).metricsInterceptor())
  .options
```

`OpenTelemetryTracing(otel)` is **prepended** — it runs first in the interceptor
chain. This is important: it extracts the trace context from incoming request
headers (W3C `traceparent` format) and creates a span before any other
interceptor runs. This way, all subsequent processing (including business logic)
happens within the span's context.

The span is automatically populated with:
- HTTP method (`http.request.method`)
- URL path (`url.path`)
- Response status code (`http.response.status_code`)
- Request/response sizes

If the incoming request carries a `traceparent` header (e.g., from another
service), the span is created as a child of that trace, enabling distributed
tracing across services.

### Interceptor ordering

The final interceptor chain is assembled by `CustomiseInterceptors` in a fixed
order:

1. **Prepended interceptors** — in the order `prependInterceptor` is called
2. **Metrics** — `OpenTelemetryMetrics`
3. **Exception/logging handlers** — (from `defaultHandlers`)
4. **CORS** — handles preflight requests
5. **Reject handler** — 404 for unmatched requests (from `defaultHandlers`)
6. **Decode failure handler** — error formatting (from `defaultHandlers`)

The `prependInterceptor` calls maintain their call order — `OpenTelemetryTracing`
runs before `SetTraceIdInMDCInterceptor` because it's prepended first.
`.defaultHandlers(...)` customises the behaviour of the exception, reject, and
decode failure handlers, but doesn't change where they sit in the chain — their
positions are fixed by the framework.

## Server-side metrics

```scala
.metricsInterceptor(OpenTelemetryMetrics.default[Identity](otel).metricsInterceptor())
```

`OpenTelemetryMetrics.default` registers standard HTTP server metrics:
- `http.server.request.duration` — histogram of request durations
- `http.server.active_requests` — gauge of in-flight requests
- `http.server.request.body.size` / `http.server.response.body.size` —
  request/response sizes

These are recorded using the OpenTelemetry Meter API and exported alongside the
SDK's other metrics. The `[Identity]` type parameter indicates synchronous
(direct-style) execution — no `IO` monad or `Future`.

## Client-side instrumentation

Outgoing HTTP calls (e.g., to external APIs) are instrumented by wrapping sttp's
backend:

```scala
val sttpBackend = Slf4jLoggingBackend(
  OpenTelemetryMetricsBackend(
    OpenTelemetryTracingBackend(
      HttpClientSyncBackend(),
      otel
    ),
    otel
  )
)
```

The backends compose as wrappers, innermost first:

1. `HttpClientSyncBackend()` — the actual HTTP client (JDK `HttpClient`,
   blocking/direct-style)
2. `OpenTelemetryTracingBackend` — creates a child span for each outgoing
   request and injects the `traceparent` header, enabling distributed tracing
3. `OpenTelemetryMetricsBackend` — records client-side HTTP metrics (duration,
   status codes)
4. `Slf4jLoggingBackend` — logs requests/responses via SLF4J

When a Tapir endpoint handler makes an outgoing HTTP call using this backend,
the child span is automatically linked to the server span created by the tracing
interceptor. This produces end-to-end traces showing: client → this service →
downstream service.

## Custom business metrics

Application-specific metrics are defined using the OpenTelemetry Meter API:

```scala
class Metrics(otel: OpenTelemetry):
  private val meter = otel.meterBuilder("bootzooka")
    .setInstrumentationVersion("1.0")
    .build()

  lazy val registeredUsersCounter: LongCounter =
    meter
      .counterBuilder("bootzooka_registered_users_counter")
      .setDescription("How many users registered on this instance since it was started")
      .build()

  lazy val emailQueueGauge: DoubleGauge =
    meter
      .gaugeBuilder("bootzooka_email_queue_gauge")
      .setDescription("How many emails are waiting to be sent")
      .build()
```

Usage in endpoint logic is straightforward — just call the metric:

```scala
private val registerUserServerEndpoint = registerUserEndpoint.handle { data =>
  val apiKeyResult = db.transactEither(
    userService.registerNewUser(data.login, data.email, data.password)
  )
  metrics.registeredUsersCounter.add(1)
  apiKeyResult.map(apiKey => Register_OUT(apiKey.id.toString))
}
```

The `Metrics` class takes the `OpenTelemetry` instance — the same one used for
tracing and server metrics. In tests, `OpenTelemetry.noop()` is passed, so
metric calls become no-ops.

The `meterBuilder("bootzooka")` creates a named meter — this appears as the
"instrumentation scope" in your observability backend, making it easy to
distinguish your application's metrics from library metrics (like the HTTP
server metrics or JVM metrics).

## Trace context propagation with Ox

Virtual threads created within Ox scopes don't automatically inherit the
OpenTelemetry context from the parent thread. This is a problem: if a Tapir
endpoint handler forks work using `ox.fork`, the forked code won't have access
to the current span.

The fix is `PropagatingVirtualThreadFactory` from the `ox otel-context` module:

```scala
import ox.otel.context.PropagatingVirtualThreadFactory

object Main extends OxApp.Simple:
  override protected def settings: Settings =
    Settings.Default.copy(threadFactory = Some(PropagatingVirtualThreadFactory()))

  override def run(using Ox): Unit =
    val deps = Dependencies.create
    deps.emailService.startProcesses()
    deps.httpApi.start().discard
    never
```

`PropagatingVirtualThreadFactory` creates virtual threads that automatically
copy the OpenTelemetry context from the thread that created them. This means:

- When Tapir's tracing interceptor sets up a span on the request-handling
  thread, and the handler forks work using `supervised` / `fork`, the forked
  threads see the same span context.
- Outgoing HTTP calls made from forked threads will correctly produce child
  spans linked to the request span.
- Log statements from forked threads will carry the correct trace ID in the MDC.

Without this factory, forked threads would have an empty context, producing
orphaned spans and logs without trace correlation.

### How it works with `OxApp`

`OxApp` is Ox's application entry point. It manages the top-level `supervised`
scope and handles graceful shutdown. The `settings` override injects the
propagating factory so that *all* virtual threads in the application — not just
those created directly by the application — use context propagation.

## Log correlation via MDC

The MDC (Mapped Diagnostic Context) is a per-thread map of key-value pairs that
logging frameworks include in log output. The challenge with virtual threads:
standard MDC (`ThreadLocal`-based) breaks when work moves between threads.

Ox solves this with `InheritableMDC`, which propagates MDC values across virtual
threads within `supervised` scopes. The application initializes it at startup:

```scala
object Main extends OxApp.Simple:
  InheritableMDC.init
```

A custom Tapir interceptor extracts the trace ID from the current span and sets
it in the MDC:

```scala
object SetTraceIdInMDCInterceptor extends RequestInterceptor[Identity]:
  val MDCKey = "traceId"

  override def apply[R, B](
      responder: Responder[Identity, B],
      requestHandler: EndpointInterceptor[Identity] => RequestHandler[Identity, R, B]
  ): RequestHandler[Identity, R, B] =
    RequestHandler.from { case (request, endpoints, monad) =>
      val traceId = Span.current().getSpanContext().getTraceId()
      InheritableMDC.unsupervisedWhere(MDCKey -> traceId) {
        requestHandler(EndpointInterceptor.noop)(request, endpoints)(using monad)
      }
    }
```

This interceptor:

1. Reads the current span (set by `OpenTelemetryTracing` which runs earlier in
   the chain).
2. Extracts the trace ID string.
3. Wraps the remaining request handling inside
   `InheritableMDC.unsupervisedWhere`, which sets `traceId` in the MDC for the
   duration of the block — including any virtual threads forked within it.

The result: every log line produced while handling a request includes the trace
ID. In a Logback pattern like `%d [%X{traceId}] %msg%n`, you get:

```
2025-03-10 12:34:56 [abc123def456] Registering new user: john@example.com
2025-03-10 12:34:56 [abc123def456] Creating a new api key for user 789
```

The same trace ID appears in the span sent to your tracing backend, enabling you
to click a trace in Jaeger/Grafana and see the corresponding log lines.

### The full observability stack

Putting it all together, a single HTTP request produces:

1. **A trace span** (via `OpenTelemetryTracing`) — with method, path, status
   code, duration
2. **HTTP server metrics** (via `OpenTelemetryMetrics`) — request count,
   duration histogram
3. **Application metrics** (via `Metrics`) — e.g., registered users counter
4. **JVM metrics** (via runtime observers) — CPU, memory, GC, threads
5. **Correlated logs** (via `SetTraceIdInMDCInterceptor` +
   `OpenTelemetryAppender`) — with trace ID
6. **Client spans** (via `OpenTelemetryTracingBackend`) — for any outgoing HTTP
   calls

All telemetry flows through the same `OpenTelemetry` instance, uses the same
OTLP exporter, and shares the same trace context. The direct-style approach
means none of this requires monadic effect types — it's all plain method calls
on virtual threads, with context propagation handled by the thread factory.
