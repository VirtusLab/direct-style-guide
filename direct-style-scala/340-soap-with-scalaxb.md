# SOAP with scalaxb

## Dependencies

- `"org.scalaxb" %% "scalaxb"` — XSD/WSDL-to-Scala code generator (sbt plugin)
- `"org.scala-lang.modules" %% "scala-xml"` — XML literal support and parsing
- `"org.scala-lang.modules" %% "scala-parser-combinators"` — required by scalaxb
  at runtime
- `"javax.xml.bind" % "jaxb-api"` — `XMLGregorianCalendar` for `xs:dateTime`
  fields

---

## Code generation from XSD

scalaxb generates case classes and `XMLFormat` typeclass instances from XSD
schemas. Place the schema in `src/main/xsd/`, enable the sbt plugin, and
configure the output package:

```scala
// project/plugins.sbt
addSbtPlugin("org.scalaxb" % "sbt-scalaxb" % "1.12.2")

// build.sbt
lazy val root = project
  .enablePlugins(ScalaxbPlugin)
  .settings(
    Compile / scalaxb / scalaxbPackageName := "myapp.generated",
    Compile / scalaxb / scalaxbGenerateDispatchAs := false
  )
```

`scalaxbGenerateDispatchAs := false` disables the generated async dispatch
client — you'll call SOAP services through Tapir endpoints instead. The plugin
runs at compile time, producing case classes (e.g. `MyRequest(field1, field2,
...)`) and bidirectional `XMLFormat[T]` instances that convert between XML and
Scala types.

The generated package also exports a `defaultScope` — an XML `NamespaceBinding`
that maps prefixes to the target namespace. You'll need it when serialising
back to XML with `scalaxb.toXML`.

## SOAP envelope handling

SOAP 1.1 wraps payloads in
`<soapenv:Envelope><soapenv:Body>...</soapenv:Body></soapenv:Envelope>`. Two
utility methods handle wrapping and unwrapping:

```scala
import scala.xml.{Elem, NodeSeq}

object SoapEnvelope:

  def wrapBody(payload: NodeSeq): Elem =
    <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
      <soapenv:Body>{payload}</soapenv:Body>
    </soapenv:Envelope>

  def extractBody(envelope: Elem): Either[String, Elem] =
    (envelope \ "Body").headOption match
      case None => Left("Missing soapenv:Body")
      case Some(body) =>
        body.child
          .collectFirst { case e: Elem => e }
          .toRight("Empty soapenv:Body")
```

`extractBody` finds the first child element inside `<Body>`, skipping
whitespace text nodes. This is the element that scalaxb's `XMLFormat` will
deserialise. Both methods are used by the Tapir codec below.

## Tapir XML codecs for scalaxb types

Tapir doesn't know about scalaxb's `XMLFormat`. A bridging codec handles the
full decode/encode cycle — parse XML string, extract SOAP body, deserialise
with scalaxb, and the reverse for encoding:

```scala
import sttp.tapir.*
import scalaxb.XMLFormat
import scala.xml.{Elem, XML}
import scala.util.Try

private def soapCodec[T](elementLabel: String)(using fmt: XMLFormat[T])
    : Codec[String, T, CodecFormat.Xml] =
  given Schema[T] = Schema.string
  Codec.xml { xmlStr =>
    Try(XML.loadString(xmlStr)) match
      case scala.util.Failure(e) => DecodeResult.Error(xmlStr, e)
      case scala.util.Success(envelope) =>
        SoapEnvelope.extractBody(envelope) match
          case Left(err) =>
            DecodeResult.Error(xmlStr, new Exception(err))
          case Right(bodyElem) =>
            fmt.reads(bodyElem, Nil) match
              case Left(err) => DecodeResult.Error(xmlStr, new Exception(err))
              case Right(t)  => DecodeResult.Value(t)
  } { t =>
    SoapEnvelope
      .wrapBody(scalaxb.toXML[T](t, elementLabel, defaultScope))
      .toString()
  }
```

`elementLabel` is the XML element name that `scalaxb.toXML` uses as the root
tag — it must match the element name from the XSD. `Schema.string` is a
placeholder; the actual validation happens in the codec itself.

Define one codec instance per SOAP type, plus a raw `Elem` codec for fault
responses:

```scala
private given Codec[String, Elem, CodecFormat.Xml] =
  given Schema[Elem] = Schema.string
  Codec.xml { s =>
    Try(XML.loadString(s)) match
      case scala.util.Failure(e) => DecodeResult.Error(s, e)
      case scala.util.Success(e) => DecodeResult.Value(e)
  }(_.toString())

private given Codec[String, MyRequest, CodecFormat.Xml] =
  soapCodec("MyRequest")
private given Codec[String, MyResponse, CodecFormat.Xml] =
  soapCodec("MyResponse")
```

## Defining SOAP endpoints

Each SOAP operation becomes a separate Tapir endpoint. Operations share the same
path and are distinguished by the `SOAPAction` header:

```scala
import sttp.tapir.{ValidationError as _, *}
import sttp.tapir.server.interceptor.decodefailure.DefaultDecodeFailureHandler
  .OnDecodeFailure.*
import sttp.model.StatusCode as SC
import myapp.generated.*
import myapp.generated.`package`.*

private val soapPath = "soap" / "MyService"

private def soapAction(operation: String) =
  header("SOAPAction", s"\"$operation\"").onDecodeFailureNextEndpoint
```

> **Important:** `onDecodeFailureNextEndpoint` is required here — without it,
> a non-matching `SOAPAction` header returns a 400 instead of trying the next
> endpoint. This lets multiple operations share the same path, with the header
> acting as the discriminator.

Each endpoint maps the request type to a response or a SOAP fault. The error
output uses `statusCode(SC.InternalServerError)` because SOAP conventionally
returns 500 for faults. `SoapFault` is a helper that builds standard
`<soapenv:Fault>` XML with `faultcode`, `faultstring`, and optional `<detail>`:

```scala
case class ValidationError(code: String, message: String)

private def toFaultElem(err: ValidationError): Elem =
  val detail = scalaxb.toXML[MyFault](
    MyFault(err.code, err.message),
    "MyFault",
    defaultScope
  )
  SoapFault.serverFault(err.message, Some(detail))

private val faultErrorOutput =
  statusCode(SC.InternalServerError).and(xmlBody[Elem])

def myOperationEndpoint(service: MyService) =
  endpoint.post
    .in(soapPath)
    .in(soapAction("MyOperation"))
    .in(xmlBody[MyRequest])
    .out(xmlBody[MyResponse])
    .errorOut(faultErrorOutput)
    .handle { req =>
      service.myOperation(req).left.map(toFaultElem)
    }
```

> **Important:** The catch-all endpoint below must be registered last — it
> matches any `SOAPAction`, so placing it earlier would shadow all real
> operations.


```scala
val unknownActionEndpoint =
  endpoint.post
    .in(soapPath)
    .in(header[String]("SOAPAction"))
    .out(statusCode(SC.InternalServerError).and(xmlBody[Elem]))
    .handleSuccess { action =>
      SoapFault.clientFault(s"Unknown SOAPAction: $action")
    }

def endpoints(service: MyService) = List(
  myOperationEndpoint(service),
  otherOperationEndpoint(service),
  unknownActionEndpoint
)
```

## SOAP error handlers

Framework-level errors (malformed XML, missing headers, unhandled exceptions)
must also produce SOAP faults instead of Tapir's default plain-text responses.
Three custom interceptors cover these cases.

The `isSoapRequest` check ensures that non-SOAP endpoints in the same
application still get default error responses — only requests to `/soap/*` paths
get SOAP-formatted errors:

```scala
import sttp.model.{Header, StatusCode}
import sttp.monad.IdentityMonad
import sttp.shared.Identity
import sttp.tapir.server.interceptor.decodefailure.*
import sttp.tapir.server.interceptor.exception.ExceptionHandler
import sttp.tapir.server.interceptor.reject.{DefaultRejectHandler, RejectHandler}
import sttp.tapir.server.model.ValuedEndpointOutput

private def isSoapRequest(request: sttp.tapir.model.ServerRequest): Boolean =
  request.pathSegments.headOption.contains("soap")

private def soapFaultResponse(fault: Elem): ValuedEndpointOutput[?] =
  ValuedEndpointOutput(
    statusCode.and(headers).and(stringBody),
    (StatusCode.InternalServerError,
     List(Header("Content-Type", "text/xml")),
     fault.toString())
  )
```

The decode failure handler reuses `DefaultDecodeFailureHandler.respond` to
decide *whether* to respond (vs. passing to the next endpoint), then replaces
the response body with a SOAP fault. The reject handler catches requests where
no endpoint matched. The exception handler wraps unexpected failures:

```scala
val decodeFailureHandler: DecodeFailureHandler[Identity] =
  val default = DefaultDecodeFailureHandler[Identity]
  DecodeFailureHandler.pure[Identity] { ctx =>
    if isSoapRequest(ctx.request) then
      DefaultDecodeFailureHandler.respond(ctx).map { _ =>
        val msg = DefaultDecodeFailureHandler.FailureMessages.failureMessage(ctx)
        soapFaultResponse(SoapFault.clientFault(msg))
      }
    else default.apply(ctx)(IdentityMonad)
  }

val rejectHandler: RejectHandler[Identity] =
  val default = DefaultRejectHandler.orNotFound[Identity]
  RejectHandler.pure[Identity] { ctx =>
    if isSoapRequest(ctx.request) then
      Some(soapFaultResponse(SoapFault.clientFault("No matching SOAP endpoint")))
    else default.apply(ctx)(IdentityMonad)
  }

val exceptionHandler: ExceptionHandler[Identity] =
  val default = DefaultExceptionHandler[Identity]
  ExceptionHandler.pure[Identity] { ctx =>
    if isSoapRequest(ctx.request) then
      Some(soapFaultResponse(SoapFault.serverFault("Internal server error")))
    else default.apply(ctx)(IdentityMonad)
  }
```

Wire them into the server options:

```scala
val options = NettySyncServerOptions.customiseInterceptors
  .decodeFailureHandler(SoapErrorHandlers.decodeFailureHandler)
  .rejectHandler(SoapErrorHandlers.rejectHandler)
  .exceptionHandler(SoapErrorHandlers.exceptionHandler)
  .options
```

## Testing SOAP endpoints

Use `TapirSyncStubInterpreter` to test SOAP endpoints in-process without
starting a server (see [Testing HTTP Endpoints](500-testing-http-endpoints.md)):

```scala
import munit.FunSuite
import sttp.client4.*
import sttp.tapir.server.stub4.TapirSyncStubInterpreter

class MyEndpointSuite extends FunSuite:

  private val service = MyService(NoOpProducer)

  private val stub =
    TapirSyncStubInterpreter()
      .whenServerEndpointsRunLogic(MyEndpoint.endpoints(service))
      .backend()

  private val serviceUrl = uri"http://test.com/soap/MyService"

  private def soapRequest(action: String, body: String) =
    basicRequest
      .post(serviceUrl)
      .header("SOAPAction", action)
      .contentType("application/xml")
      .body(body)

  test("successful operation returns 200 with response envelope") {
    val xml =
      """<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
        |                  xmlns:tns="http://example.com/my-service">
        |  <soapenv:Body>
        |    <tns:MyRequest>
        |      <tns:field1>value1</tns:field1>
        |    </tns:MyRequest>
        |  </soapenv:Body>
        |</soapenv:Envelope>""".stripMargin

    val response = soapRequest("\"MyOperation\"", xml).send(stub)
    assertEquals(response.code.code, 200)
    val body = response.body.getOrElse(fail("Expected response body"))
    assert(body.contains("MyResponse"))
  }
```

To test the SOAP error handlers, pass the custom interceptors to the stub:

```scala
import sttp.tapir.server.interceptor.CustomiseInterceptors
import sttp.shared.Identity

private val options =
  new CustomiseInterceptors[Identity, Any](_ => ())
    .decodeFailureHandler(SoapErrorHandlers.decodeFailureHandler)
    .rejectHandler(SoapErrorHandlers.rejectHandler)
    .exceptionHandler(SoapErrorHandlers.exceptionHandler)

private val stub =
  TapirSyncStubInterpreter(options)
    .whenServerEndpointsRunLogic(MyEndpoint.endpoints(service))
    .backend()

test("malformed XML returns SOAP Client Fault") {
  val response = basicRequest
    .post(serviceUrl)
    .header("SOAPAction", "\"MyOperation\"")
    .contentType("text/xml")
    .body("this is not xml")
    .send(stub)
  assertEquals(response.code.code, 500)
  val body = response.body.fold(identity, identity)
  assert(body.contains("soapenv:Client"))
}
```
