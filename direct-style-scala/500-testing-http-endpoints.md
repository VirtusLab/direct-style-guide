# Testing HTTP Endpoints

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-stub4-server"` — stub backend
  that executes endpoint logic without network I/O
- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-client4"` — interprets endpoint
  descriptions as sttp HTTP requests

---

## Setting up the stub backend

The stub backend is created from the list of server endpoints:

```scala
import sttp.tapir.server.stub4.TapirSyncStubInterpreter

val serverStub: SyncBackend =
  TapirSyncStubInterpreter()
    .whenServerEndpointsRunLogic(allEndpoints)
    .backend()
```

## Converting endpoints to requests

> **Important:** Prefer typed requests over raw `basicRequest`.
> `SttpClientInterpreter` generates request builders from endpoint descriptions,
> giving you compile-time type safety for inputs and outputs. Use `basicRequest`
> only when the endpoint format makes typed builders impractical (e.g. raw
> XML/SOAP bodies).

`SttpClientInterpreter` converts endpoint descriptions into request builders.
For **public endpoints** (no security input):

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

For **secured endpoints**, provide the security input (the bearer token):

```scala
def getUser(apiKey: String): Response[Either[Fail, GetUser_OUT]] =
  SttpClientInterpreter()
    .toSecureRequestThrowDecodeFailures(UserApi.getUserEndpoint, basePath)
    .apply(apiKey.asId)   // security input: Id[ApiKey]
    .apply(())            // regular input: Unit (GET with no body)
    .send(backend)
```

There's also `toRequestThrowErrors` (note: *Errors*, not *DecodeFailures*) which
throws on both decode failures and application errors. This is useful for test
setup methods where the call must succeed:

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

## Writing endpoint tests

```scala
class UserApiTest extends BaseTest with TestDependencies with EitherValues:

  "/user/register" should "register" in {
    val (login, email, password) = requests.randomLoginEmailPassword()

    val response = requests.registerUser(login, email, password)
    val apiKey = response.body.value.apiKey

    requests.getUser(apiKey).body.value.email shouldBe email
  }
```

Error responses are tested the same way — the client interpreter decodes errors
back into the application's error type:

```scala
  "/user/register" should "not register if data is invalid" in {
    val (_, email, password) = requests.randomLoginEmailPassword()

    val response = requests.registerUser("x", email, password) // too short

    response.code shouldBe StatusCode.BadRequest
    response.body should matchPattern { case Left(_: Fail) => }
  }
```

