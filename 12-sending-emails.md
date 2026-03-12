# Sending Emails

## Dependencies

- `"com.sun.mail" % "javax.mail"` — JavaMail API for SMTP delivery
- `"com.softwaremill.sttp.client4" %% "core"` — HTTP client for Mailgun API
  delivery
- `"com.softwaremill.ox" %% "core"` — `fork`, `forever`, `sleep` for background
  email processing

---

## Architecture

Emails are not sent synchronously during request handling. Instead, they are
scheduled — stored in a database table — and sent asynchronously by a background
process. This decouples the HTTP response time from email delivery latency and
provides resilience: if email delivery fails, the emails remain in the queue and
are retried on the next batch.

## Scheduling emails

The `EmailScheduler` trait defines a single method that inserts an email into
the database within the current transaction:

```scala
trait EmailScheduler:
  def schedule(data: EmailData)(using DbTx): Unit
```

`EmailData` contains the recipient, subject, and content:

```scala
case class EmailData(recipient: String, subject: String, content: String)
```

Because `schedule` takes `(using DbTx)`, it participates in the same transaction
as the business logic that triggers it. If user registration fails and the
transaction rolls back, the welcome email is never scheduled.

## Email templates

Templates are plain text files loaded from the classpath, with `{{placeholder}}`
substitution:

```scala
class EmailTemplates:
  def registrationConfirmation(userName: String): EmailSubjectContent =
    EmailTemplateRenderer("registrationConfirmation", Map("userName" -> userName))

  def passwordReset(userName: String, resetLink: String): EmailSubjectContent =
    EmailTemplateRenderer("resetPassword",
      Map("userName" -> userName, "resetLink" -> resetLink))
```

`EmailTemplateRenderer` loads `/templates/email/{name}.txt`, replaces `{{key}}`
with values, and splits the first line as the subject and the rest as the body.
A shared signature is appended to all emails.

## Pluggable email senders

The `EmailSender` trait abstracts over the delivery mechanism:

```scala
trait EmailSender:
  def apply(email: EmailData): Unit

object EmailSender:
  def create(sttpBackend: SyncBackend, config: EmailConfig): EmailSender =
    if config.mailgun.enabled then new MailgunEmailSender(config.mailgun, sttpBackend)
    else if config.smtp.enabled then new SmtpEmailSender(config.smtp)
    else DummyEmailSender
```

Three implementations:
- **`MailgunEmailSender`** — sends via Mailgun's HTTP API using sttp
- **`SmtpEmailSender`** — sends via SMTP using JavaMail
- **`DummyEmailSender`** — logs the email and stores it in memory (for tests)

The factory method selects the implementation based on configuration. In tests,
neither Mailgun nor SMTP is enabled, so `DummyEmailSender` is used
automatically.

## The Mailgun sender

`MailgunEmailSender` uses sttp's `SyncBackend` to call the Mailgun API. Because
the same backend is used for all outgoing HTTP calls, it benefits from the
observability instrumentation (tracing, metrics) configured at the application
level (see [OpenTelemetry Observability](15-opentelemetry-observability.md)):

```scala
class MailgunEmailSender(config: MailgunConfig, sttpBackend: SyncBackend)
    extends EmailSender:
  override def apply(email: EmailData): Unit =
    basicRequest.auth
      .basic("api", config.apiKey.value)
      .post(uri"${config.url}")
      .response(asStringOrFail)
      .body(Map(
        "from" -> s"${config.senderDisplayName} <${config.senderName}@${config.domain}>",
        "to" -> email.recipient,
        "subject" -> email.subject,
        "html" -> email.content
      ))
      .send(sttpBackend)
      .discard
```

## Background processing

`EmailService.startProcesses` launches two background forks within the Ox scope
(see [Background Processes](02-background-processes.md)):

```scala
class EmailService(...) extends EmailScheduler:
  def startProcesses()(using Ox): Unit =
    foreverPeriodically("Exception when sending emails") {
      sendBatch()
    }.discard

    foreverPeriodically("Exception when counting emails") {
      val count = db.transact(emailModel.count())
      metrics.emailQueueGauge.set(count.toDouble)
    }.discard
```

`foreverPeriodically` uses the `fork` + `forever` + `sleep` pattern described in
[Background Processes](02-background-processes.md) to create a daemon loop. The
first fork sends queued emails in batches; the second updates a gauge metric
with the current queue size.

## Batch sending

`sendBatch` fetches a configurable number of emails from the database, sends
them, and deletes the sent ones:

```scala
def sendBatch(): Unit =
  val emails = db.transact(emailModel.find(config.batchSize))
  if emails.nonEmpty then logger.info(s"Sending ${emails.size} emails")
  emails.map(_.data).foreach(emailSender.apply)
  db.transact(emailModel.delete(emails.map(_.id)))
```

The find-send-delete pattern is simple but effective: if sending fails partway
through, unsent emails remain in the database and are picked up by the next
batch. The batch size and interval are configurable via `EmailConfig`.

## The email database model

Emails are stored in a `scheduled_emails` table. The model provides `insert`,
`find`, `count`, and `delete` operations using the database patterns from [SQL
Persistence](10-sql-persistence.md). All methods take `(using DbTx)` to
participate in transactions.

## Testing emails

In tests, `DummyEmailSender` captures sent emails in a thread-safe queue. Tests
trigger the batch send manually and then assert on the captured emails:

```scala
dependencies.emailService.sendBatch()
DummyEmailSender.findSentEmail(email, s"registration confirmation for user $login")
  .isDefined shouldBe true
```

This avoids any asynchronous waiting — the test controls exactly when emails are
sent.
