# Streaming Data from Kafka

## Dependencies

- `"com.softwaremill.ox" %% "kafka"` — Kafka integration for Ox Flows

---

## Ox Flows

A `Flow[T]` is a lazy, asynchronous data pipeline. It describes transformations
that only execute when a terminal operation (`runToList`, `runForeach`,
`runDrain`) is called. Flows are built on top of Ox's channels and structured
concurrency.

```scala
Flow.fromValues(1, 2, 3)
  .map(_ * 2)
  .filter(_ > 2)
  .runToList()  // List(4, 6)
```

Flows support concurrent transformations:

```scala
Flow.fromValues(1, 2, 3, 4, 5)
  .mapPar(3)(expensiveOperation)  // up to 3 concurrent operations
  .runForeach(println)
```

## Consuming from Kafka

`KafkaFlow.subscribe` creates a `Flow[ReceivedMessage[K, V]]` from a Kafka
topic:

```scala
import ox.kafka.{ConsumerSettings, KafkaFlow, ReceivedMessage}
import ox.kafka.ConsumerSettings.AutoOffsetReset

val settings = ConsumerSettings.default("my_group")
  .bootstrapServers("localhost:9092")
  .autoOffsetReset(AutoOffsetReset.Earliest)

KafkaFlow
  .subscribe(settings, "my_topic")
  .map(msg => process(msg.value))
  .runDrain()
```

`runDrain()` consumes the flow indefinitely, discarding results. This is the
typical pattern for a consumer that processes messages for side effects.

## Publishing to Kafka

Messages can be published as a terminal operation (drain) or as an intermediate
step that returns metadata:

```scala
// As a drain — fire and forget
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .pipe(KafkaDrain.runPublish(producerSettings))

// As an intermediate step — returns RecordMetadata per message
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .mapPublish(producerSettings)
  .runForeach(metadata => println(s"Sent to partition ${metadata.partition()}"))
```

## Consuming, processing, and committing offsets

A common pattern: consume messages, process them (possibly in parallel), and
commit offsets:

```scala
KafkaFlow
  .subscribe(consumerSettings, "source_topic")
  .mapPar(10) { msg =>
    process(msg.value)
    CommitPacket(msg)
  }
  .pipe(KafkaDrain.runCommit(consumerSettings))
```

`CommitPacket` wraps the received message so that the commit drain knows which
offsets to commit. Offsets are committed in batches (by default, every second),
computing the maximum offset per topic-partition.

## Transactional produce-and-commit

For exactly-once semantics when reading from one topic and writing to another:

```scala
KafkaFlow
  .subscribe(consumerSettings, "source_topic")
  .map { msg =>
    val output = ProducerRecord[String, String]("target_topic", transform(msg.value))
    SendPacket(send = List(output), commit = List(msg))
  }
  .pipe(KafkaDrain.runPublishAndCommit(producerSettings, consumerSettings))
```

`SendPacket` combines records to publish with messages to commit. The drain uses
Kafka transactions to atomically publish the output records and commit the input
offsets.

## Integration with structured concurrency

Kafka consumers run within Ox scopes. When the scope ends (e.g., application
shutdown), the consumer is closed gracefully — pending offsets are committed and
the consumer group rebalances.

```scala
object Main extends OxApp.Simple:
  override def run(using Ox): Unit =
    KafkaFlow
      .subscribe(settings, "my_topic")
      .mapPar(5)(process)
      .runDrain()
```

Because `runDrain()` blocks indefinitely (the Kafka consumer keeps polling),
there's no need for `never` — the flow itself keeps the scope alive. On SIGTERM,
`OxApp` interrupts the scope, which interrupts the flow, which closes the Kafka
consumer.
