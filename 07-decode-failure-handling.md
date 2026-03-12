# Decode Failure Handling

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend

---

## When decoding fails

Tapir decodes incoming requests by matching them against endpoint descriptions.
Inputs are decoded in order: method, path, query, headers, body. When any input
fails to decode, the `DecodeFailureHandler` decides what to do: respond with an
error, or skip this endpoint and try the next one.

This decision is critical for routing. If two endpoints share a path prefix but
differ in a path parameter type, the handler must distinguish "this endpoint
doesn't match, try the next" from "this endpoint matches but the input is
invalid."

## Default behavior

The `DefaultDecodeFailureHandler` applies these rules:

| Failing input | Result |
|---|---|
| Query parameter, header, cookie, body | 400 Bad Request |
| Content-Type mismatch | 415 Unsupported Media Type |
| Body exceeds size limit | 413 Payload Too Large |
| Auth input (created via `auth.bearer`, etc.) | 401 Unauthorized + `WWW-Authenticate` header |
| Path capture — validation error | 400 Bad Request |
| Path capture — other decode error | Try next endpoint |
| Path segment mismatch | Try next endpoint |

The key distinction: a path capture that fails validation (e.g., an enum value
outside the allowed set) returns 400, while a path capture that simply can't be
parsed (e.g., `"abc"` for an `Int`) silently passes to the next endpoint. This
makes path-based routing work correctly when multiple endpoints share a prefix.

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
  Customisation](06-error-output-customisation.md) for returning JSON instead).

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

With `Identity` as the effect type (direct-style), the handler is a plain
function — no monadic wrapping needed.

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

This is useful when endpoints have overlapping path patterns and the path
parameter type determines which endpoint should handle the request.

## Hiding authenticated endpoints

`DefaultDecodeFailureHandler.hideEndpointsWithAuth` converts all error responses
(both 400 and 401) to 404 for endpoints that contain auth inputs:

```scala
val serverOptions = NettySyncServerOptions.customiseInterceptors
  .decodeFailureHandler(DefaultDecodeFailureHandler.hideEndpointsWithAuth[Identity])
  .options
```

This prevents an attacker from discovering authenticated endpoints by probing
paths — they get the same 404 response whether the endpoint exists or not. Note
that timing attacks may still reveal endpoint existence.

## Combining custom rules with defaults

To add a custom rule while preserving default behavior, match your case first
and delegate to `DefaultDecodeFailureHandler.respond` as the fallback:

```scala
val handler = DefaultDecodeFailureHandler[Identity].copy(
  respond = ctx =>
    ctx.failingInput match
      case _: EndpointInput.PathCapture[?] => None
      case _ => DefaultDecodeFailureHandler.respond(ctx)
)
```

The custom case is checked first. Everything else falls through to the default
logic (400 for query/header/body, 401 for auth, etc.). This works because
`DefaultDecodeFailureHandler.respond` is a standalone function — it doesn't
depend on internal state.

For full control over the response (bypassing the respond → message → response
pipeline entirely), use `DecodeFailureHandler.pure`:

```scala
val defaultHandler = DefaultDecodeFailureHandler[Identity]

val handler = DecodeFailureHandler.pure[Identity] { ctx =>
  ctx.failingInput match
    case _: EndpointInput.PathCapture[?] => None
    case _ => defaultHandler(ctx)  // delegate to the default respond/message/response pipeline
}
```

This gives you a single function that returns the complete output directly,
rather than composing it from separate respond/message/response steps.