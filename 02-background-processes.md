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

The `using Ox` parameter is the capability token that proves you're inside a
supervised scope. It's required by `fork`, `useInScope`, and other scope-aware
operations.

`never` blocks the main fork indefinitely. Without it, the `run` method would
return, the scope would end, and all forked processes (HTTP server, email
sender) would be interrupted.

`OxApp` handles SIGINT/SIGTERM by interrupting the root scope, which triggers
orderly shutdown: all forks are interrupted, resources are released in reverse
order.

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
user forks do). `forever` repeats the block indefinitely. `sleep` blocks the
virtual thread for the configured interval without consuming an OS thread.

The `try`/`catch` around the body is important: without it, a single exception
would crash the fork and — because it's supervised — terminate the entire
application. By catching `NonFatal` exceptions, the fork logs the error and
continues to the next iteration.

The return type `Fork[Nothing]` indicates the fork never completes normally (it
loops forever or is interrupted).

## Starting multiple background processes

Multiple processes are started by calling `fork` multiple times within the same
scope:

`foreverPeriodically` is the application-defined helper shown above — it combines
Ox's `fork`, `forever`, and `sleep` with error handling:

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

Both forks run concurrently within the application's root scope. They share the
same lifecycle — when the application shuts down, both are interrupted.

## Fork types

Ox distinguishes between fork types based on two dimensions:

**Daemon vs. User:**
- `fork` — daemon fork. The scope can end while daemon forks are still running
  (they get interrupted).
- `forkUser` — user fork. The scope waits for all user forks to complete before
  ending.

**Supervised vs. Unsupervised:**
- Supervised forks (default) — if a fork fails with an exception, the enclosing
  scope is terminated and the exception is re-thrown.
- `forkUnsupervised` — failures don't affect the scope; exceptions only surface
  when `.join()` is called.

For background processes, `fork` (supervised daemon) is the right choice: the
process should run until the application shuts down, and if it fails
unexpectedly (after the `try`/`catch`), the application should terminate rather
than silently continue without the background process.

## The HTTP server as a fork

The Netty HTTP server is also started within the scope:

```scala
def start()(using Ox): NettySyncServerBinding =
  NettySyncServer(serverOptions, NettyConfig.default.host(config.host).port(config.port))
    .addEndpoints(allEndpoints)
    .start()
```

`NettySyncServer.start()` takes `(using Ox)` — it internally forks a user thread
to accept connections. When the scope ends, the server is stopped gracefully.
