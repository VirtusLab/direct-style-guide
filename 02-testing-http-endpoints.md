# Testing HTTP Endpoints

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-stub4-server"` — stub backend that executes endpoint logic without network I/O
- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-client4"` — interprets endpoint descriptions as sttp HTTP requests
- `"org.scalatest" %% "scalatest"` — test framework
- `"com.opentable.components" % "otj-pg-embedded"` — embedded PostgreSQL for integration tests

---

## The testing approach

This chapter builds on the endpoint definitions, `Fail` ADT, and authentication patterns from the [Authentication](01-authentication.md) chapter.

The goal is to test the full endpoint stack — JSON serialization, error mapping, security logic, and business logic — without starting an HTTP server or making network calls.

This works because Tapir separates endpoint *descriptions* from endpoint *implementations*:

- **Endpoint descriptions** are data structures that define the shape of an HTTP endpoint (method, path, inputs, outputs, errors). They are server-agnostic.
- **Server endpoints** bind descriptions to implementation logic, producing values that can be interpreted by a server — or by a stub.

The testing approach uses two interpreters:

1. **`TapirSyncStubInterpreter`** — takes a list of server endpoints and creates an sttp `SyncBackend` that routes requests to matching endpoint logic. No TCP, no HTTP server, but the full Tapir pipeline runs (input decoding, security, error mapping, output encoding).

2. **`SttpClientInterpreter`** — takes an endpoint *description* and produces a function that builds an sttp `Request` from typed inputs. The request is fully formed (correct path, headers, JSON body) and can be sent through the stub backend.

Together, you write tests that look like real HTTP calls — with the same types, the same error handling, the same authentication — but execute in-process.

## Setting up the stub backend

The stub backend is created from the full list of server endpoints, including documentation and static file endpoints:

```scala
import sttp.client4.SyncBackend
import sttp.client4.httpclient.HttpClientSyncBackend
import sttp.tapir.server.stub4.TapirSyncStubInterpreter

trait TestDependencies extends BeforeAndAfterAll with TestEmbeddedPostgres:
  self: Suite & BaseTest =>

  var dependencies: Dependencies = uninitialized

  override protected def beforeAll(): Unit =
    super.beforeAll()
    dependencies = Dependencies.create(
      TestConfig,
      OpenTelemetry.noop(),
      HttpClientSyncBackend.stub,
      currentDb,
      testClock
    )

  private lazy val serverStub: SyncBackend =
    TapirSyncStubInterpreter()
      .whenServerEndpointsRunLogic(dependencies.httpApi.allEndpoints)
      .backend()

  lazy val requests = new Requests(serverStub)
```

Key points:

- `HttpClientSyncBackend.stub` is passed as the sttp backend dependency. This means any *outgoing* HTTP calls (e.g., to external services) made by the application during tests are also stubbed — they won't hit the network.
- `OpenTelemetry.noop()` disables all observability instrumentation in tests, avoiding noise.
- `TapirSyncStubInterpreter().whenServerEndpointsRunLogic(endpoints).backend()` creates a backend where sending a request matches it against the endpoint list, runs the matched endpoint's full logic (security + business logic), and returns the response.
- The `Dependencies.create` method is the same one used in production, just with test implementations injected. This means tests exercise the real wiring.

## Converting endpoints to requests

`SttpClientInterpreter` converts endpoint descriptions into request builders. For **public endpoints** (no security input):

```scala
class Requests(backend: SyncBackend):
  private val basePath = Some(uri"http://localhost:8080/api/v1")

  def registerUser(login: String, email: String, password: String)
      : Response[Either[Fail, Register_OUT]] =
    SttpClientInterpreter()
      .toRequestThrowDecodeFailures(UserApi.registerUserEndpoint, basePath)
      .apply(Register_IN(login, email, password))
      .send(backend)
```

`toRequestThrowDecodeFailures` returns a function `Register_IN => Request[Either[Fail, Register_OUT]]`. The method name means: if the response can't be decoded at all (e.g., completely malformed), throw an exception rather than returning it as a value. But expected errors (like validation failures) are returned as `Left(Fail)`.

The `basePath` simulates the `/api/v1` context path that the server prepends to all endpoints.

For **secured endpoints**, there's an extra step — providing the security input (the bearer token):

```scala
def getUser(apiKey: String): Response[Either[Fail, GetUser_OUT]] =
  SttpClientInterpreter()
    .toSecureRequestThrowDecodeFailures(UserApi.getUserEndpoint, basePath)
    .apply(apiKey.asId)   // security input: Id[ApiKey]
    .apply(())            // regular input: Unit (GET with no body)
    .send(backend)
```

`toSecureRequestThrowDecodeFailures` returns a curried function: first apply the security input (`Id[ApiKey]`), then the regular input. The security input is automatically placed in the `Authorization: Bearer` header — the same encoding specified in the endpoint description.

For endpoints with both security and body inputs:

```scala
def changePassword(apiKey: String, password: String, newPassword: String)
    : Response[Either[Fail, ChangePassword_OUT]] =
  SttpClientInterpreter()
    .toSecureRequestThrowDecodeFailures(UserApi.changePasswordEndpoint, basePath)
    .apply(apiKey.asId)
    .apply(ChangePassword_IN(password, newPassword))
    .send(backend)
```

A helper creates a registered user in one call, useful for test setup:

```scala
def newRegisteredUsed(): RegisteredUser =
  val (login, email, password) = randomLoginEmailPassword()
  val apiKey = SttpClientInterpreter()
    .toRequestThrowErrors(UserApi.registerUserEndpoint, basePath)
    .apply(Register_IN(login, email, password))
    .send(backend)
    .body
    .apiKey
  RegisteredUser(login, email, password, apiKey)
```

`toRequestThrowErrors` (note: *Errors*, not *DecodeFailures*) throws on both decode failures and application errors. This is appropriate for setup methods where registration must succeed.

## Test infrastructure

### Embedded PostgreSQL

Tests run against a real PostgreSQL instance, started once per test suite:

```scala
trait TestEmbeddedPostgres extends BeforeAndAfterAll with BeforeAndAfterEach:
  self: Suite =>

  var currentDb: DB = uninitialized
  private var postgres: EmbeddedPostgres = uninitialized

  override protected def beforeAll(): Unit =
    super.beforeAll()
    postgres = EmbeddedPostgres.builder().start()
    // ... create DB, run migrations

  override protected def afterEach(): Unit =
    // Flyway.clean() between tests for isolation
    super.afterEach()

  override protected def afterAll(): Unit =
    postgres.close()
    super.afterAll()
```

Using a real database (rather than mocks) ensures that SQL queries, constraints, and transactions behave exactly as in production.

### Test clock

Time-dependent behavior (like API key expiration) is tested using a controllable clock:

```scala
trait BaseTest extends AnyFlatSpec with Matchers:
  InheritableMDC.init
  val testClock = new TestClock()
```

The `TestClock` can be advanced programmatically (e.g., `testClock.forward(2.hours)`), allowing tests to verify expiration logic without waiting.

### Noop OpenTelemetry

`OpenTelemetry.noop()` is passed during test setup. This disables all tracing and metrics collection, ensuring tests don't produce observability side effects or require a collector.

## Writing endpoint tests

Tests extend `BaseTest with TestDependencies` to get access to the stub backend, the embedded database, and the test clock.

### Basic registration and retrieval

```scala
class UserApiTest extends BaseTest with Eventually with TestDependencies with EitherValues:

  "/user/register" should "register" in {
    val (login, email, password) = requests.randomLoginEmailPassword()

    val response1 = requests.registerUser(login, email, password)
    val apiKey = response1.body.value.apiKey

    val response4 = requests.getUser(apiKey)
    response4.body.value.email shouldBe email
  }
```

The test registers a user, extracts the API key from the response, then uses it to fetch the user profile. `body.value` (from ScalaTest's `EitherValues`) unwraps the `Right` or fails the test with a clear message.

### Error response testing

```scala
"/user/register" should "not register if data is invalid" in {
  val (_, email, password) = requests.randomLoginEmailPassword()

  val response1 = requests.registerUser("x", email, password) // too short

  response1.code shouldBe StatusCode.BadRequest
  response1.body should matchPattern { case Left(_: Fail) => }
}
```

The full error pipeline is tested: the `Fail.IncorrectInput` returned by the service → mapped to `StatusCode.BadRequest` → serialized as JSON → decoded back to `Fail.IncorrectInput` by the client interpreter.

### Authorization testing

```scala
"/user/info" should "respond with 401 if the token is invalid" in {
  requests.getUser("invalid").code shouldBe StatusCode.Unauthorized
}

"/user/logout" should "logout the user" in {
  val RegisteredUser(_, _, _, apiKey) = requests.newRegisteredUsed()

  val response = requests.logoutUser(apiKey)
  response.code shouldBe StatusCode.Ok

  val responseAfterLogout = requests.getUser(apiKey)
  responseAfterLogout.code shouldBe StatusCode.Unauthorized
}
```

The logout test verifies that the API key is actually invalidated — a subsequent request with the same key returns 401.

### Time-dependent behavior

```scala
"/user/login" should "login the user for the given number of hours" in {
  val RegisteredUser(login, _, password, _) = requests.newRegisteredUsed()

  val apiKey = requests.loginUser(login, password, Some(3)).body.value.apiKey

  requests.getUser(apiKey).body should matchPattern { case Right(_) => }

  testClock.forward(2.hours)
  requests.getUser(apiKey).body should matchPattern { case Right(_) => }

  testClock.forward(2.hours)
  requests.getUser(apiKey).code shouldBe StatusCode.Unauthorized
}
```

The test creates an API key valid for 3 hours, advances time by 2 hours (still valid), then by another 2 hours (now expired). The `testClock` is the same clock instance used by `Auth[T]` for expiration checks.

### Password change invalidates all sessions

```scala
"/user/changepassword" should "create new session and invalidate all existing sessions" in {
  val RegisteredUser(login, _, password, apiKey1) = requests.newRegisteredUsed()
  val newPassword = password + password

  val apiKey2 = requests.loginUser(login, password, Some(3)).body.value.apiKey

  val response = requests.changePassword(apiKey1, password, newPassword)

  val newApiKey = response.body.value.apiKey
  requests.getUser(newApiKey).code shouldBe StatusCode.Ok
  requests.getUser(apiKey1).code shouldBe StatusCode.Unauthorized
  requests.getUser(apiKey2).code shouldBe StatusCode.Unauthorized
}
```

This tests a security-critical behavior: changing the password invalidates *all* existing API keys (not just the one used for the request) and returns a fresh one. The test creates two sessions, changes the password via the first, then verifies both old keys are invalid while the new one works.
