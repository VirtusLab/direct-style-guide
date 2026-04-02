# Direct Style Scala 3: A Practical Guide

A reference for LLM agents implementing direct-style Scala 3 applications with
virtual threads. Each chapter covers a single use-case using
[Tapir](https://tapir.softwaremill.com), [Ox](https://ox.softwaremill.com), and
the direct-style ecosystem.

## Application Structure

- [Resource Management](100-resource-management.md) — `useInScope`,
  `useCloseableInScope`, reverse-order release, scope-based cleanup.

- [Background Processes](110-background-processes.md) — `OxApp` entry point,
  `fork`/`forkUser` for daemon vs. user threads, `forever`/`sleep` for periodic
  loops, orderly shutdown.

- [Type-Safe Configuration](120-type-safe-configuration.md) — PureConfig with
  `derives ConfigReader`, environment variable overrides, `Sensitive` wrapper,
  load-time validation.

- [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md)
  — MacWire `autowire`, `autowireMembersOf` for config extraction, `wireList`
  for collecting endpoints.

- [Functional Patterns in Direct Style](140-functional-patterns.md) — Small
  single-responsibility methods, domain types over primitives, truthful
  signatures, making illegal states unrepresentable, immutable state with
  scoped `var` mutation, pure state transitions, trait-based effect boundaries.

- [Shared State Across Fibers](150-shared-state-across-fibers.md) — Single-owner
  pattern with channels, `AtomicReference` with `updateAndGet`, extracting
  return values, choosing the right pattern.

## Error Handling

- [Error Handling](200-error-handling.md) — `Fail` ADT, Ox `either` blocks with
  `.ok()` short-circuiting, `transactEither`, `.catching`, nesting rules.

- [Error Output Customisation](210-error-output-customisation.md) — JSON error
  responses for all error types. Bidirectional `Fail` → HTTP status code
  mapping, `failOutput`, `defaultHandlers` for decode failures and 404s.

- [Decode Failure Handling](220-decode-failure-handling.md) —
  `DefaultDecodeFailureHandler` customisation: respond/message/response pipeline,
  `onDecodeFailureNextEndpoint`, custom failure messages,
  `hideEndpointsWithAuth`.

## HTTP & Endpoints

- [Authentication](300-authentication.md) — `secureEndpoint[T]`,
  `AuthTokenOps[T]` trait, `Auth[T]` authenticator, `handleSecurity` wiring.

- [HTTP Server Configuration](310-http-server-configuration.md) — Security
  headers, CORS, serving static files for SPAs, request cancellation,
  `NettySyncServer` startup.

- [Version API](320-version-api.md) — `sbt-buildinfo` generating `BuildInfo`
  with git commit hash, served from a Tapir endpoint.

- [Compile-Time OpenAPI Generation](330-compile-time-openapi-generation.md) —
  Build-time OpenAPI YAML generation for frontend client codegen (not runtime
  Swagger UI). `EndpointsForDocs`, `@main` generator, sbt task wiring.

- [SOAP with scalaxb](340-soap-with-scalaxb.md) — XSD-to-Scala code generation,
  SOAP envelope wrapping/unwrapping, Tapir XML codecs for scalaxb types,
  `SOAPAction`-based endpoint routing, SOAP fault error handlers.

## Data & Integration

- [SQL Persistence](400-sql-persistence.md) — Magnum with PostgreSQL: `@Table`
  case classes, `DbCodec` for opaque types, `Repo`/`TableInfo`, `sql`
  interpolation, Flyway migrations, HikariCP.

- [Sending Emails](410-sending-emails.md) — `EmailScheduler` trait, pluggable
  senders (SMTP, Mailgun, dummy), email templates, background batch processing.

- [Kafka Streaming](420-kafka-streaming.md) — `KafkaFlow.subscribe`, `mapPar`,
  `KafkaDrain` publishing, offset commits, transactional produce-and-commit,
  graceful shutdown.

## Testing & Observability

- [Testing HTTP Endpoints](500-testing-http-endpoints.md) —
  `TapirSyncStubInterpreter` stub backend, `SttpClientInterpreter` for
  type-safe requests, testing public and secured endpoints in-process.

- [OpenTelemetry Observability](510-opentelemetry-observability.md) — SDK
  auto-configuration, Tapir tracing/metrics interceptors, sttp client
  instrumentation, custom metrics, `PropagatingVirtualThreadFactory` for
  context propagation, MDC log correlation.
