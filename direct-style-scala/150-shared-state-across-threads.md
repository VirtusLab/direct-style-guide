# Concurrency and Inter-Thread Communication

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `Flow`, `supervised`, `fork`, channels,
  actors

---

## Flows

Flows are the preferred way to express concurrent data processing in Ox. A
`Flow[T]` describes a lazy, composable pipeline that handles concurrency,
backpressure, and error propagation behind a declarative API:

```scala
import ox.flow.Flow

Flow.fromValues(1, 2, 3, 5, 6)
  .filter(_ % 2 == 0)
  .map(_ * 10)
  .runForeach(println)
```

Flows are lazy — no elements are emitted until a `.run*` method is called.

### Concurrent processing with mapPar

`mapPar` runs a mapping function across multiple virtual threads, handling fork
management and error propagation automatically:

```scala
Flow.fromIterable(urls)
  .mapPar(8)(url => httpClient.get(url))
  .runForeach(response => process(response))
```

Without Flows, this would require manually creating a `supervised` scope,
forking workers, coordinating via channels, and handling errors — all of which
`mapPar` encapsulates.

### Merging multiple event sources

`merge` combines two flows into one, processing elements from both
concurrently. This replaces manual multi-channel select patterns:

```scala
val userEvents: Flow[Event] = Flow.fromSource(userChannel)
val systemEvents: Flow[Event] = Flow.fromSource(systemChannel)

userEvents
  .merge(systemEvents)
  .runForeach(event => handle(event))
```

### Stateful processing

`mapStateful` threads state through a flow without any shared mutable state:

```scala
Flow.fromIterable(items)
  .mapStateful(initialState()): (state, item) =>
    val newState = process(state, item)
    (newState, newState.lastResult)
  .runForeach(result => emit(result))
```

### Periodic operations with tick

`Flow.tick` creates a periodic signal that can be merged with a data flow:

```scala
val data: Flow[Event] = eventSource()
val flushSignal: Flow[Event] = Flow.tick(flushInterval, FlushTick)

data
  .merge(flushSignal)
  .mapStateful(initialState()):
    case (state, FlushTick)    => val s = flush(state); (s, None)
    case (state, DataEvent(e)) => val s = process(state, e); (s, Some(s.lastResult))
  .collect { case Some(result) => result }
  .runForeach(emit)
```

> **Important:** Use Flows as the default for concurrent data processing. They
> handle fork lifecycle, error propagation, and backpressure automatically.
> Drop to channels or manual forks only when Flow's declarative API doesn't
> fit your use case.

## Channels

Channels are the lower-level building block underneath Flows. Use them when you
need imperative control over sending and receiving — for example, custom
protocols or integration with callback-based APIs:

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

Channels and Flows interoperate: `Flow.fromSource(channel)` wraps a channel in
a Flow, and `flow.runToChannel()` materialises a Flow into a channel.

### Signaling between threads

Use a channel instead of a shared flag when one thread needs to signal another:

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

  fork:
    repeatWhile:
      saveTrigger.receiveSafe() match
        case ChannelClosed.Done => false
        case _                 => save(state); true
  .discard
```

## Actors

When multiple threads need serialized access to a mutable object, use Ox's
built-in `Actor`. It guarantees that method invocations happen one at a time,
even when called from multiple threads:

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

  val total = ref.ask(_.current)
```

`ask` blocks until the invocation completes and returns the result. `tell`
schedules the invocation without waiting — use it for fire-and-forget
operations.

## AtomicReference as a last resort

For simple cases where channels, actors, or Flows are overkill (e.g. a shared
counter read by many threads), `AtomicReference` with atomic read-modify-write
operations works:

```scala
import java.util.concurrent.atomic.AtomicReference

val stateRef = AtomicReference(initialState())

stateRef.updateAndGet(state => process(state, item)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another thread can modify the state between the get and set, silently
> overwriting those changes. Always use `updateAndGet` or `getAndUpdate`.

> **Warning:** `updateAndGet` may retry the function under contention (CAS
> loop). The function passed to it MUST be side-effect-free — no I/O, no
> logging, no channel operations.

## Scope propagation

Only propagate `(using Ox)` when a method genuinely needs to start forks or
register resources in the caller's scope. Otherwise, create a local
`supervised` block:

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
| **Flow** | Data processing pipelines, concurrent mapping, merging streams, stateful transformations. The default choice. |
| **Channel** | Imperative send/receive, custom protocols, callback integration. Lower-level than Flows. |
| **Actor** | Multiple threads need serialized access to a mutable object. |
| **AtomicReference** | Simple shared value with pure update functions. No I/O in updates. |
