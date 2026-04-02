# Shared State Across Fibers

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `AtomicReference` integration,
  structured concurrency scopes, `Channel` for inter-fiber communication

---

## The problem: concurrent state mutation

When multiple fibers (e.g. a consumer loop and a periodic flush timer) need to
share mutable state, naive approaches create data races. Virtual threads don't
change the JVM memory model — `var` writes are not guaranteed visible across
threads without synchronization.

## Single-owner pattern with channels

The safest approach is to confine state mutation to a single fiber and
communicate via Ox channels. Other fibers send signals; the owner fiber acts
on them:

```scala
import ox.*
import ox.channels.{Channel, ChannelClosedException}

def run()(using Ox): Unit =
  var state = initialState()

  val flushSignal = Channel.buffered[Unit](1)

  // Timer sends flush signals
  fork:
    forever:
      sleep(flushInterval)
      try flushSignal.send(())
      catch case _: ChannelClosedException => ()
  .discard

  // Single owner: consumer loop processes events AND flushes
  dataSource.foreach: item =>
    state = processItem(state, item)

    // Non-blocking check for flush signal
    flushSignal.tryReceive() match
      case _: ChannelClosedException => ()
      case _ => state = flush(state)
```

> **Important:** State is owned by one fiber only — no concurrent reads or
> writes. The flush timer does not touch state directly; it sends a signal
> that the owner fiber acts on at a safe point between events.

## AtomicReference pattern

When the single-owner pattern doesn't fit (e.g. the data source itself blocks
the fiber and cannot interleave flush checks), use `AtomicReference` with
**atomic read-modify-write** operations:

```scala
import java.util.concurrent.atomic.AtomicReference

def run()(using Ox): Unit =
  val stateRef = AtomicReference(initialState())

  // Flush fiber: atomic read-modify-write
  fork:
    forever:
      sleep(flushInterval)
      stateRef.updateAndGet(flush).discard
  .discard

  // Consumer fiber: atomic read-modify-write
  dataSource.foreach: item =>
    stateRef.updateAndGet(state => processItem(state, item)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another fiber can modify the state between the get and set, and the set
> silently overwrites those changes. Always use `updateAndGet` or
> `getAndUpdate` for atomic read-modify-write.

This requires that `processItem` and `flush` are **pure functions** from old
state to new state — they cannot read the `AtomicReference` themselves, or
they'll see stale data:

```scala
// Pure state transitions — no external state reads
def processItem(state: State, item: Item): State =
  state.copy(items = state.items.updated(item.key, item.value))

def flush(state: State): State =
  storage.upload(state.pendingData)
  state.copy(pendingData = Map.empty, dirty = false)
```

## Extracting return values from `updateAndGet`

When a fiber needs both the updated state and a derived value (e.g. the outcome
of processing an event), return a tuple in the state and extract it afterward:

```scala
case class StateWithResult(state: State, lastOutcome: Outcome)

stateRef.updateAndGet: current =>
  val (newState, outcome) = processItem(current.state, item)
  StateWithResult(newState, outcome)
```

This avoids the anti-pattern of using a mutable `var` to smuggle values out of
the `updateAndGet` lambda.

## When to use which

| Pattern | Use when |
|---------|----------|
| **Single-owner + channels** | One fiber owns all state; others only send signals. Simplest, no races possible. |
| **AtomicReference + updateAndGet** | Multiple fibers must independently update state; state transitions are pure functions. |
| **Ox supervised + actor** | Complex protocols where fibers need request/response interaction, not just fire-and-forget signals. |

> **Important:** Avoid sharing mutable state across fibers whenever possible.
> Restructure the design so one fiber owns the state and others communicate
> through channels. Use `AtomicReference` only when the single-owner pattern
> is impractical.
