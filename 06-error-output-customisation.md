# Error Output Customisation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend
- `"com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-macros"` — JSON
  codec derivation

---

## The problem

By default, Tapir returns errors as plain text. When a request can't be decoded
(wrong content type, missing query parameter, malformed JSON), the server
responds with a plain-text body like `Invalid value for: query parameter name`.
For an API that returns JSON everywhere, this inconsistency is a problem —
clients expect all responses, including errors, to be JSON.

## Defining a JSON error output

All errors are returned as a JSON body with a single `error` field:

```scala
case class Error_OUT(error: String) derives ConfiguredJsonValueCodec, Schema
```

The `derives` clause uses Scala 3's type class derivation to automatically
generate the JSON codec (via jsoniter-scala) and the Tapir schema (for OpenAPI
documentation).

## Mapping Fail to HTTP responses

The `Fail` ADT (see [Error Handling](05-error-handling.md)) needs a
bidirectional mapping to HTTP status codes and error messages. The forward
direction is used by the server; the reverse is used by the client interpreter
in tests:

```scala
val jsonErrorOutOutput: EndpointOutput[Error_OUT] = jsonBody[Error_OUT]

private val failOutput: EndpointOutput[Fail] =
  statusCode
    .and(jsonErrorOutOutput.map(_.error)(Error_OUT.apply))
    .map(responseDataToFail.tupled)(failToResponseData)
```

`.map` on a Tapir output creates a bidirectional transformation. The first
function decodes (response → domain), the second encodes (domain → response).
The decode direction is used by the client interpreter (in tests), and the
encode direction by the server.

The `failToResponseData` function maps each `Fail` variant to a status code and
message:

```scala
private val failToResponseData: Fail => (StatusCode, String) = {
  case Fail.NotFound(what)      => (StatusCode.NotFound, what)
  case Fail.Conflict(msg)       => (StatusCode.Conflict, msg)
  case Fail.IncorrectInput(msg) => (StatusCode.BadRequest, msg)
  case Fail.Forbidden           => (StatusCode.Forbidden, "Forbidden")
  case Fail.Unauthorized(msg)   => (StatusCode.Unauthorized, msg)
  case _                        => (StatusCode.InternalServerError, "Internal server error")
}
```

## Applying to all endpoints

The `failOutput` is used in `baseEndpoint`, from which all endpoints in the
application inherit:

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint.errorOut(failOutput)
```

This means every endpoint automatically returns errors as JSON with the
appropriate status code. Endpoint implementations just return
`Left(Fail.IncorrectInput("..."))` — the mapping to HTTP is handled once,
centrally.

## Customising default error handlers

Tapir has built-in error handling for situations that happen outside endpoint
logic — decode failures, unmatched routes, and unhandled exceptions. By default,
these produce plain-text responses. To make them return JSON too:

```scala
val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .defaultHandlers(
    msg => ValuedEndpointOutput(Http.jsonErrorOutOutput, Error_OUT(msg)),
    notFoundWhenRejected = true
  )
  .options
```

`defaultHandlers` takes a function that wraps any error message string in the
same `Error_OUT` JSON format used by endpoint error outputs.
`notFoundWhenRejected = true` returns a 404 (as JSON) instead of a 405 when no
endpoint matches the request path.

This ensures that *all* error responses from the server — whether from endpoint
logic, input decoding, or routing — are consistently formatted as JSON.
