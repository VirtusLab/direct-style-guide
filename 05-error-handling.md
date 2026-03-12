# Error Handling

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `either` blocks, `.ok()` short-circuiting,
  `.catching` for exception conversion

---

## Errors as values

In direct-style Scala 3, application errors are represented as `Either` values
rather than exceptions. Exceptions are reserved for bugs and unexpected failures
— things that should crash the scope or propagate to a supervisor.

This separation is important: exceptions are automatically managed by Ox's
structured concurrency (a failing fork terminates the scope), while application
errors are explicit in return types and must be handled by the caller.

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
output maps it to HTTP status codes (see the
[Authentication](08-authentication.md) chapter).

## The `either` block

Composing multiple `Either`-returning operations with `for`/`yield` or pattern
matching gets verbose. Ox provides `either` blocks that let you write sequential
code that short-circuits on the first error:

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

`.ok()` unwraps a `Right` value and returns it. If the `Either` is a `Left`, the
block short-circuits immediately, returning that `Left` as the result of the
entire `either` block. The last expression in the block (here,
`apiKeyService.create(...)`) is the success value, automatically wrapped in
`Right`.

Without `either`, the same logic would require nested `flatMap` or
`for`/`yield`:

```scala
def login(...)(using DbTx): Either[Fail, ApiKey] =
  for
    user <- userOrNotFound(userModel.findByLoginOrEmail(loginOrEmailClean.toLowerCased))
    _ <- verifyPassword(user, password, validationErrorMsg = IncorrectLoginOrPassword)
  yield apiKeyService.create(user.id, apiKeyValid.getOrElse(config.defaultApiKeyValid))
```

The `either` version reads top-to-bottom like imperative code. This is
especially valuable when there are many steps, or when intermediate results are
used across multiple subsequent operations (which would require nested `flatMap`
rather than simple `for`/`yield`).

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

Each `.ok()` is a potential exit point. If `validateUserData` returns
`Left(Fail.IncorrectInput("Login is too short!"))`, neither
`checkUserDoesNotExist` nor `doRegister` runs. The error propagates directly to
the caller.

Note the mix of styles: `checkUserDoesNotExist` uses `for`/`yield` internally
(it's simple enough), while the outer method uses `either` to compose the steps.
Use whichever reads better for the given complexity.

## Transactions and `Either`

The `DB` class provides two transaction methods, distinguishing between
operations that can fail with application errors and those that cannot:

```scala
class DB(dataSource: DataSource & Closeable):
  /** Runs `f` in a transaction. Committed if the result is a Right, rolled back otherwise. */
  def transactEither[E, T](f: DbTx ?=> Either[E, T]): Either[E, T] =
    try transact(transactor)(Right(f.fold(e => throw LeftException(e), identity)))
    catch case e: LeftException[E] @unchecked => Left(e.left)

  /** Runs `f` in a transaction. The result cannot be an Either. Committed if no exception. */
  def transact[T](f: DbTx ?=> T)(using NotGiven[T <:< Either[?, ?]]): T =
    transact(transactor)(f)
```

`transactEither` ensures that a `Left` result triggers a rollback. It does this
by converting the `Left` to an exception internally (to integrate with the JDBC
transaction mechanism), then converting it back. This is transparent to the
caller.

The `NotGiven[T <:< Either[?, ?]]` constraint on `transact` is a compile-time
guard: it prevents accidentally using `transact` when the body returns an
`Either`, which would commit the transaction even on a `Left`. You must use
`transactEither` instead.

Endpoint handlers use `transactEither` to wrap service calls (see the
[Authentication](08-authentication.md) chapter for how these handlers are
wired):

```scala
private val registerUserServerEndpoint = registerUserEndpoint.handle { data =>
  db.transactEither(
    userService.registerNewUser(data.login, data.email, data.password)
  ).map(apiKey => Register_OUT(apiKey.id.toString))
}
```

The transaction boundary is at the endpoint handler level. The service method
runs entirely within the transaction, including all validation and database
operations. If any step returns a `Left`, the entire transaction rolls back.

## Converting exceptions to `Either`

When integrating with Java libraries that throw exceptions for expected
failures, use `.catching` to convert them to `Either`:

```scala
import ox.catching

val result: Either[IllegalArgumentException, Int] =
  (if userInput then 10 else throw new IllegalArgumentException("boom"))
    .catching[IllegalArgumentException]
```

The reverse — converting an `Either[Exception, T]` to a thrown exception — is
available via `.orThrow`:

```scala
val value: Int = result.orThrow
```

## Nesting rules

`either` blocks cannot be nested in the same scope. This is a deliberate design
constraint — nested blocks would create ambiguity about which block an `.ok()`
call short-circuits to:

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

This maintains clarity about control flow — each `.ok()` exits exactly the
`either` block it's lexically inside.
