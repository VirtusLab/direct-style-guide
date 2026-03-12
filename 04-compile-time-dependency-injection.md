# Compile-Time Dependency Injection

## Dependencies

- `"com.softwaremill.macwire" %% "macros" % Provided` — compile-time DI via
  macros (no runtime dependency)

---

## Why compile-time DI

Runtime DI frameworks (Spring, Guice) resolve dependencies at application
startup using reflection. Errors — missing bindings, circular dependencies,
wrong types — surface only when the application starts. Compile-time DI detects
these at compilation, and adds no runtime overhead or classpath scanning.

MacWire provides two macros: `autowire` constructs a dependency graph by
matching types, and `wireList` collects all values of a given type into a list.

## The dependency graph

The entire application's dependency graph is constructed in a single
`Dependencies.create` method:

```scala
case class Dependencies(httpApi: HttpApi, emailService: EmailService)

object Dependencies:
  def create(config: Config, otel: OpenTelemetry, sttpBackend: SyncBackend,
             db: DB, clock: Clock): Dependencies =
    autowire[Dependencies](
      autowireMembersOf(config),
      otel,
      sttpBackend,
      db,
      DefaultIdGenerator,
      clock,
      EmailSender.create,
      (apis: Apis, otel: OpenTelemetry, httpConfig: HttpConfig) =>
        new HttpApi(apis.endpoints, Dependencies.endpointsForDocs, otel, httpConfig),
      classOf[EmailService],
      new Auth(_: ApiKeyAuthToken, _: DB, _: Clock),
      new Auth(_: PasswordResetAuthToken, _: DB, _: Clock)
    )
```

`autowire[Dependencies](...)` constructs a `Dependencies` instance by resolving
all transitive dependencies from the provided values. The macro inspects
constructor signatures at compile time and wires them together.

Key patterns:

- **`autowireMembersOf(config)`** — makes all fields of `Config` (which is a
  case class containing `DBConfig`, `HttpConfig`, `EmailConfig`, etc.) available
  as individual dependencies. Instead of passing `config.db`, `config.api`, etc.
  separately, the macro extracts them.

- **Factory functions** — when the default wiring isn't sufficient, you provide
  explicit constructors. The `HttpApi` lambda shows this: it takes specific
  parameters and constructs the instance manually.

- **`classOf[EmailService]`** — tells MacWire which concrete class to
  instantiate when the type alone is ambiguous (e.g., `EmailService` extends
  `EmailScheduler`, and both types might be needed).

- **Disambiguating generic types** — `Auth[T]` is parameterised, so `autowire`
  can't tell `Auth[ApiKey]` from `Auth[PasswordResetCode]`. The explicit `new
  Auth(_: ApiKeyAuthToken, ...)` constructors disambiguate by specifying which
  `AuthTokenOps` implementation to use.

## Collecting endpoints with wireList

Each API class exposes its server endpoints as a list:

```scala
class UserApi(...) extends ServerEndpoints:
  private val registerUserServerEndpoint = ...
  private val loginServerEndpoint = ...
  private val getUserServerEndpoint = ...
  // ...

  override val endpoints = wireList
```

`wireList` collects all values in the current scope that match the list's
element type (`ServerEndpoint[Any, Identity]`). This avoids manually maintaining
a list that must be updated every time an endpoint is added or removed.

Similarly, for documentation endpoints:

```scala
object UserApi extends EndpointsForDocs:
  val registerUserEndpoint = ...
  val loginEndpoint = ...
  // ...

  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("user"))
```

## Infrastructure as constructor parameters

The `Dependencies.create` method takes infrastructure as parameters rather than
constructing it internally:

```scala
def create(config: Config, otel: OpenTelemetry, sttpBackend: SyncBackend,
           db: DB, clock: Clock): Dependencies
```

This is what makes testing possible — tests call the same method with test
implementations:

```scala
Dependencies.create(TestConfig, OpenTelemetry.noop(), HttpClientSyncBackend.stub, currentDb, testClock)
```

The dependency graph is identical in production and tests. Only the leaf
infrastructure (database, clock, HTTP backend, telemetry) differs.

## No runtime dependency

MacWire's `macros` artifact is declared as `Provided` — it's only needed at
compile time:

```scala
"com.softwaremill.macwire" %% "macros" % Provided
```

The generated code is plain Scala constructor calls. There's no reflection, no
classpath scanning, and no MacWire classes in the runtime classpath.
