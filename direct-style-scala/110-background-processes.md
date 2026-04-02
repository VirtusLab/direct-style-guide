# Background Processes

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`, `fork`, `forkUser`,
  `forever`, `sleep`, `never`

---

## Application entry point

`OxApp` provides the application's root scope — a `supervised` block that
manages all forks and resources:

```scala
object Main extends OxApp.Simple:
  InheritableMDC.init

  override def run(using Ox): Unit =
    val deps = Dependencies.create
    deps.emailService.startProcesses()
    deps.httpApi.start().discard
    never
```

`never` blocks the main fork indefinitely — without it, `run` returns and the
scope ends, interrupting all forks.

## Daemon forks with forever

Background processes use `fork` + `forever` + `sleep` to create periodic loops:

```scala
private def foreverPeriodically(errorMsg: String)(t: => Unit)(using Ox): Fork[Nothing] =
  fork {
    forever {
      sleep(config.emailSendInterval)
      try t
      catch case NonFatal(e) => logger.error(errorMsg, e)
    }
  }
```

The interval (`config.emailSendInterval`) comes from the service's own
configuration — each background process can define its own schedule.

`fork` creates a daemon fork — it doesn't prevent the scope from ending (only
user forks do). `forever` repeats the block indefinitely.

> **Warning:** The `try`/`catch` is essential: without it, a single exception
> would crash the fork and terminate the application (supervised scope).
> Catching `NonFatal` lets the fork continue to the next iteration.

## Starting multiple background processes

Multiple processes are started by forking multiple times within the same scope:

```scala
def startProcesses()(using Ox): Unit =
  foreverPeriodically("Exception when sending emails") {
    sendBatch()
  }.discard

  foreverPeriodically("Exception when counting emails") {
    val count = db.transact(emailModel.count())
    metrics.emailQueueGauge.set(count.toDouble)
  }.discard
```

Both use `fork` (daemon). If a fork fails after the `try`/`catch`, the
supervised scope terminates the application rather than silently continuing.
