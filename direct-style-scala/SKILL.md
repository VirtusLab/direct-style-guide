---
name: direct-style-scala
description: This skills covers direct-style Scala, Ox structured concurrency, synchronous Tapir. Auto-load when writing Scala code, applications that use Ox, direct-style, or synchronous Tapir interpreters.
---

You are an expert backend software engineer and architect.

# Scala tooling

* ALWAYS use tools to compile and run tests instead of relying on bash commands
* after adding a dependency to `build.sbt`, ALWAYS run the `import-build` tool
* to lookup a dependency or the latest version, use the `find-dep` tool
* to lookup the API of a class, use the `inspect` tool. To lookup the docs or
  usages, use the `get-docs` and `get-usages` tools
* to compile the project, use `compile-full`, `compile-module` tools
* to search for symbols, use `glob-search` and `typed-glob-search` tools
* if you do need to use `sbt`, use `sbt --client` instead of `sbt` to connect to
  a running sbt server for faster execution
* to verify that the app starts use `sbt run`, WITHOUT `--client`, as it
  prevents interrupting the process
* before committing, ALWAYS format all changed Scala files using the sbt
  `scalafmt` plugin: `sbt --client scalafmtAll`
* the project MUST compile with zero warnings. Ensure `build.sbt` includes
  `-Wunused:all`, `-Wvalue-discard`, `-Wnonunit-statement` in `scalacOptions`.
  Fix warnings in code; only use `@nowarn` for generated code or unfixable
  third-party issues (with a comment explaining why).

# Coding style

* avoid using `{}` and use braceless syntax
* take care of good naming; responsibilities in code should be segregated
  between appropriately named entities
* when dealing with resources, properly track who owns which resources, and
  ensure proper ordering on cleanup
* when possible, restrict visibility of classes and top-level constructs to
  appropriate sub-packages. No need to restrict visibility to the main package.
* it's fine to create multiple classes in one file, especially if they are used
  only by that class
* comment on any aspects that aren't obvious from the implementation, but are
  important to know when reading the code
* write small, focused functions. Introduce abstractions only when they reduce
  duplication or clarify intent.
* write targeted tests, ensuring that each test covers exactly one functionality.
  Make sure that there are no overlaps, that is that no test is redundant.
* every public function, val, and given MUST have an explicit return type. This
  prevents accidental signature drift during refactors.

# Functional programming

* prefer pure functions, immutable data, higher-order functions, ADTs. Avoid
  loops and mutable state; never use shared mutable state.
* APIs must be **lawful**: given identical arguments and explicit dependencies,
  they yield the same observable result. Do not hide dependencies like `Clock`,
  `Random`, or `UUID` inside methods — pass them explicitly or capture them in
  the class constructor:

```scala
// Wrong — hidden non-determinism:
class OrderService:
  def place(order: Order): Confirmation =
    val id = UUID.randomUUID()
    val now = Instant.now()
    Confirmation(id, now)

// Right — dependencies are explicit and injectable:
class OrderService(clock: Clock, idGenerator: () => UUID):
  def place(order: Order): Confirmation =
    val id = idGenerator()
    val now = clock.instant()
    Confirmation(id, now)
```

## Domain types over primitives

* avoid `String`, `Int`, `Long`, and `Boolean` for domain values. Wrap them in
  opaque types, enums, or small case classes so the compiler prevents mixing
  them up (e.g. `UserId`, `Email`, `Port`).
* avoid boolean parameters and boolean fields — they erase meaning at the call
  site. Use enums or named alternatives instead:

```scala
// Wrong — what does `true` mean?
case class State(key: String, dirty: Boolean)

// Right — self-documenting:
enum SyncStatus:
  case Clean, Dirty
case class State(key: String, syncStatus: SyncStatus)
```

## Truthful signatures

* if a method can fail, its return type MUST reflect that — return
  `Either[E, T]`, not `T` with a hidden exception.
* if a value can be absent, use `Option[T]`, not `null` or a sentinel value.
* model different states of an entity as separate types rather than using
  `Option` fields and hoping callers check.

## Make illegal states unrepresentable

* design domain models so that invalid data cannot be constructed. Use enums,
  opaque types, smart constructors, or refined types to encode invariants.

## Error handling

* return `Either[E, A]` for expected, recoverable failures (domain-level
  errors). Throw exceptions only for defects (bugs, unrecoverable failures).
  See the guide's [Error Handling](200-error-handling.md) chapter for
  composition patterns.
* define sealed-trait or enum error hierarchies — avoid stringly-typed errors:

```scala
// Wrong — stringly-typed, no exhaustiveness checking:
case class AppError(code: String, message: String)
Left(AppError("NOT_FOUND", "User 42 not found"))

// Right — compiler-checked, pattern-matchable:
enum AppError:
  case NotFound(entity: String, id: String)
  case InvalidInput(field: String, reason: String)
Left(AppError.NotFound("user", "42"))
```
* **never use bare `try`/`catch` for expected failures**. Reserve `try`/`catch`
  for defect boundaries only (e.g. `catch NonFatal` at the top of a processing
  loop to log and continue). To convert exceptions from Java/third-party APIs
  into `Either`, see the guide's [Error Handling](200-error-handling.md)
  chapter.
* `Option` is not for errors — use it only for presence/absence where absence
  is perfectly ordinary.

# Direct Style Scala: A Practical Guide

BEFORE writing any code that uses Tapir, Ox, sttp, or direct-style Scala, you MUST 
fetch the chapter(s) relevant to your current task from this guide and follow the 
patterns shown there. This is not optional — code that ignores guide patterns will 
be rejected in review.

When fetching, you MUST request the COMPLETE content — every code block, every
paragraph. Summaries lose critical details (e.g. required factory overrides,
specific API calls). Use a prompt like: "Return the COMPLETE raw content. Every
line, every code block. Do not summarize or omit anything."

Every API, pattern, and constraint described in the fetched chapter MUST be
followed. If the chapter says to use a specific API (e.g. `useInScope` for
resource management), do NOT substitute a different approach. If the chapter
marks something as required, it is required.

To fetch a chapter, use the base URL below followed by the chapter filename
listed in the index that follows:
https://raw.githubusercontent.com/VirtusLab/direct-style-guide/refs/heads/master/

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

- [Functional Patterns in Direct Style](140-functional-patterns.md) — Immutable
  state with scoped `var` mutation, pure state transitions via `.copy()`,
  `foldLeft` accumulation, handling failures with `either:`, testing without
  mocks, trait-based effect boundaries.

- [Concurrency and Inter-Fiber Communication](150-shared-state-across-fibers.md)
  — Channels for signaling and data exchange, actors for serialized mutable
  state, `AtomicReference` as a last resort.

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
