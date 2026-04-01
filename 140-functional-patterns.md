# Functional Patterns in Direct Style

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `either`, `.ok()`

---

## Small, single-responsibility methods

Each method should do one thing — either perform a single operation, or
orchestrate a sequence of steps. A method that does both is too large. Names
should describe *what* the method does, not *how*:

```scala
// Too large — mixes validation, persistence, and notification:
def handle(order: Order): Either[Fail, Confirmation] =
  either:
    if order.items.isEmpty then Fail.IncorrectInput("empty order").fail()
    if order.total.isNegative then Fail.IncorrectInput("negative total").fail()
    val saved = db.transactEither(orderModel.insert(order)).ok()
    emailScheduler.schedule(EmailData(order.email, subject, body))
    Confirmation(saved.id)

// Better — the orchestrator reads like a summary, each step is testable:
def handle(order: Order): Either[Fail, Confirmation] =
  either:
    validate(order).ok()
    val saved = persist(order).ok()
    notifyCustomer(order)
    Confirmation(saved.id)
```

The orchestrating method reads like a summary of the workflow. Each extracted
method can be tested independently and its name serves as documentation for the
reader.

## Domain types over primitives

Avoid `String`, `Int`, `Long`, and `Boolean` for domain values. Wrap them in
opaque types or enums so the compiler prevents mixing them up:

```scala
// Stringly-typed — easy to swap email and login by mistake:
def register(login: String, email: String, password: String): Unit

// Type-safe — the compiler catches argument order mistakes:
opaque type Login <: String = String
opaque type Email <: String = String
opaque type Password <: String = String

def register(login: Login, email: Email, password: Password): Unit
```

The same applies to numeric types and identifiers:

```scala
// Primitive — nothing prevents passing a user ID where an order ID is expected:
def findOrder(orderId: Long): Option[Order]

// Domain-typed — Id[User] and Id[Order] are distinct types:
opaque type Id[T] = String
def findOrder(orderId: Id[Order]): Option[Order]
```

Avoid boolean parameters — they erase meaning at the call site. Use enums or
named alternatives:

```scala
// Boolean blindness — what does `true` mean here?
def sendEmail(email: Email, urgent: Boolean): Unit
sendEmail(email, true)

// Self-documenting:
enum Priority:
  case Normal, Urgent

def sendEmail(email: Email, priority: Priority): Unit
sendEmail(email, Priority.Urgent)
```

## Truthful signatures and honest types

Method signatures should not lie. If a method can fail, its return type must
reflect that — return `Either[E, T]`, not `T` with a hidden exception. If a
value can be absent, use `Option[T]`, not `null` or a sentinel value:

```scala
// Dishonest — caller has no idea this can fail:
def findUser(id: Id[User]): User

// Honest — failure is visible in the type:
def findUser(id: Id[User])(using DbTx): Either[Fail, User]
```

Model different states of an entity as separate types rather than using
`Option` fields and hoping callers check:

```scala
// Callers must remember to check confirmedAt:
case class Order(id: Id[Order], items: List[Item], confirmedAt: Option[Instant])

// The type tells you what state the order is in:
case class PendingOrder(id: Id[Order], items: List[Item])
case class ConfirmedOrder(id: Id[Order], items: List[Item], confirmedAt: Instant)
```

> **Important:** Make illegal states unrepresentable. If a combination of field
> values should never occur, restructure the types so it *can't* occur — don't
> rely on validation to catch it at runtime.

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
