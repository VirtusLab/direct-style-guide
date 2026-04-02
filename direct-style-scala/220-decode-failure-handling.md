# Decode Failure Handling

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend

---

## When decoding fails

Inputs are decoded in order: method, path, query, headers, body. When any input
fails to decode, the `DecodeFailureHandler` decides what to do: respond with an
error, or skip this endpoint and try the next one.

## Default behavior

The `DefaultDecodeFailureHandler` applies these rules:

| Failing input | Result |
|---|---|
| Query parameter, header, cookie, body | 400 Bad Request |
| Content-Type mismatch | 415 Unsupported Media Type |
| Body exceeds size limit | 413 Payload Too Large |
| Auth input (created via `auth.bearer`, etc.) | 401 Unauthorized + `WWW-Authenticate` header |
| Path capture — parse error or validation error | 400 Bad Request |
| Path capture — missing/mismatch | Try next endpoint |
| Path segment mismatch | Try next endpoint |

## The handler structure

`DefaultDecodeFailureHandler` is a case class with three customizable
components:

```scala
case class DefaultDecodeFailureHandler[F[_]](
    respond: DecodeFailureContext => Option[(StatusCode, List[Header])],
    failureMessage: DecodeFailureContext => String,
    response: (StatusCode, List[Header], String) => ValuedEndpointOutput[?]
)
```

- **`respond`** — decides whether to return an error response (and with which
  status code and headers) or `None` to try the next endpoint.
- **`failureMessage`** — generates the error message string from the failure
  context (which input failed, why).
- **`response`** — assembles the final output from the status code, headers, and
  message. By default, this returns plain text (see [Error Output
  Customisation](210-error-output-customisation.md) for returning JSON instead).

## Customizing the handler

Replace any of the three components via `copy`:

```scala
val customHandler = DefaultDecodeFailureHandler[Identity].copy(
  respond = ctx => myRespondLogic(ctx),
  failureMessage = ctx => myMessageLogic(ctx)
)

val serverOptions = NettySyncServerOptions.customiseInterceptors
  .decodeFailureHandler(customHandler)
  .options
```

### Custom respond logic

Override `respond` to change which failures trigger error responses. For
example, to treat all path capture failures as "try next endpoint" (including
validation errors):

```scala
import sttp.tapir.server.interceptor.decodefailure.DefaultDecodeFailureHandler

val handler = DefaultDecodeFailureHandler[Identity].copy(
  respond = ctx =>
    ctx.failingInput match
      case _: EndpointInput.PathCapture[?] => None  // always try next endpoint
      case _ => DefaultDecodeFailureHandler.respond(ctx)  // default for everything else
)
```

### Custom failure messages

Override `failureMessage` to control the error text. The default messages follow
the pattern `"Invalid value for: query parameter name (details)"`:

```scala
val handler = DefaultDecodeFailureHandler[Identity].copy(
  failureMessage = ctx =>
    val source = DefaultDecodeFailureHandler.FailureMessages.failureSourceMessage(ctx.failingInput)
    val detail = DefaultDecodeFailureHandler.FailureMessages.failureDetailMessage(ctx.failure)
    s"$source${detail.map(d => s": $d").getOrElse("")}"
)
```

The `FailureMessages` object provides building blocks: `failureSourceMessage`
identifies which input failed (e.g., "query parameter name"),
`failureDetailMessage` describes why (e.g., "missing", "value mismatch", or
validation details).

## Per-input routing control

For individual inputs, `onDecodeFailureNextEndpoint` overrides the handler to
always try the next endpoint when that input fails to decode:

```scala
import sttp.tapir.server.interceptor.decodefailure.DefaultDecodeFailureHandler.OnDecodeFailure.*

endpoint.in("customer" / path[UserId]("user_id").onDecodeFailureNextEndpoint)
endpoint.in("customer" / "some_special_case")
```

Without `onDecodeFailureNextEndpoint`, a request to `/customer/some_special_case`
would fail to decode `"some_special_case"` as a `UserId` and return 400. With
the attribute, the first endpoint is skipped and the second one matches.

## Hiding authenticated endpoints

`DefaultDecodeFailureHandler.hideEndpointsWithAuth` converts all error responses
to 404 for endpoints that contain auth inputs:

```scala
val serverOptions = NettySyncServerOptions.customiseInterceptors
  .decodeFailureHandler(DefaultDecodeFailureHandler.hideEndpointsWithAuth[Identity])
  .options
```

This prevents discovering authenticated endpoints by probing paths — the same
404 response is returned whether the endpoint exists or not.

