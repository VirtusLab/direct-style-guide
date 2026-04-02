# Shared State Across Fibers

## Dependencies

- `"com.softwaremill.ox" %% "core"` — structured concurrency scopes

---

## The problem: concurrent state mutation

When multiple fibers (e.g. a request handler and a periodic background task)
need to share mutable state, naive approaches create data races. Virtual threads
don't change the JVM memory model — `var` writes are not guaranteed visible
across threads without synchronization.

## Single-owner pattern

The safest approach is to confine all state mutation to a single fiber. Other
fibers only set a lightweight flag; the owner fiber checks it at safe points:

```scala
import ox.*
import java.util.concurrent.atomic.AtomicBoolean

def run()(using Ox): Unit =
  var state = initialState()

  val saveRequested = AtomicBoolean(false)

  // Timer sets the flag periodically
  fork:
    forever:
      sleep(saveInterval)
      saveRequested.set(true)
  .discard

  // Single owner: main loop updates state AND checks the flag
  inputSource.foreach: item =>
    state = process(state, item)

    if saveRequested.compareAndSet(true, false) then
      state = save(state)
```

> **Important:** State is owned by one fiber only — no concurrent reads or
> writes. The timer does not touch state directly; it sets a flag that the
> owner fiber checks at a safe point between items.

## AtomicReference pattern

When the single-owner pattern doesn't fit (e.g. the input source itself blocks
the fiber and cannot interleave flag checks), use `AtomicReference` with
**atomic read-modify-write** operations:

```scala
import java.util.concurrent.atomic.AtomicReference

def run()(using Ox): Unit =
  val stateRef = new AtomicReference(initialState())

  // Background fiber: atomic read-modify-write
  fork:
    forever:
      sleep(saveInterval)
      stateRef.updateAndGet(save).discard
  .discard

  // Main fiber: atomic read-modify-write
  inputSource.foreach: item =>
    stateRef.updateAndGet(state => process(state, item)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another fiber can modify the state between the get and set, and the set
> silently overwrites those changes. Always use `updateAndGet` or
> `getAndUpdate` for atomic read-modify-write.

This requires that `process` and `save` are **pure functions** from old state to
new state — they cannot read the `AtomicReference` themselves, or they'll see
stale data.

> **Warning:** `updateAndGet` may retry the function under contention (CAS
> loop). The function passed to it must be side-effect-free. If `save` performs
> I/O (e.g. writing to storage), do it outside the atomic update — compute the
> new state atomically, then perform I/O with the result:

```scala
// Pure state transition — no I/O inside updateAndGet
def markSaved(state: State): State =
  state.copy(pending = Set.empty, lastSavedAt = Instant.now())

// I/O happens outside the atomic update
val snapshot = stateRef.get()
storage.write(snapshot.pending)
stateRef.updateAndGet(markSaved).discard
```

## Extracting return values from `updateAndGet`

When a fiber needs both the updated state and a derived value (e.g. whether the
item was accepted or rejected), return both in the state and extract afterward:

```scala
case class StateWithResult(state: State, lastResult: Result)

val updated = stateRef.updateAndGet: current =>
  val (newState, result) = process(current.state, item)
  StateWithResult(newState, result)
val result = updated.lastResult
```

This avoids the anti-pattern of using a mutable `var` to smuggle values out of
the `updateAndGet` lambda.

## When to use which

| Pattern | Use when |
|---------|----------|
| **Single-owner + flag** | One fiber owns all state; others only signal. Simplest, no races possible. |
| **AtomicReference + updateAndGet** | Multiple fibers must independently update state; state transitions are pure functions with no I/O. |
| **Ox supervised + actor** | Complex protocols where fibers need request/response interaction, not just fire-and-forget signals. |

> **Important:** Avoid sharing mutable state across fibers whenever possible.
> Restructure the design so one fiber owns the state and others communicate
> through flags or channels. Use `AtomicReference` only when the single-owner
> pattern is impractical.
