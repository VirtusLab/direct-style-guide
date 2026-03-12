# Authentication

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous (direct-style) backend
- `"com.softwaremill.ox" %% "core"` — `sleep` for timing-attack mitigation

---

## Secured endpoints

Two building blocks are defined for all endpoints in the application.
`baseEndpoint` specifies the error output format (see [Error Output
Customisation](06-error-output-customisation.md)):

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint
    .errorOut(failOutput)

def secureEndpoint[T]: Endpoint[Id[T], Unit, Fail, Unit, Any] =
  baseEndpoint.securityIn(auth.bearer[String]().map(_.asId[T])(_.toString))
```

`Id[T]` is an opaque type over `String`, phantom-typed to prevent mixing up IDs
of different entities at compile time:

```scala
opaque type Id[T] = String

extension (s: String)
  def asId[T]: Id[T] = s
```

`secureEndpoint[T]` reads a bearer token from the `Authorization` header and
parses it as an `Id[T]`.

The type parameter `T` is the token type. For API key–protected endpoints, you
use `secureEndpoint[ApiKey]`, which produces `Endpoint[Id[ApiKey], Unit, Fail,
Unit, Any]` — the security input is `Id[ApiKey]`.

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

The request/response body format (here JSON via `jsonBody`) is independent of
the authentication mechanism.

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

`deleteWhenValid` controls whether the token is single-use. API keys return
`false` (the same key works for many requests). Password-reset codes return
`true` (consumed on use).

A concrete implementation connects the trait to a data access layer:

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

`Auth[T]` is the authenticator. Given a token ID, it returns the authenticated
user's ID or an error:

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
1. **Token not found** — sleep a random 0–100ms before responding. This prevents
   timing attacks where an attacker measures response times to distinguish
   "token doesn't exist" from "token exists but is invalid." The `sleep` call is
   from Ox and blocks the virtual thread without consuming an OS thread.
2. **Token expired** — delete the stale token and return unauthorized.
3. **Token valid** — if it's a one-time token (`deleteWhenValid`), consume it.
   Return the user ID.

The `apply` method makes `Auth[T]` callable as a function, which is how Tapir's
security logic expects it.

## Wiring auth into endpoints

Tapir's security model has two phases: first validate the security input (the
bearer token), then run the business logic with the authenticated identity. The
`handleSecurity` method connects `Auth[ApiKey]` to the endpoint:

```scala
class UserApi(auth: Auth[ApiKey], userService: UserService, db: DB, metrics: Metrics)
    extends ServerEndpoints:

  private def authedEndpoint[I, O](e: Endpoint[Id[ApiKey], I, Fail, O, Any]) =
    e.handleSecurity(authData => auth(authData))
```

`handleSecurity` takes a function `Id[ApiKey] => Either[Fail, Id[User]]` — which
is exactly what `auth.apply` provides. On success, the resulting `Id[User]` is
passed to the business logic handler. On failure, the `Fail.Unauthorized` is
mapped to a 401 response by the error output.

This then lets you write secured endpoints cleanly:

```scala
private val getUserServerEndpoint = authedEndpoint(getUserEndpoint).handle { id => (_: Unit) =>
  val userResult = db.transactEither(userService.findById(id))
  userResult.map(user => GetUser_OUT(user.login, user.emailLowerCase, user.createdOn))
}
```

The `id` parameter is the authenticated `Id[User]` — already validated. The
business logic just uses it.

For secured endpoints with a request body, the handler is curried — `id` is the
authenticated user, `data` is the request body:

```scala
private val changePasswordServerEndpoint =
  authedEndpoint(changePasswordEndpoint).handle { id => data =>
    db.transactEither(
      userService.changePassword(id, data.currentPassword, data.newPassword)
    ).map(apiKey => ChangePassword_OUT(apiKey.id.toString))
  }
```
