# Authentication

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous (direct-style) backend
- `"com.softwaremill.ox" %% "core"` — `sleep` for timing-attack mitigation

---

## Secured endpoints

Two building blocks are defined for all endpoints in the application.
`baseEndpoint` specifies the error output format (see [Error Output
Customisation](210-error-output-customisation.md)):

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint
    .errorOut(failOutput)

def secureEndpoint[T]: Endpoint[Id[T], Unit, Fail, Unit, Any] =
  baseEndpoint.securityIn(auth.bearer[String]().map(_.asId[T])(_.toString))
```

`Id[T]` is an opaque type over `String` (see [SQL
Persistence](400-sql-persistence.md)). Secured endpoints use
`secureEndpoint[ApiKey]`:

```scala
val getUserEndpoint = secureEndpoint[ApiKey].get
  .in("user")
  .out(jsonBody[GetUser_OUT])
```

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

## Wiring auth into endpoints

`handleSecurity` connects `Auth[ApiKey]` to the endpoint — it validates the
bearer token before the business logic runs:

```scala
class UserApi(auth: Auth[ApiKey], userService: UserService, db: DB, metrics: Metrics)
    extends ServerEndpoints:

  private def authedEndpoint[I, O](e: Endpoint[Id[ApiKey], I, Fail, O, Any]) =
    e.handleSecurity(authData => auth(authData))
```

Secured endpoints then look like:

```scala
private val getUserServerEndpoint = authedEndpoint(getUserEndpoint).handle { id => (_: Unit) =>
  val userResult = db.transactEither(userService.findById(id))
  userResult.map(user => GetUser_OUT(user.login, user.emailLowerCase, user.createdOn))
}
```

With a request body, the handler is curried — `id` then `data`:

```scala
private val changePasswordServerEndpoint =
  authedEndpoint(changePasswordEndpoint).handle { id => data =>
    db.transactEither(
      userService.changePassword(id, data.currentPassword, data.newPassword)
    ).map(apiKey => ChangePassword_OUT(apiKey.id.toString))
  }
```
