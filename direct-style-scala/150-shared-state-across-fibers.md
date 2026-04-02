# Concurrency and Inter-Fiber Communication

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`, `fork`, channels, actors

---

## Channels

Channels are the primary mechanism for communication between fibers in Ox. A
channel is a typed, backpressure-aware queue that supports completion and error
propagation:

```scala
import ox.*
import ox.channels.*

supervised:
  val ch = Channel.bufferedDefault[String]

  fork:
    ch.send("hello")
    ch.send("world")
    ch.done()

  var msg = ch.receiveSafe()
  while msg != ChannelClosed.Done do
    println(msg)
    msg = ch.receiveSafe()
```

`send` blocks when the buffer is full; `receive` blocks when the buffer is
empty. Since Ox runs on virtual threads, blocking is cheap.

### Signaling between fibers

Use a channel instead of a shared flag when one fiber needs to signal another.
For example, a timer that triggers periodic saves:

```scala
supervised:
  val saveTrigger = Channel.rendezvous[Unit]

  fork:
    forever:
      sleep(saveInterval)
      saveTrigger.send(())
  .discard

  var state = init()
  inputSource.foreach: item =>
    state = process(state, item)

  // Meanwhile, a separate fiber handles save triggers:
  fork:
    repeatWhile:
      saveTrigger.receiveSafe() match
        case ChannelClosed.Done => false
        case _                 => save(state); true
  .discard
```

> **Important:** Prefer channels over `AtomicBoolean` flags or shared `var`s.
> Channels are type-safe, support backpressure, and integrate with Ox's
> structured concurrency — when the scope ends, channel operations are
> interrupted cleanly.

### Request-response between fibers

When a fiber needs a result back, send a response channel along with the
request:

```scala
case class Request(query: String, replyTo: Channel[Result])

supervised:
  val requests = Channel.bufferedDefault[Request]

  // Worker fiber
  fork:
    repeatWhile:
      requests.receiveSafe() match
        case ChannelClosed.Done => false
        case req: Request =>
          val result = processQuery(req.query)
          req.replyTo.send(result)
          true

  // Client fiber
  val replyTo = Channel.rendezvous[Result]
  requests.send(Request("lookup", replyTo))
  val result = replyTo.receive()
```

## Actors

When multiple fibers need serialized access to a mutable object, use Ox's
built-in `Actor`. It guarantees that method invocations happen one at a time,
even when called from multiple fibers:

```scala
import ox.channels.Actor

class StateHolder:
  private var counter: Int = 0

  def increment(delta: Int): Int =
    counter += delta
    counter

  def current: Int = counter

supervised:
  val ref = Actor.create(new StateHolder)

  fork(ref.ask(_.increment(5))).discard
  fork(ref.ask(_.increment(3))).discard

  // later
  val total = ref.ask(_.current)
```

`ask` blocks until the invocation completes and returns the result. `tell`
schedules the invocation without waiting — use it for fire-and-forget
operations.

> **Important:** Actors use channels internally — they are not a separate
> concurrency primitive, but a convenience for serialized access patterns.
> Prefer actors over manual locking or `synchronized` blocks.

## AtomicReference as a last resort

For simple cases where a full channel or actor is overkill (e.g. a shared
counter or a single configuration value read by many fibers), `AtomicReference`
with atomic read-modify-write operations works:

```scala
import java.util.concurrent.atomic.AtomicReference

val stateRef = AtomicReference(initialState())

// In fiber A:
stateRef.updateAndGet(state => process(state, item)).discard

// In fiber B:
stateRef.updateAndGet(state => process(state, otherItem)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another fiber can modify the state between the get and set, silently
> overwriting those changes. Always use `updateAndGet` or `getAndUpdate`.

> **Warning:** `updateAndGet` may retry the function under contention (CAS
> loop). The function passed to it must be side-effect-free — no I/O, no
> logging, no channel operations.

## Scope propagation

Only propagate `(using Ox)` when a method genuinely needs to start forks or
register resources in the caller's scope. Prefer creating local, nested
`supervised` blocks instead:

```scala
// Avoid — leaks concurrency scope to the caller:
def processAll(items: List[Item])(using Ox): Unit =
  items.foreach(item => fork(handle(item)).discard)

// Prefer — concurrency is contained within the method:
def processAll(items: List[Item]): Unit =
  supervised:
    items.foreach(item => fork(handle(item)).discard)
```

> **Important:** `(using Ox)` in a method signature means "I will start
> forks or register resources in your scope." If the method manages its own
> concurrency lifecycle, use a local `supervised` block instead.

## Choosing the right pattern

| Pattern | Use when |
|---------|----------|
| **Channel** | Fibers need to communicate data or signals. The default choice. |
| **Actor** | Multiple fibers need serialized access to a mutable object. |
| **AtomicReference** | Simple shared value with pure update functions. No I/O in updates. |
