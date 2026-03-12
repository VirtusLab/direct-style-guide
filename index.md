# Direct Style Scala 3: A Practical Guide

A hands-on guide to writing applications using Scala 3, Java 25 virtual threads,
and direct style. Each chapter is a self-contained use-case using
[Tapir](https://tapir.softwaremill.com), [Ox](https://ox.softwaremill.com), and
the direct-style ecosystem. The patterns and code shown are starting points —
adapt them to your application's specific requirements.

## Chapters

- [Resource Management](01-resource-management.md) — Tying resource lifecycle to
  Ox concurrency scopes. Covers `useInScope`, `useCloseableInScope`,
  reverse-order release, and how `OxApp` ensures cleanup on shutdown signals.

- [Background Processes](02-background-processes.md) — Running long-lived
  background tasks within Ox's structured concurrency scopes. Covers `OxApp` as
  the application entry point, `fork`/`forkUser` for daemon vs. user threads,
  `forever`/`sleep` for periodic loops, and how scope termination triggers
  orderly shutdown.

- [Type-Safe Configuration](03-type-safe-configuration.md) — Reading HOCON
  configuration into Scala 3 case classes using PureConfig. Covers `derives
  ConfigReader`, environment variable overrides, the `Sensitive` wrapper for
  secrets, and validation at load time.

- [Compile-Time Dependency Injection](04-compile-time-dependency-injection.md) —
  Wiring the application's dependency graph at compile time using MacWire.
  Covers `autowire` for automatic constructor resolution, `autowireMembersOf`
  for config extraction, `wireList` for collecting endpoints, and how the same
  wiring is used in production and tests.

- [Error Handling](05-error-handling.md) — Representing application errors as
  `Either` values using Ox's `either` blocks and `.ok()` short-circuiting.
  Covers the `Fail` ADT pattern, composing validations, integrating `Either`
  with database transactions (`transactEither`), converting exceptions with
  `.catching`, and the nesting rules for `either` blocks.

- [Error Output Customisation](06-error-output-customisation.md) — Making all
  error responses (including decode failures and 404s) return consistent JSON.
  Covers defining a bidirectional `Fail` → HTTP status code mapping, the
  `failOutput` Tapir output, and customising `defaultHandlers` in server
  options.

- [Decode Failure Handling](07-decode-failure-handling.md) — Customising how
  Tapir handles requests that can't be decoded. Covers the
  `DefaultDecodeFailureHandler` structure, the respond/message/response pipeline,
  per-input routing control with `onDecodeFailureNextEndpoint`, custom failure
  messages, and hiding authenticated endpoints.

- [Authentication](08-authentication.md) — Implementing bearer-token
  authentication with Tapir's security model: defining secured endpoints,
  writing a generic token-validation layer, and wiring it into the HTTP server.
  Covers `secureEndpoint[T]`, the `AuthTokenOps[T]` trait, the `Auth[T]`
  authenticator class, and `handleSecurity` for connecting auth to endpoint
  logic.

- [HTTP Server Configuration](09-http-server-configuration.md) — Configuring the
  HTTP server for production use. Covers security headers (clickjacking
  prevention), CORS setup, serving static files for single-page applications,
  and request cancellation handling for database-backed services.

- [SQL Persistence](10-sql-persistence.md) — Implementing database access with
  Magnum and PostgreSQL. Covers table-mapped case classes, custom `DbCodec`
  instances for opaque types, the `Repo`/`TableInfo` pattern, type-safe SQL
  interpolation, Flyway migrations, and HikariCP with virtual threads.

- [Version API](11-version-api.md) — Exposing build metadata via an HTTP
  endpoint using sbt-buildinfo. Covers generating a `BuildInfo` object at
  compile time with the git commit hash, and serving it from a simple Tapir
  endpoint.

- [Sending Emails](12-sending-emails.md) — Asynchronous email delivery with
  transactional scheduling and background batch processing. Covers the
  `EmailScheduler` trait, pluggable senders (SMTP, Mailgun, dummy), email
  templates, and the `fork`/`forever` pattern for periodic background tasks.

- [Compile-Time OpenAPI Generation](13-compile-time-openapi-generation.md) —
  Not the usual runtime Swagger UI approach — this chapter covers generating
  OpenAPI YAML at build time from Tapir endpoint descriptions, for use as input
  to frontend client code generation. Covers the `EndpointsForDocs` trait for
  collecting endpoints, the `@main` generator method, and wiring it as an sbt
  task.

- [Testing HTTP Endpoints](14-testing-http-endpoints.md) — Testing HTTP
  endpoints without starting a server using sttp's stub backend and Tapir's
  `SttpClientInterpreter`. Covers setting up `TapirSyncStubInterpreter`,
  converting endpoint descriptions into type-safe HTTP requests, and testing
  both public and secured endpoints.

- [OpenTelemetry Observability](15-opentelemetry-observability.md) — Adding
  tracing, metrics, and log correlation to a direct-style application using
  OpenTelemetry. Covers SDK auto-configuration, Tapir server interceptors for
  tracing and metrics, sttp client instrumentation, custom business metrics, JVM
  runtime observers, propagating trace context across virtual threads with Ox,
  and correlating logs via MDC.

- [Kafka Streaming](16-kafka-streaming.md) — Consuming and producing Kafka
  messages using Ox Flows. Covers `KafkaFlow.subscribe`, `mapPar` for concurrent
  processing, publishing with `KafkaDrain`, offset commits, transactional
  produce-and-commit, and integration with structured concurrency for graceful
  shutdown.
