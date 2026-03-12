# Authentication

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server with synchronous (direct-style) backend
- `"com.softwaremill.ox" %% "core"` — `sleep` for timing-attack mitigation
- `"com.augustnagro" %% "magnum"` — database access (storing and retrieving API keys in PostgreSQL)
- `"com.password4j" % "password4j"` — password hashing (Argon2)

---

## Base endpoint definitions

Two building blocks are defined for all endpoints in the application. `baseEndpoint` specifies the error output format (see [Error Handling](04-error-handling.md) for the `Fail` ADT and how it maps to HTTP status codes):

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint
    .errorOut(failOutput)

def secureEndpoint[T]: Endpoint[Id[T], Unit, Fail, Unit, Any] =
  baseEndpoint.securityIn(auth.bearer[String]().map(_.asId[T])(_.toString))
```

`secureEndpoint[T]` reads a bearer token from the `Authorization` header and parses it as an `Id[T]` — a phantom-typed string wrapper that prevents mixing up API key IDs with user IDs at the type level.

The type parameter `T` is the token type. For API key–protected endpoints, you use `secureEndpoint[ApiKey]`, which produces `Endpoint[Id[ApiKey], Unit, Fail, Unit, Any]` — the security input is `Id[ApiKey]`.

Endpoints that don't require authentication use `baseEndpoint` directly:

```scala
val loginEndpoint = baseEndpoint.post
  .in("user" / "login")
  .in(jsonBody[Login_IN])
  .out(jsonBody[Login_OUT])
```

Secured endpoints start from `secureEndpoint[ApiKey]`:

```scala
val getUserEndpoint = secureEndpoint[ApiKey].get
  .in("user")
  .out(jsonBody[GetUser_OUT])
```

The request/response body format (here JSON via `jsonBody`) is independent of the authentication mechanism.

## Token operations trait

Authentication is generic over the token type via `AuthTokenOps[T]`:

```scala
trait AuthTokenOps[T]:
  def tokenName: String
  def findById: DbTx ?=> Id[T] => Option[T]
  def delete: DbTx ?=> T => Unit
  def userId: T => Id[User]
  def validUntil: T => Instant
  def deleteWhenValid: Boolean
```

Two things to note about the Scala 3 syntax here:

1. **Context function types** — `DbTx ?=> Id[T] => Option[T]` means "a function from `Id[T]` to `Option[T]` that requires a `DbTx` in context." The `?=>` makes the database transaction an implicit requirement, threaded through automatically when calling within a `transact` block.

2. **`deleteWhenValid`** — controls whether the token is single-use. API keys return `false` (the same key works for many requests). Password-reset codes return `true` (consumed on use).

The `ApiKeyAuthToken` implementation connects the trait to the `ApiKeyModel`:

```scala
class ApiKeyAuthToken(apiKeyModel: ApiKeyModel) extends AuthTokenOps[ApiKey]:
  override def tokenName: String = "ApiKey"
  override def findById: DbTx ?=> Id[ApiKey] => Option[ApiKey] = apiKeyModel.findById
  override def delete: DbTx ?=> ApiKey => Unit = ak => apiKeyModel.delete(ak.id)
  override def userId: ApiKey => Id[User] = _.userId
  override def validUntil: ApiKey => Instant = _.validUntil
  override def deleteWhenValid: Boolean = false
```

## The Auth class

`Auth[T]` is the authenticator. Given a token ID, it returns the authenticated user's ID or an error:

```scala
class Auth[T](authTokenOps: AuthTokenOps[T], db: DB, clock: Clock):
  private val random = SecureRandom.getInstance("NativePRNGNonBlocking")

  def apply(id: Id[T]): Either[Fail.Unauthorized, Id[User]] =
    db.transact(authTokenOps.findById(id)) match
      case None =>
        // random sleep to prevent timing attacks
        sleep(random.nextInt(100).millis)
        Left(Fail.Unauthorized("Unauthorized"))
      case Some(token) if expired(token) =>
        db.transact(authTokenOps.delete(token))
        Left(Fail.Unauthorized("Unauthorized"))
      case Some(token) =>
        if authTokenOps.deleteWhenValid then db.transact(authTokenOps.delete(token))
        Right(authTokenOps.userId(token))

  private def expired(token: T): Boolean = clock.now().isAfter(authTokenOps.validUntil(token))
```

Three cases:
1. **Token not found** — sleep a random 0–100ms before responding. This prevents timing attacks where an attacker measures response times to distinguish "token doesn't exist" from "token exists but is invalid." The `sleep` call is from Ox and blocks the virtual thread without consuming an OS thread.
2. **Token expired** — delete the stale token and return unauthorized.
3. **Token valid** — if it's a one-time token (`deleteWhenValid`), consume it. Return the user ID.

The `apply` method makes `Auth[T]` callable as a function, which is how Tapir's security logic expects it.

## API key model and service

The `ApiKey` case class maps directly to a PostgreSQL table via Magnum's `@Table` annotation:

```scala
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
@SqlName("api_keys")
case class ApiKey(id: Id[ApiKey], userId: Id[User], createdOn: Instant, validUntil: Instant)
```

The model provides basic CRUD. Note the `(using DbTx)` context parameter — all database operations require an active transaction:

```scala
class ApiKeyModel:
  private val apiKeyRepo = Repo[ApiKey, ApiKey, Id[ApiKey]]
  private val a = TableInfo[ApiKey, ApiKey, Id[ApiKey]]

  def insert(apiKey: ApiKey)(using DbTx): Unit = apiKeyRepo.insert(apiKey)
  def findById(id: Id[ApiKey])(using DbTx): Option[ApiKey] = apiKeyRepo.findById(id)
  def delete(id: Id[ApiKey])(using DbTx): Unit = apiKeyRepo.deleteById(id)
  def deleteAllForUser(id: Id[User])(using DbTx): Unit =
    sql"""DELETE FROM $a WHERE ${a.userId} = $id""".update.run().discard
```

`Repo` provides standard CRUD operations. `TableInfo` gives access to table and column metadata for use in custom SQL queries — here `$a` interpolates the table name, and `${a.userId}` the column name.

The service handles key creation with configurable validity:

```scala
class ApiKeyService(apiKeyModel: ApiKeyModel, idGenerator: IdGenerator, clock: Clock):
  def create(userId: Id[User], valid: Duration)(using DbTx): ApiKey =
    val id = idGenerator.nextId[ApiKey]()
    val now = clock.now()
    val validUntil = now.plus(valid.toMillis, ChronoUnit.MILLIS)
    val apiKey = ApiKey(id, userId, now, validUntil)
    apiKeyModel.insert(apiKey)
    apiKey

  def invalidate(id: Id[ApiKey])(using DbTx): Unit = apiKeyModel.delete(id)

  def invalidateAllForUser(userId: Id[User])(using DbTx): Unit =
    apiKeyModel.deleteAllForUser(userId)
```

All methods take `(using DbTx)` — they're always called from within a transaction managed by the `DB` class.

## Wiring auth into endpoints

Tapir's security model has two phases: first validate the security input (the bearer token), then run the business logic with the authenticated identity. The `handleSecurity` method connects `Auth[ApiKey]` to the endpoint:

```scala
class UserApi(auth: Auth[ApiKey], userService: UserService, db: DB, metrics: Metrics)
    extends ServerEndpoints:

  private def authedEndpoint[I, O](e: Endpoint[Id[ApiKey], I, Fail, O, Any]) =
    e.handleSecurity(authData => auth(authData))
```

`handleSecurity` takes a function `Id[ApiKey] => Either[Fail, Id[User]]` — which is exactly what `auth.apply` provides. On success, the resulting `Id[User]` is passed to the business logic handler. On failure, the `Fail.Unauthorized` is mapped to a 401 response by the error output.

This then lets you write secured endpoints cleanly:

```scala
private val getUserServerEndpoint = authedEndpoint(getUserEndpoint).handle { id => (_: Unit) =>
  val userResult = db.transactEither(userService.findById(id))
  userResult.map(user => GetUser_OUT(user.login, user.emailLowerCase, user.createdOn))
}
```

The `id` parameter is the authenticated `Id[User]` — already validated. The business logic just uses it.

For secured endpoints with a request body, the handler is curried — `id` is the authenticated user, `data` is the request body:

```scala
private val changePasswordServerEndpoint =
  authedEndpoint(changePasswordEndpoint).handle { id => data =>
    db.transactEither(
      userService.changePassword(id, data.currentPassword, data.newPassword)
    ).map(apiKey => ChangePassword_OUT(apiKey.id.toString))
  }
```
