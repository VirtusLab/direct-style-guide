# Compile-Time Dependency Injection

## Dependencies

- `"com.softwaremill.macwire" %% "macros" % Provided` — compile-time DI via
  macros (no runtime dependency)

---

> **When to use:** MacWire is most valuable when the main class grows large or
> the dependency graph is complex enough that manual constructor wiring becomes
> error-prone. For small projects with a handful of components, direct
> constructor calls in `Main` are simpler and sufficient.

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
  as individual dependencies.

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


