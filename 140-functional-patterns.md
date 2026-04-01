# Functional Patterns in Direct Style

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`

---

## Immutable state, scoped mutation

In effect-based Scala (Cats Effect, ZIO), mutable state lives in a `Ref` — an
atomic reference embedded in the effect type. Direct style doesn't have an
effect type, but the principle remains: keep state immutable, keep mutation
scoped.

> **Rules:**
> - `var` declarations must be inside methods (e.g. `run()`, processing loops),
>   **never** as class fields. Class-level `var`s break reasoning and
>   testability.
> - Use only immutable collections (`Map`, `Set`, `List`) in state. Never use
>   `mutable.Map`, `mutable.Set`, `mutable.Buffer`, or similar — use
>   immutable collections with `.updated()` / `+` / `-` instead.

The pattern: define state as an immutable case class, write functions that take
the old state and return a new one, and confine the `var` that threads state
through a processing loop to the smallest possible scope — a local variable
inside a method, never a class field:

```scala
case class ProcessingState(
    processed: Map[String, Long] = Map.empty,
    pending: Set[String] = Set.empty,
    errors: List[String] = Nil
)

class Processor(store: Store):
  def run(items: Iterator[Item]): Unit =
    var state = store.init()
    for item <- items do
      state = handle(state, item)
    state = store.update(state)
```

The `var` lives inside `run()`, not on the class. Nothing else can read or
write it — it exists only for the duration of the processing loop. In a
concurrent setting (e.g. `Flow.runForeach`), the same pattern works because
elements are processed sequentially on one thread, so no atomic reference is
needed.

## Pure state transitions

Each state transition is a method that takes the current state and returns a new
state. The method may perform side effects (writing to files, logging), but the
state management itself is pure — the caller decides what to do with the
returned state:

```scala
def handleItem(state: ProcessingState, item: Item): ProcessingState =
  if isDuplicate(state, item) then
    state.copy(processed = state.processed.updated(item.key, item.offset))
  else
    writeToLocal(item)
    state.copy(
      processed = state.processed.updated(item.key, item.offset),
      pending = state.pending + item.key
    )
```

Every branch returns a new `ProcessingState` via `.copy()`. The caller in
`run()` reassigns `state =` with the result. No field mutation, no shared
mutable state.

## Typed outcomes for state transitions

When a state transition has multiple outcomes — appended, skipped as duplicate,
dropped as late — the return type must distinguish them. Returning a bare
`ProcessingState` in every branch hides what happened and forces callers to
guess or rely on side-effect signals (metrics, logs) to know the outcome.

Encode outcomes as an ADT:

```scala
enum HandleResult:
  case Appended(state: ProcessingState)
  case Duplicate(state: ProcessingState)
  case Dropped
```

Each variant carries exactly the data it needs — `Dropped` carries no state
because nothing changed. The transition function returns the ADT:

```scala
def handleItem(state: ProcessingState, item: Item): HandleResult =
  if isLate(item) then HandleResult.Dropped
  else if isDuplicate(state, item) then HandleResult.Duplicate(state)
  else
    writeToLocal(item)
    HandleResult.Appended(
      state.copy(
        processed = state.processed.updated(item.key, item.offset),
        pending = state.pending + item.key
      )
    )
```

The caller pattern-matches to thread state and report metrics:

```scala
def run(items: Iterator[Item]): Unit =
  var state = init()
  for item <- items do
    handleItem(state, item) match
      case HandleResult.Appended(s) =>
        state = s
        metrics.itemProcessed(ProcessedReason.Appended)
      case HandleResult.Duplicate(s) =>
        state = s
        metrics.itemProcessed(ProcessedReason.Duplicate)
      case HandleResult.Dropped =>
        metrics.itemProcessed(ProcessedReason.Dropped)
```

The same principle applies to batch operations. A flush that can succeed or
partially fail should not return `(State, Boolean)` — use an ADT:

```scala
enum FlushResult:
  case Success(state: ProcessingState)
  case PartialFailure(state: ProcessingState, failedKeys: List[String])
```

> **Rule of thumb:** if a function returns the same type regardless of what
> happened, and the caller would need to inspect side effects (logs, metrics,
> external state) to learn the outcome, the return type is lying. Add an ADT.

## Accumulating state over collections

When building or updating state from a collection, `foldLeft` or a
tail-recursive function both work — the key is that each step takes the
previous state and returns a new one:

```scala
def init(): ProcessingState =
  computeActiveKeys().foldLeft(ProcessingState()): (acc, key) =>
    val existing = loadExisting(key)
    if existing.nonEmpty then
      acc.copy(processed = acc.processed.updated(key, existing.maxOffset))
    else acc
```

## Functional combinators over mutable loops

When processing a collection with a pass/fail outcome per element, use
functional combinators instead of `var` + imperative loop + mutation:

```scala
// Bad: mutable accumulator
def flushAll(buckets: List[Bucket]): Boolean =
  var allSuccess = true
  for bucket <- buckets do
    try upload(bucket)
    catch case e: Exception =>
      logger.error("Failed: {}", e.getMessage)
      allSuccess = false
  allSuccess

// Good: partition into successes and failures
def flushAll(buckets: List[Bucket]): FlushResult =
  val (successes, failures) = buckets.partition(tryUpload)
  if failures.isEmpty then FlushResult.Success
  else FlushResult.PartialFailure(failures.map(_.key))

private def tryUpload(bucket: Bucket): Boolean =
  try
    upload(bucket)
    true
  catch case e: Exception =>
    logger.error("Failed to upload {}: {}", bucket.key, e.getMessage)
    false
```

Similarly, filtering a map with a predicate:

```scala
// Bad: var + imperative removal
var newBuckets = state.buckets
for (key, bucket) <- state.buckets do
  if isSealed(key) && !bucket.isDirty then
    newBuckets = newBuckets.removed(key)

// Good: filterNot
val newBuckets = state.buckets.filterNot: (key, bucket) =>
  isSealed(key) && !bucket.isDirty
```

The functional version is shorter, avoids intermediate `var`s, and makes the
intent immediately clear — "keep buckets that are not sealed-and-clean."

## Handling failures in transitions

When a state transition can fail, return `Either` and use `either` blocks to
compose multiple transitions with short-circuiting (see [Error
Handling](200-error-handling.md)):

```scala
import ox.either
import ox.either.ok

def handleItem(state: ProcessingState, item: Item)
    : Either[ProcessingError, ProcessingState] =
  either:
    val validated = validate(item).ok()
    val written = writeToLocal(validated).ok()
    state.copy(
      processed = state.processed.updated(item.key, item.offset),
      pending = state.pending + item.key
    )
```

The processing loop then short-circuits on the first error, or threads the
state through:

```scala
def run(items: Iterator[Item]): Either[ProcessingError, ProcessingState] =
  either:
    var state = init()
    for item <- items do
      state = handleItem(state, item).ok()
    flush(state).ok()
```

## Testing pure state transitions

Because state transitions are functions (`ProcessingState => ProcessingState`),
tests don't need to mock internal state or intercept side effects. They call the
transition functions directly, threading state explicitly:

```scala
test("duplicates are skipped"):
  var state = processor.init()
  state = processor.handleItem(state, makeItem(offset = 5))
  state = processor.handleItem(state, makeItem(offset = 10))
  state = processor.handleItem(state, makeItem(offset = 10)) // dup
  state = processor.handleItem(state, makeItem(offset = 11))

  assertEquals(readStored().size, 3)
```

No broker, no running application. The same pattern tests crash recovery —
create state from one instance, then feed overlapping items through a fresh
instance and verify deduplication:

```scala
test("restart from store, skip already-processed items"):
  // First run: process 1-3, flush
  var state1 = processor1.init()
  state1 = processor1.handleItem(state1, makeItem(offset = 1))
  state1 = processor1.handleItem(state1, makeItem(offset = 2))
  state1 = processor1.handleItem(state1, makeItem(offset = 3))
  state1 = processor1.flush(state1)

  // Second run: fresh local state, same store (has 1-3)
  var state2 = processor2.init()
  state2 = processor2.handleItem(state2, makeItem(offset = 2)) // dup
  state2 = processor2.handleItem(state2, makeItem(offset = 3)) // dup
  state2 = processor2.handleItem(state2, makeItem(offset = 4)) // new
  state2 = processor2.handleItem(state2, makeItem(offset = 5)) // new

  assertEquals(readStored().map(_.offset), List(1L, 2L, 3L, 4L, 5L))
```

## Pushing side effects to the edge

The state transition functions are testable because the side effects they depend
on (storage, messaging) are behind traits:

```scala
trait Store:
  def download(key: String): Option[Path]
  def upload(key: String, source: Path): Unit
```

Tests substitute an in-memory implementation that can also simulate failures:

```scala
class InMemoryStore extends Store:
  val contents = mutable.Map[String, Array[Byte]]()
  var failUploads = false

  def upload(key: String, source: Path): Unit =
    if failUploads then throw new IOException("Simulated failure")
    contents(key) = Files.readAllBytes(source)

  def download(key: String): Option[Path] =
    contents.get(key).map: bytes =>
      val path = Files.createTempFile("test", ".dat")
      Files.write(path, bytes)
      path
```

This lets tests verify retry behavior by toggling `failUploads` between
flushes — no real infrastructure, no test containers.

## Typed enums over stringly-typed parameters

When an interface accepts a fixed set of labels — metric reasons, flush
outcomes, event types — use an `enum`, not a `String`. A typo in a string
compiles silently and produces a wrong metric label; a typo in an enum is a
compile error.

```scala
// Bad: stringly-typed
trait Metrics:
  def itemProcessed(reason: String): Unit    // "appended", "dropped", "duplicate"
  def flushOutcome(outcome: String): Unit    // "success", "failure"

metrics.itemProcessed("droped")  // compiles, wrong label — silent bug
```

```scala
// Good: enum-typed
enum ProcessedReason:
  case Appended, Dropped, Duplicate

enum FlushOutcome:
  case Success, Failure

trait Metrics:
  def itemProcessed(reason: ProcessedReason): Unit
  def flushOutcome(outcome: FlushOutcome): Unit

metrics.itemProcessed(ProcessedReason.Droped)  // compile error
```

This applies to any boundary where a fixed vocabulary is passed across a trait
— metric labels, log categories, status codes. The enum is the source of truth;
the `toString` conversion happens once, at the edge (e.g. in the OpenTelemetry
implementation), not at every call site.

## Small, well-named functions

A function that checks three conditions and performs an action is four functions:
three named predicates and one orchestrator. The orchestrator reads like an
outline:

```scala
// Bad: one function doing everything
def handleItem(state: State, item: Item): HandleResult =
  val key = keyFrom(item.timestamp)
  val hourStart = parseHourStart(key)
  val cutoff = Instant.now().minus(Duration.ofHours(maxLateHours))
  if hourStart.isBefore(cutoff) then
    logger.debug("Dropping late item")
    return HandleResult.Dropped
  val bucket = getOrCreateBucket(state, key)
  if bucket.isDuplicate(item.partition, item.offset) then
    return HandleResult.Duplicate(state.withBucket(key, bucket))
  bucket.append(item)
  HandleResult.Appended(state.withBucket(key, bucket))
```

```scala
// Good: named predicates, flat orchestration
def handleItem(state: State, item: Item): HandleResult =
  val key = keyFrom(item.timestamp)
  if isLate(key) then HandleResult.Dropped
  else
    val bucket = getOrCreateBucket(state, key)
    if bucket.isDuplicate(item.partition, item.offset) then
      HandleResult.Duplicate(state.withBucket(key, bucket))
    else
      appendToBucket(bucket, item)
      HandleResult.Appended(state.withBucket(key, bucket))

private def isLate(key: String): Boolean =
  HourKey.hourStart(key).isBefore(Instant.now().minus(Duration.ofHours(maxLateHours)))

private def appendToBucket(bucket: Bucket, item: Item): Unit =
  val record = buildRecord(item)
  bucket.append(item.partition, item.offset, record)
```

Benefits: each predicate is independently testable, the orchestrator has no
`return` statements, and the reader sees intent ("is this late?") before
mechanism ("parse hour, compare to cutoff").
