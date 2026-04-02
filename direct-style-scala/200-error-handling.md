# Error Handling

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `either` blocks, `.ok()` short-circuiting,
  `.catching` for exception conversion

---

## Recoverable vs unrecoverable errors

The decision rule is simple: can the caller do something meaningful with this
failure?

- **Recoverable** — the caller can handle it: show a message, try an
  alternative, return a different response. Model these as `Either[E, T]`.
  Examples: validation errors, not-found conditions, conflicts, unauthorized
  access.

- **Unrecoverable** — the caller cannot fix it; the current fiber, request, or
  processing unit should be terminated. Throw an exception. Ox's supervised
  scopes, `par`, and `race` handle propagation and cleanup automatically.
  Examples: bugs, database connection failures, corrupted data,
  out-of-memory.

```scala
// Wrong — "not found" is recoverable, the caller can show a 404:
def findUser(id: Id[User])(using DbTx): User =
  userModel.findById(id).getOrElse(throw new NoSuchElementException("user not found"))

// Right — caller sees the failure in the type and decides what to do:
def findUser(id: Id[User])(using DbTx): Either[Fail, User] =
  userModel.findById(id).toRight(Fail.NotFound("user"))
```

> **Important:** If a failure is recoverable, its return type MUST be `Either`.
> Throwing exceptions for recoverable failures hides error paths from callers
> and bypasses the type system.

## The Fail ADT

The application defines a `Fail` ADT as the application-wide error type:

```scala
abstract class Fail

object Fail:
  case class NotFound(what: String) extends Fail
  case class Conflict(msg: String) extends Fail
  case class IncorrectInput(msg: String) extends Fail
  case class Unauthorized(msg: String) extends Fail
  case object Forbidden extends Fail
```

This is not a sealed hierarchy — modules can extend it with domain-specific
failures. The `Fail` type flows through the entire application: service methods
return `Either[Fail, T]`, endpoint handlers propagate it, and Tapir's error
output maps it to HTTP status codes (see [Error Output
Customisation](210-error-output-customisation.md)).

## The `either` block

Ox provides `either` blocks for composing multiple `Either`-returning operations
with short-circuiting:

```scala
import ox.either
import ox.either.ok

def login(loginOrEmail: String, password: String, apiKeyValid: Option[Duration])
    (using DbTx): Either[Fail, ApiKey] = either:
  val loginOrEmailClean = loginOrEmail.trim()
  val user = userOrNotFound(userModel.findByLoginOrEmail(loginOrEmailClean.toLowerCased)).ok()
  verifyPassword(user, password, validationErrorMsg = IncorrectLoginOrPassword).ok()
  apiKeyService.create(user.id, apiKeyValid.getOrElse(config.defaultApiKeyValid))
```

`.ok()` unwraps a `Right` or short-circuits the `either` block with the `Left`.
The last expression is the success value, wrapped in `Right`.

## Composing validations

A more complex example — user registration performs input validation, uniqueness
checks, and then the actual registration:

```scala
def registerNewUser(login: String, email: String, password: String)
    (using DbTx): Either[Fail, ApiKey] =
  val loginClean = login.trim()
  val emailClean = email.trim()

  def checkUserDoesNotExist(): Either[Fail, Unit] = for
    _ <- failIfDefined(userModel.findByLogin(loginClean.toLowerCased), LoginAlreadyUsed)
    _ <- failIfDefined(userModel.findByEmail(emailClean.toLowerCased), EmailAlreadyUsed)
  yield ()

  either:
    validateUserData(Some(loginClean), Some(emailClean), Some(password)).ok()
    checkUserDoesNotExist().ok()
    doRegister()
```


## Transactions and `Either`

The `DB` class provides two transaction methods, distinguishing between
operations that can fail with application errors and those that cannot:

```scala
class DB(dataSource: DataSource & Closeable):
  /** Runs `f` in a transaction. Committed if the result is a Right, rolled back otherwise. */
  def transactEither[E, T](f: DbTx ?=> Either[E, T]): Either[E, T] =
    try com.augustnagro.magnum.transact(transactor)(Right(f.fold(e => throw LeftException(e), identity)))
    catch case e: LeftException[E] @unchecked => Left(e.left)

  /** Runs `f` in a transaction. The result cannot be an Either. Committed if no exception. */
  def transact[T](f: DbTx ?=> T)(using NotGiven[T <:< Either[?, ?]]): T =
    com.augustnagro.magnum.transact(transactor)(f)
```

`transactEither` ensures that a `Left` result triggers a rollback.

> **Important:** The `NotGiven` constraint makes it a compile error to use
> `transact` with an `Either`-returning body — use `transactEither` instead.
> Using `transact` with `Either` would silently skip rollback on `Left`.

Endpoint handlers use `transactEither` to wrap service calls (see the
[Authentication](300-authentication.md) chapter for how these handlers are
wired):

```scala
private val registerUserServerEndpoint = registerUserEndpoint.handle { data =>
  db.transactEither(
    userService.registerNewUser(data.login, data.email, data.password)
  ).map(apiKey => Register_OUT(apiKey.id.toString))
}
```

## Converting exceptions to `Either`

Java libraries throw exceptions for failures that are expected in your domain
(e.g. `IllegalArgumentException` for invalid input). Convert these to `Either`
at the boundary using `.catching`:

```scala
import ox.either.catching

val result: Either[IllegalArgumentException, Int] =
  (if userInput then 10 else throw new IllegalArgumentException("boom"))
    .catching[IllegalArgumentException]
```

The reverse — converting an `Either[Exception, T]` to a thrown exception — is
available via `.orThrow`:

```scala
val value: Int = result.orThrow
```

## Wrapping exception-throwing code

Java and third-party libraries often throw exceptions for failures that are
expected in your domain. Convert these to values at the boundary using
`.catching` — never let `try`/`catch` become your error-handling strategy for
expected failures.

```scala
import ox.either.catching

// Wrong — bare try/catch for an expected failure:
def download(key: String): Option[Path] =
  try
    val path = doDownload(key)
    Some(path)
  catch case _: NotFoundException => None

// Right — use .catching to convert the exception to Either, then to Option:
def download(key: String): Option[Path] =
  doDownload(key).catching[NotFoundException].toOption

// Or when the failure is domain-meaningful, map the Left:
def download(key: String): Either[StorageError, Path] =
  doDownload(key)
    .catching[NotFoundException]
    .left.map(e => StorageError.NotFound(key))
```

> **Important:** Never use bare `try`/`catch` for expected failures — this
> hides error paths and bypasses the type system. The `try`/`catch` pattern is
> only appropriate for truly unexpected exceptions (defects) at the outermost
> boundary of a processing loop (e.g. a `catch NonFatal` in a `forever` loop
> to prevent the fiber from dying). Always catch the most specific expected
> exception type — avoid `.catching[Exception]` as it swallows defects.

## Nesting rules

> **Important:** `either` blocks cannot be nested in the same scope — nested
> blocks would create ambiguity about which block an `.ok()` call
> short-circuits to. Extract the inner block into a separate method instead.

```scala
// This won't compile:
either:
  val x = either:
    someEither.ok()  // which `either` does this exit?
  x.ok()
```

The fix is to extract the inner block into a separate method:

```scala
def inner(): Either[E, T] = either:
  someEither.ok()

either:
  inner().ok()
```

