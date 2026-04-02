# Kafka Streaming

## Dependencies

- `"com.softwaremill.ox" %% "kafka"` — Kafka integration for Ox Flows

---

> **Important:** The ox-kafka abstractions (`KafkaFlow`, `KafkaDrain`,
> `ConsumerSettings`, `ProducerSettings`) replace direct use of the low-level
> `KafkaConsumer` and `KafkaProducer` APIs from `org.apache.kafka.clients`.
> Types such as `ProducerRecord` and `RecordMetadata` are still used with
> ox-kafka, but you should not construct or manage `KafkaConsumer` /
> `KafkaProducer` instances directly — use the Flow-based API shown below instead.

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

## Publishing to Kafka

Messages can be published as a terminal operation (drain) or as an intermediate
step that returns metadata:

```scala
// As a drain — fire and forget
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .pipe(KafkaDrain.runPublish(producerSettings))

// As an intermediate step — returns RecordMetadata per message
import ox.kafka.KafkaStage.*
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .mapPublish(producerSettings)
  .runForeach(metadata => println(s"Sent to partition ${metadata.partition()}"))
```

## Consuming, processing, and committing offsets

```scala
supervised:
  val consumer = consumerSettings.toThreadSafeConsumerWrapper
  KafkaFlow
    .subscribe(consumer, "source_topic")
    .mapPar(10) { msg =>
      process(msg.value)
      CommitPacket(msg)
    }
    .pipe(KafkaDrain.runCommit(consumer))
```

> **Required:** The consumer must be shared between subscribe and commit stages
> via `toThreadSafeConsumerWrapper`. Using separate consumer instances would
> silently fail to commit offsets.

Offsets are committed in batches (by default, every second), computing the
maximum offset per topic-partition.

## Transactional produce-and-commit

For exactly-once semantics when reading from one topic and writing to another:

```scala
supervised:
  val consumer = consumerSettings.toThreadSafeConsumerWrapper
  KafkaFlow
    .subscribe(consumer, "source_topic")
    .map { msg =>
      val output = ProducerRecord[String, String]("target_topic", transform(msg.value))
      SendPacket(output, msg)
    }
    .pipe(KafkaDrain.runPublishAndCommit(producerSettings, consumer))
```

`SendPacket` combines records to publish with messages to commit. The drain uses
Kafka transactions to atomically publish the output records and commit the input
offsets.

`runDrain()` blocks indefinitely (the Kafka consumer keeps polling), so there's
no need for `never` in `OxApp`. On scope termination, consumers are closed
gracefully.
