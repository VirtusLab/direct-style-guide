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

* ALWAYS use braceless syntax — do not use `{}`
* responsibilities in code MUST be segregated between appropriately named
  entities
* when dealing with resources, properly track who owns which resources, and
  ensure proper ordering on cleanup
* restrict visibility of classes and top-level constructs to appropriate
  sub-packages when possible. No need to restrict visibility to the main
  package.
* multiple classes in one file is acceptable when they are tightly coupled
* comment on any aspects that aren't obvious from the implementation, but are
  important to know when reading the code
* functions MUST be small and focused. Introduce abstractions only when they
  reduce duplication or clarify intent.
* tests MUST be targeted — each test covers exactly one scenario. No
  overlapping or redundant tests.
* every public function, val, and given MUST have an explicit return type — this
  prevents accidental signature drift during refactors:

```scala
// Wrong — inferred return type can silently change:
def findUser(id: Id[User])(using DbTx) =
  userModel.findById(id).toRight(Fail.NotFound("user"))

// Right — return type is explicit and stable:
def findUser(id: Id[User])(using DbTx): Either[Fail, User] =
  userModel.findById(id).toRight(Fail.NotFound("user"))
```

# Performance

* NEVER materialize unbounded data into memory. Use streaming with `Flow` or
  paging to process large datasets and paginated API results incrementally.

# Direct-style Scala

* in Tapir, use `.handle` / `.handleSecurity` / `.handleSuccess` to wire
  endpoint logic — NEVER use `.serverLogic` / `.serverSecurityLogic`. The
  `.handle` family is the direct-style API. The `.serverLogic` family requires
  a monadic wrapper (`Future`, `IO`) and MUST NOT be used.
* only propagate `(using Ox)` when the method genuinely needs to start forks or
  register resources in the caller's scope. Otherwise, create a local
  `supervised` block.

# Functional programming

* use pure functions, immutable data, higher-order functions, ADTs. Avoid loops
  and mutable state. NEVER use shared mutable state.
* APIs MUST be **lawful**: given identical arguments and explicit dependencies,
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

* wrap `String`, `Int`, `Long`, and `Boolean` domain values in opaque types or
  enums — NEVER use raw primitives for domain concepts. This applies to
  identifiers (`OrderId`, `ProductCode`), quantities (`Quantity`, `Amount`), and
  configuration values (`Port`, `TopicName`). When a generated library (e.g.
  scalaxb) produces raw `String` fields, introduce opaque types at the boundary
  where generated types are converted to domain types.
* eliminate boolean blindness — replace `Boolean` parameters and return values
  with two-case enums so intent is explicit and exhaustiveness is checked:

```scala
// Wrong — caller must remember what `true` means:
def recordFlush(success: Boolean, durationMs: Double): Unit

// Right — intent is unambiguous:
enum FlushOutcome:
  case Success, Failure
def recordFlush(outcome: FlushOutcome, duration: Duration): Unit
```
* if a method can fail, its return type MUST be `Either[E, T]` — NEVER throw
  exceptions for recoverable failures. 
* if a value can be absent, use `Option[T]` — NEVER use `null` or sentinel
  values. `Option` is for presence/absence only, not for errors.
* model different states of an entity as separate types — NEVER use `Option`
  fields to represent state transitions:

```scala
// Wrong — callers must remember to check confirmedAt:
case class Order(id: Id[Order], items: List[Item], confirmedAt: Option[Instant])

// Right — the type tells you what state the order is in:
case class PendingOrder(id: Id[Order], items: List[Item])
case class ConfirmedOrder(id: Id[Order], items: List[Item], confirmedAt: Instant)
```

* design domain models so that invalid data CANNOT be constructed. Use enums,
  opaque types, or smart constructors to encode invariants:

```scala
// Wrong — any string is accepted:
def setPort(port: Int): Unit

// Right — invalid values are rejected at construction:
opaque type Port = Int
object Port:
  def apply(value: Int): Either[String, Port] =
    if value >= 1 && value <= 65535 then Right(value)
    else Left(s"Port out of range: $value")
```

* define sealed-trait or enum error hierarchies — NEVER use stringly-typed
  errors.
* NEVER use bare `try`/`catch` for recoverable failures. Reserve `try`/`catch` for
  defect or unrecoverable error boundaries only.

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
https://raw.githubusercontent.com/VirtusLab/scala-skill/refs/heads/master/direct-style-scala/

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

- [Concurrency and Inter-Thread Communication](150-shared-state-across-threads.md)
  — Flows for declarative concurrent pipelines (`mapPar`, `merge`,
  `mapStateful`), channels for imperative send/receive, actors for serialized
  mutable state.

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
