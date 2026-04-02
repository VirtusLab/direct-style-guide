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
- `"io.opentelemetry.instrumentation" % "opentelemetry-runtime-telemetry-java17"`
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
import io.opentelemetry.instrumentation.runtimemetrics.java17.RuntimeMetrics
import io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender

private def initializeOtel(): OpenTelemetry =
  AutoConfiguredOpenTelemetrySdk
    .initialize()
    .getOpenTelemetrySdk()
    .tap(otel => RuntimeMetrics.create(otel).discard)
    .tap(OpenTelemetryAppender.install)
```

Creates a configured `OpenTelemetry` instance from `OTEL_*` environment
variables (noop when unset), registers JVM runtime metric observers, and hooks
Logback into OpenTelemetry for log export.

## Server-side tracing

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

> **Important:** `OpenTelemetryTracing(otel)` must be **prepended** so it
> extracts the `traceparent` header and creates a span before any other
> interceptor runs. Appending it would leave other interceptors without trace
> context.

## Server-side metrics

```scala
.metricsInterceptor(OpenTelemetryMetrics.default[Identity](otel).metricsInterceptor())
```

`OpenTelemetryMetrics.default` registers standard HTTP server metrics:
- `http.server.request.duration` — histogram of request durations
- `http.server.request.total` — counter of total requests
- `http.server.active_requests` — up-down counter of in-flight requests

## Client-side instrumentation

Outgoing HTTP calls are instrumented by wrapping sttp's backend:

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

Wrappers compose innermost first: `OpenTelemetryTracingBackend` creates child
spans and injects `traceparent`, `OpenTelemetryMetricsBackend` records client
metrics.

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

```scala
private val registerUserServerEndpoint = registerUserEndpoint.handle { data =>
  val apiKeyResult = db.transactEither(
    userService.registerNewUser(data.login, data.email, data.password)
  )
  metrics.registeredUsersCounter.add(1)
  apiKeyResult.map(apiKey => Register_OUT(apiKey.id.toString))
}
```

In tests, `OpenTelemetry.noop()` is passed, so metric calls become no-ops.

## Trace context propagation with Ox

> **Required:** When using OpenTelemetry, you **must** configure
> `PropagatingVirtualThreadFactory` in `OxApp`. Without it, forked threads have
> an empty context, producing orphaned spans and logs without trace correlation.

`PropagatingVirtualThreadFactory` from the `ox otel-context` module propagates
OpenTelemetry context to virtual threads created by Ox:

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

## Log correlation via MDC

`InheritableMDC` from Ox propagates MDC values across virtual threads (standard
`ThreadLocal`-based MDC breaks with virtual threads):

```scala
object Main extends OxApp.Simple:
  InheritableMDC.init
```

> **Required:** `InheritableMDC.init` must be called at the class level (not
> inside `run`), so it runs during class initialization before any MDC usage.
> Without it, MDC values silently don't propagate across virtual threads.

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

Every log line produced while handling a request includes the trace ID. In a
Logback pattern like `%d [%X{traceId}] %msg%n`:

```
2025-03-10 12:34:56 [abc123def456] Registering new user: john@example.com
2025-03-10 12:34:56 [abc123def456] Creating a new api key for user 789
```

The same trace ID appears in exported spans, enabling trace-to-log correlation.

