# Testing HTTP Endpoints

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-stub4-server"` — stub backend
  that executes endpoint logic without network I/O
- `"com.softwaremill.sttp.tapir" %% "tapir-sttp-client4"` — interprets endpoint
  descriptions as sttp HTTP requests

---

## The testing approach

The goal is to test the full endpoint stack — input decoding, error mapping,
and business logic — without starting an HTTP server or making network calls.

This works because Tapir separates endpoint *descriptions* from endpoint
*implementations*:

- **Endpoint descriptions** are data structures that define the shape of an HTTP
  endpoint (method, path, inputs, outputs, errors). They are server-agnostic.
- **Server endpoints** bind descriptions to implementation logic, producing
  values that can be interpreted by a server — or by a stub.

The testing approach uses two interpreters:

1. **`TapirSyncStubInterpreter`** — takes a list of server endpoints and creates
   an sttp `SyncBackend` that routes requests to matching endpoint logic. No
   TCP, no HTTP server, but the full Tapir pipeline runs (input decoding,
   security, error mapping, output encoding).

2. **`SttpClientInterpreter`** — takes an endpoint *description* and produces a
   function that builds an sttp `Request` from typed inputs. The request is
   fully formed (correct path, headers, JSON body) and can be sent through the
   stub backend.

Together, you write tests that look like real HTTP calls — with the same types
and the same error handling — but execute in-process.

## Setting up the stub backend

The stub backend is created from the list of server endpoints:

```scala
import sttp.tapir.server.stub4.TapirSyncStubInterpreter

val serverStub: SyncBackend =
  TapirSyncStubInterpreter()
    .whenServerEndpointsRunLogic(allEndpoints)
    .backend()
```

`whenServerEndpointsRunLogic` creates a backend where sending a request matches
it against the endpoint list, runs the matched endpoint's full logic (including
security), and returns the response. The `allEndpoints` list is the same one
used by the production server.

## Converting endpoints to requests

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

`toRequestThrowDecodeFailures` returns a function `Register_IN =>
Request[Either[Fail, Register_OUT]]`. If the response can't be decoded at all
(e.g., completely malformed), it throws an exception. But expected errors (like
validation failures) are returned as `Left(Fail)`.

The `basePath` simulates the context path that the server prepends to all
endpoints.

For **secured endpoints**, there's an extra step — providing the security input
(the bearer token):

```scala
def getUser(apiKey: String): Response[Either[Fail, GetUser_OUT]] =
  SttpClientInterpreter()
    .toSecureRequestThrowDecodeFailures(UserApi.getUserEndpoint, basePath)
    .apply(apiKey.asId)   // security input: Id[ApiKey]
    .apply(())            // regular input: Unit (GET with no body)
    .send(backend)
```

`toSecureRequestThrowDecodeFailures` returns a curried function: first apply the
security input (`Id[ApiKey]`), then the regular input. The security input is
automatically placed in the `Authorization: Bearer` header — the same encoding
specified in the endpoint description.

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

`body.value` (from ScalaTest's `EitherValues`) unwraps the `Right` or fails the
test with a clear message.

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

The full error pipeline is exercised: `Fail.IncorrectInput` returned by the
service → mapped to `StatusCode.BadRequest` → serialized as JSON → decoded back
to `Fail` by the client interpreter.
