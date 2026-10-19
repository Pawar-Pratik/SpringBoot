# Lesson 37 — Messaging with Kafka

> **Why this lesson exists:** Asynchronous messaging is how services communicate without being coupled to each other's availability. Kafka is the default choice in the JVM world, and it's asked about constantly — usually with follow-ups about delivery guarantees, ordering, and duplicate handling that separate people who've *used* it from people who've *read about* it. This lesson is the depth.

**Time:** ~80 minutes · **Prereq:** Lessons 32, 34

---

## 1. The idea in one sentence

> **Kafka is not a queue, it's a replicated append-only log — messages are retained rather than consumed, ordering exists only within a partition, and every consumer group reads everything; almost every Kafka question is a consequence of those three facts.**

The second half of the lesson follows from one more: you cannot make a database transaction and a publish atomic. That is why the outbox exists, why delivery is at-least-once, and why every consumer must be idempotent.

---

## 2. Why messaging at all

```
SYNCHRONOUS                              ASYNCHRONOUS
api ──HTTP──▶ notifier                   api ──▶ [topic] ──▶ notifier
· notifier down → api request FAILS      · notifier down → messages WAIT, api unaffected
· notifier slow → api slow               · notifier slow → the queue absorbs it
· api must know notifier's address       · api knows only the topic name
· adding a consumer = changing api       · adding a consumer = deploying a new service
· 1 caller → 1 callee                    · 1 producer → N independent consumers
```

**The four genuine benefits:**
1. **Temporal decoupling** — the consumer doesn't have to be up when you produce
2. **Load levelling** — a burst is absorbed by the queue instead of overwhelming a downstream
3. **Fan-out** — one event, many independent consumers, none of which the producer knows about
4. **Replay** — with Kafka's retained log, a new consumer can process history

**And the honest costs, which you must be able to state:**
1. **Eventual consistency** — the effect happens *later*, so your read-after-write assumptions break
2. **Debugging is harder** — no stack trace crosses the broker; you need correlation IDs and tracing
3. **Duplicate delivery** — at-least-once is the practical guarantee, so consumers **must** be idempotent
4. **Ordering is limited** — only within a partition
5. **Operational burden** — a broker is another stateful system to run, monitor, and upgrade

**The senior framing:** *"Messaging trades synchronous simplicity for availability and decoupling. It's the right trade when the operation doesn't need to complete before you respond — notifications, analytics, cross-service propagation. It's the wrong trade when the caller needs the result, because you've added latency and complexity to get an answer you needed anyway."*

**Vigil's use is genuinely justified**, and being able to explain *why* matters more than the code:
- `check.results` — the checker produces at high volume; the API, aggregator, and incident engine all consume independently
- `incident.events` — the incident engine produces; notifier, audit, and analytics consume
- `notification.requests` — buffers delivery so a Slack outage doesn't block incident processing

---

## 3. Kafka's model

```
TOPIC "check.results" (retention: 7 days)
├─ Partition 0: [msg0][msg1][msg2][msg3]...          ← an append-only, ORDERED log
├─ Partition 1: [msg0][msg1][msg2]...
└─ Partition 2: [msg0][msg1][msg2][msg3][msg4]...
                  ▲                    ▲
            consumer group A     consumer group B
            offset = 1           offset = 4
            (independent progress through the same data)
```

**Kafka is not a queue — it's a distributed, replicated, append-only log.** That single distinction explains most of its behaviour:

| Traditional queue (SQS, classic RabbitMQ) | Kafka |
|---|---|
| A message is **consumed and removed** | Messages are **retained** (by time or size); consumers track an offset |
| One consumer gets each message | **Every consumer group** reads every message |
| Can't replay | **Replay by resetting the offset** ✅ |
| Ordering is usually not guaranteed | **Ordered within a partition** |

### The concepts that actually matter

**Partition** — the unit of parallelism and of ordering. Messages within one partition are strictly ordered; across partitions there is **no ordering at all**.

**Key → partition.** `partition = hash(key) % partitionCount`. So **all messages with the same key land in the same partition and are therefore ordered relative to each other.**

```java
// ★ THE most important design decision in Kafka: choose your key deliberately.
kafkaTemplate.send("check.results", monitorId.toString(), result);
// All results for one monitor → one partition → processed in order.
// Different monitors → different partitions → processed in parallel.
```

**That's the pattern:** key by the entity whose ordering matters. You get per-entity ordering *and* cross-entity parallelism simultaneously. **Being able to explain that trade — "I key by monitorId because I need ordering per monitor but parallelism across monitors" — is exactly the design reasoning interviews look for.**

**Consumer group** — consumers sharing a group ID split the partitions between them.

```
Topic with 3 partitions, consumer group "notifier":
  1 consumer  → handles all 3 partitions
  3 consumers → 1 partition each  ✅ maximum parallelism
  5 consumers → 3 work, 2 IDLE    ⚠️ partitions cap your parallelism
```

**Critical consequence: partition count is your maximum consumer parallelism.** 3 partitions means never more than 3 useful consumers. **And you can increase partitions but not decrease them** — and increasing changes the key→partition mapping, so **existing keys move to different partitions and ordering guarantees break across the change.** Over-provision partitions modestly at design time (e.g. 12 for a topic you expect to scale); it's much easier than repartitioning later. **This is real operational knowledge and a great answer.**

**Offset** — a consumer group's position per partition, stored in Kafka's internal `__consumer_offsets` topic.

**Replication** — each partition has a leader and N-1 followers. `replication.factor=3` with `min.insync.replicas=2` survives one broker loss without data loss.

> **Fun fact:** Kafka was built at LinkedIn (2010) by Jay Kreps, Neha Narkhede and Jun Rao to replace a mess of point-to-point pipelines. It's named after Franz Kafka because Kreps liked his writing and thought "a system optimized for writing" deserved a writer's name. The design insight — that a *log* is a better primitive than a queue — is what made replay, stream processing, and event sourcing practical at scale.

---

## 4. Producing

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                       # ★ wait for all in-sync replicas
      retries: 10
      properties:
        enable.idempotence: true      # ★ prevents duplicates from producer retries
        max.in.flight.requests.per.connection: 5
        compression.type: snappy      # ★ big win — JSON compresses well
        linger.ms: 10                 # ★ batch for 10ms → far better throughput
        batch.size: 32768
```

```java
@Service
@RequiredArgsConstructor
public class CheckResultProducer {

    private final KafkaTemplate<String, CheckResultEvent> kafkaTemplate;

    public void publish(CheckResultEvent event) {
        kafkaTemplate.send("check.results", event.monitorId().toString(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("failed to publish for monitor {}", event.monitorId(), ex);
                        deadLetterRepository.save(...);        // ★ don't silently lose it
                    } else {
                        log.debug("published to {}-{} offset {}",
                                result.getRecordMetadata().topic(),
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
```

### `acks` — the durability dial

| `acks` | Waits for | Risk |
|---|---|---|
| `0` | Nothing (fire and forget) | **Silent data loss.** Never for anything that matters. |
| `1` | The leader only | Loss if the leader dies before followers replicate |
| **`all`** | **All in-sync replicas** | ✅ **No loss** with `min.insync.replicas ≥ 2`. Slightly higher latency. |

**Use `acks=all`.** `acks=1` is the "it's probably fine" setting that loses messages during a broker failure — precisely when you least want to.

### `enable.idempotence=true` — set this

Without it, a producer retry after a network timeout can write the message **twice** (the first write succeeded; the ack was lost). With it, the producer attaches a sequence number and the broker deduplicates. **This gives you exactly-once *producing*** — which is a different and much easier problem than exactly-once *processing* (§6).

**Modern Kafka clients default `enable.idempotence` to true**, but set it explicitly: it documents intent and protects you if config elsewhere disables it.

### `linger.ms` — the counter-intuitive one

Setting `linger.ms=10` makes the producer wait up to 10ms to batch messages. **It adds 10ms of latency and can multiply throughput several times over**, because one network round trip carries many messages instead of one. **The default of 0 optimizes latency at a real throughput cost** — for a high-volume topic like `check.results`, lingering is clearly right. Knowing this trade-off is a nice detail.

### Async vs sync send

```java
kafkaTemplate.send(...);                                    // async — fast, fire and forget
kafkaTemplate.send(...).get(5, TimeUnit.SECONDS);           // ⚠️ blocking — kills throughput
```
**Never block on send in a request path.** Use the callback. If you need "the message is definitely published before I respond," you don't want Kafka in that path — you want the **outbox pattern** (§4).

---

## 5. Kafka and transactions — the dual-write problem

```java
// ❌ THE CLASSIC BUG
@Transactional
public void recordFailure(Monitor monitor, CheckOutcome outcome) {
    monitor.setStatus(DOWN);                             // database write
    kafkaTemplate.send("check.results", event);           // ⚠️ Kafka write
}
// The database transaction can roll back AFTER Kafka has accepted the message.
// → An event describing a state change that never happened.
// Or Kafka fails after the DB commits → the change happened, nobody was told.
```

**You cannot make a database transaction and a Kafka publish atomic.** Two different systems, no shared commit protocol (XA exists and is broadly a bad idea with modern brokers). **This is the dual-write problem, and it has exactly two real solutions.**

**Solution 1 — `AFTER_COMMIT` (good enough for most cases):**
```java
@Transactional
public void recordFailure(Monitor monitor, CheckOutcome outcome) {
    monitor.setStatus(DOWN);
    events.publishEvent(new MonitorWentDownEvent(monitor.getId()));
}

@TransactionalEventListener(phase = AFTER_COMMIT)
void publish(MonitorWentDownEvent e) {
    kafkaTemplate.send("incident.events", e.monitorId().toString(), e);
}
```
**Never publishes for rolled-back work.** Remaining gap: the process can die between commit and publish, losing the message with no record it was owed.

**Solution 2 — the transactional outbox (correct):**

```java
@Transactional
public void recordFailure(Monitor monitor, CheckOutcome outcome) {
    monitor.setStatus(DOWN);
    outboxRepository.save(new OutboxMessage("incident.events",
            monitor.getId().toString(), toJson(event)));      // ★ SAME transaction
}
```
```java
@Component
public class OutboxRelay {
    @Scheduled(fixedDelay = 500)
    void relay() {
        for (var msg : outboxRepository.claimUnpublished(200)) {   // FOR UPDATE SKIP LOCKED
            kafkaTemplate.send(msg.topic(), msg.key(), msg.payload())
                    .whenComplete((r, ex) -> {
                        if (ex == null) outboxRepository.markPublished(msg.id());
                    });
        }
    }
}
```

**Why this is correct:** "the state changed" and "the message is owed" are the same database transaction, so they're atomic. If the relay crashes, unsent rows are still there and get retried. **The consequence: at-least-once delivery, so consumers must be idempotent** (§6).

**Production alternative worth naming: Change Data Capture with Debezium** — it tails the database's write-ahead log and publishes changes to Kafka with no polling and no application code. **It's the outbox pattern implemented by reading the transaction log**, which means it can't miss anything. Mentioning Debezium in a dual-write discussion is a strong signal.

### Kafka's own transactions

```java
@Transactional("kafkaTransactionManager")
public void processAndForward(ConsumerRecord<String, Event> record) {
    kafkaTemplate.send("output-topic", transform(record.value()));
    // The consumed offset and the produced message commit together
}
```
**This gives exactly-once semantics for Kafka→Kafka processing only.** It does **not** extend to your database. **That precision matters:** people claim "Kafka supports exactly-once" and then assume it covers their database writes. It doesn't — for that you need an idempotent consumer.

---

## 6. Consuming

```yaml
spring:
  kafka:
    consumer:
      group-id: vigil-notifier
      auto-offset-reset: earliest        # ★ new group starts at the beginning (vs 'latest')
      enable-auto-commit: false          # ★★ CRITICAL — see below
      max-poll-records: 100
      properties:
        spring.json.trusted.packages: "com.vigil.events"   # ★ deserialization safety
        max.poll.interval.ms: 300000
        isolation.level: read_committed
    listener:
      ack-mode: MANUAL_IMMEDIATE         # or RECORD / BATCH
      concurrency: 3                     # ★ ≤ partition count
```

```java
@Component
@RequiredArgsConstructor
public class NotificationConsumer {

    @KafkaListener(topics = "incident.events", groupId = "vigil-notifier")
    public void consume(@Payload IncidentEvent event,
                        @Header(KafkaHeaders.RECEIVED_KEY) String key,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset,
                        Acknowledgment ack) {
        try {
            MDC.put("correlationId", event.correlationId());     // ★ tracing across the broker
            notificationService.deliver(event);
            ack.acknowledge();                                    // ★ commit only after success
        } catch (TransientException e) {
            throw e;                                              // let the error handler retry
        } catch (Exception e) {
            log.error("permanent failure at {}-{}", partition, offset, e);
            deadLetterProducer.send(event, e);
            ack.acknowledge();                                    // ★ don't block the partition
        } finally {
            MDC.clear();
        }
    }
}
```

### `enable-auto-commit=false` — the most important consumer setting

**With auto-commit (the default), Kafka commits offsets on a timer, regardless of whether you processed successfully.**

```
auto-commit interval = 5s
t=0    poll 100 records
t=5    ★ offsets COMMITTED automatically
t=6    record 50 throws — the process crashes
t=10   restart → resumes AFTER record 100
       → records 50–100 are NEVER processed, and nobody knows. Silent data loss.
```

**Disable auto-commit and acknowledge explicitly after successful processing.** This is the difference between at-least-once (correct) and "somewhere between at-most-once and at-least-once, non-deterministically" (broken).

### The ordering trap: one bad message blocks a partition

```
Partition 0: [msg1][msg2][MSG3-POISON][msg4][msg5]...
                            ▲
             Retrying forever → msg4, msg5, and everything after
             are NEVER processed. The partition is stuck.
```

**Kafka's ordering guarantee means you cannot skip ahead.** So a message that always fails — a poison pill — halts that partition indefinitely. **This is the #1 Kafka production incident**, and the answer is a dead-letter topic:

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    var backOff = new ExponentialBackOffWithMaxRetries(3);
    backOff.setInitialInterval(1000L);
    backOff.setMultiplier(2.0);

    var handler = new DefaultErrorHandler(recoverer, backOff);
    handler.addNotRetryableExceptions(                          // ★ fail fast on permanent errors
            SerializationException.class,
            ValidationException.class,
            IllegalArgumentException.class);
    return handler;
}
```

**Retry a bounded number of times, then move it to `<topic>.DLT` and advance.** The DLT preserves the message (with exception headers) for investigation and replay. **The partition keeps moving.**

**Classify your exceptions.** A `SerializationException` will never succeed on retry — retrying it three times just delays the inevitable and burns capacity. **`addNotRetryableExceptions` is the same principle as `noRetryFor` on `@Retryable`** (Lesson 34): don't retry what can't succeed.

**And build a DLT replay tool.** A dead-letter topic nobody consumes is a data-loss log with extra steps. **You need a way to fix the bug and reprocess** — that's the whole point. Mentioning replay tooling unprompted is a good sign; most people stop at "send it to a DLT."

### `concurrency` and rebalancing

```yaml
spring.kafka.listener.concurrency: 3      # 3 consumer threads in this instance
```
**Total effective consumers = instances × concurrency, capped by partition count.** 3 pods × 3 concurrency = 9 consumers, but 6 partitions means 3 sit idle.

**Rebalancing** happens when consumers join or leave. During a rebalance, **consumption stops** ("stop-the-world"), which means a rolling deploy causes a consumption pause. Modern Kafka's **cooperative sticky assignor** (`partition.assignment.strategy=CooperativeStickyAssignor`) makes rebalances incremental so only affected partitions pause — **worth enabling, and worth knowing about.**

**`max.poll.interval.ms` is the trap here:** if processing a batch takes longer than this (default 5 min), Kafka assumes the consumer is dead and rebalances — **while you're still working.** Your work completes but the offset commit fails, so the records are reprocessed by another consumer. Fix by reducing `max-poll-records`, increasing the interval, or moving slow work off the poll thread. **This is a genuinely confusing production symptom ("my messages are processed twice under load") with a precise cause.**

---

## 7. Delivery semantics — be precise here

| Guarantee | How | Trade-off |
|---|---|---|
| **At-most-once** | Commit the offset **before** processing | Fast; **loses messages** on failure |
| **At-least-once** | Commit **after** processing | ✅ **The practical default.** Duplicates possible. |
| **Exactly-once** | Kafka transactions (Kafka→Kafka), or **at-least-once + idempotent consumer** | Complexity, or careful design |

**The crucial nuance, and interviewers listen for it:**

> *"Exactly-once delivery is impossible in a distributed system — you can't distinguish a lost acknowledgement from a lost message. What's achievable is exactly-once **processing**, and the practical route is at-least-once delivery plus an idempotent consumer. Kafka's transactions give exactly-once for Kafka-to-Kafka pipelines, but they don't extend to your database, so once external state is involved you're back to idempotency."*

**That answer demonstrates you understand the theory and the practice.** Most candidates either say "Kafka supports exactly-once" (imprecise) or don't know the distinction exists.

### The idempotent consumer

```java
@Component
@RequiredArgsConstructor
public class CheckResultConsumer {

    @KafkaListener(topics = "check.results")
    @Transactional                                     // ★ dedupe + side effect, atomically
    public void consume(CheckResultEvent event, Acknowledgment ack) {
        if (processedEventRepository.existsByEventId(event.eventId())) {
            log.debug("duplicate event {}, skipping", event.eventId());
            ack.acknowledge();
            return;
        }
        checkResultService.record(event);
        processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
        ack.acknowledge();
    }
}
```

**The dedupe record and the business change must be in the same transaction.** Otherwise you can apply the change, crash before recording it, and apply it again on redelivery. **Same insight as the outbox, viewed from the consuming side** — and noticing that symmetry is the mark of having understood both.

**Better still, make the operation naturally idempotent** (Lesson 34 §5):
```sql
INSERT INTO check_result (id, monitor_id, checked_at, status)
VALUES (?, ?, ?, ?)
ON CONFLICT (monitor_id, checked_at) DO NOTHING;      -- ★ the database enforces it
```
**No dedupe table, no extra query, impossible to forget.** Vigil uses exactly this — and being able to say "I designed the schema so duplicate delivery is a no-op" is stronger than describing a dedupe table.

**Prune the dedupe table.** An unbounded `processed_event` table grows forever; keep a retention window comfortably longer than your maximum redelivery delay.

---

## 8. Schema evolution

**Producers and consumers deploy independently, so an old consumer will receive new messages and vice versa.**

```java
// v1
record CheckResultEvent(UUID monitorId, String status, long durationMs) {}

// v2 — ✅ COMPATIBLE: added an OPTIONAL field with a default
record CheckResultEvent(UUID monitorId, String status, long durationMs, String region) {}

// v3 — ❌ BREAKING: renamed a field. Old consumers see null.
record CheckResultEvent(UUID monitorId, String state, long durationMs) {}
```

**The compatibility rules:**

| Change | Safe? |
|---|---|
| Add an optional field with a default | ✅ |
| Remove an optional field | ⚠️ Only once no consumer reads it |
| Add a required field | ❌ Old producers can't supply it |
| Rename a field | ❌ Always breaking — add new, migrate, remove old |
| Change a field's type | ❌ |
| Add an enum value | ⚠️ Old consumers must tolerate unknown values |

**That last row is a subtle real trap:** adding `DEGRADED` to a status enum breaks a consumer whose `switch` has no default case, or whose deserializer rejects unknown values. **Consumers should always tolerate unknown enum values** (map to an `UNKNOWN` constant), which is the mirror image of Lesson 03's exhaustive-switch advice — inside your own service exhaustiveness is a feature; across a message boundary it's a liability.

**Options for enforcement:**

| Approach | Notes |
|---|---|
| **JSON + discipline** | Simple, human-readable, **no enforcement**. Fine for a small team, risky at scale. |
| **Avro + Schema Registry** | ✅ Compact binary, **compatibility checked at publish time**, schema stored centrally |
| **Protobuf + Schema Registry** | Similar, better cross-language tooling |
| **CloudEvents** | A standard envelope (id, source, type, time) — good for metadata, orthogonal to payload format |

**Confluent Schema Registry with `BACKWARD` compatibility mode is the industry answer:** it *rejects* an incompatible schema at publish time, so a breaking change fails your deploy instead of your consumers. **"I'd use a schema registry with backward compatibility enforcement, because otherwise schema evolution depends on everyone remembering the rules" is a strong answer.**

**And the security note:** always set `spring.json.trusted.packages`. Without it, `JsonDeserializer` can be induced to instantiate arbitrary classes named in the message — the same deserialization-gadget class of vulnerability as Jackson polymorphic typing (Lesson 15).

---

## 9. Kafka vs RabbitMQ vs SQS — have an opinion

| | **Kafka** | **RabbitMQ** | **SQS** |
|---|---|---|---|
| Model | Distributed log | Broker with exchanges/queues | Managed queue |
| Retention / replay | ✅ Days–forever, replayable | ❌ Consumed = gone | ❌ (14 days max, not replayable) |
| Ordering | Per partition | Per queue (with one consumer) | FIFO queues only |
| Throughput | **Very high** (millions/s) | High (tens of thousands/s) | High, managed |
| Routing flexibility | Basic (topic + key) | ✅ **Excellent** (topic/header exchanges) | Basic |
| Per-message TTL, priority, delay | ❌ | ✅ | Partial |
| Operational burden | **High** (or use Confluent/MSK) | Medium | ✅ **None** |
| Consumer scaling | Capped by partitions | Add consumers freely | Automatic |

**When each is right:**
- **Kafka** — high volume, event streaming, replay, multiple independent consumers, event sourcing. **Vigil's check results fit perfectly** (high volume, several consumers, replay useful for recomputing aggregates).
- **RabbitMQ** — complex routing, per-message priority/TTL/delay, work queues with unbounded consumer scaling, lower volume. **Often the better choice for classic task queues**, and simpler to operate.
- **SQS/SNS** — you're on AWS and want zero operational work. Boring, and boring is frequently correct.

**The answer that shows judgment:** *"Kafka for high-volume event streams where I want replay and multiple consumers; RabbitMQ for task queues needing rich routing, priorities, or delays; SQS when I'm on AWS and the operational simplicity is worth the feature limits. Kafka is the default in the JVM ecosystem, but 'we need Kafka' is often 'we need a queue' — and running Kafka yourself is a real, ongoing cost."*

**That last clause matters.** Kafka is genuinely operationally demanding, and choosing it for a 100-messages-per-second workload is over-engineering. Saying so is a maturity signal.

> **Worth knowing:** Kafka removed its ZooKeeper dependency in favour of **KRaft** (self-managed metadata quorum), GA since Kafka 3.3 and the default in 4.0. If someone mentions ZooKeeper as a Kafka requirement, that's dated knowledge — a good thing to be right about.

---

## 10. Build & break — Vigil's event pipeline

**1. Kafka locally with KRaft (no ZooKeeper):**
```yaml
# docker-compose.kafka.yml
services:
  kafka:
    image: apache/kafka:3.8.0
    ports: [ "9092:9092" ]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  kafka-ui:
    image: provectuslabs/kafka-ui
    ports: [ "8090:8080" ]
    environment: { KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092 }
```
**Open Kafka UI at :8090** — being able to *see* topics, partitions, offsets and consumer lag makes everything below concrete.

**2. Topics with deliberate partition counts:**
```
check.results        12 partitions   (high volume, keyed by monitorId)
incident.events       3 partitions
notification.requests 3 partitions
+ a .DLT for each
```

**3. Producer** with `acks=all`, idempotence, snappy compression, `linger.ms=10`, keyed by `monitorId`, with a completion callback.

**4. ⭐ Prove key-based ordering.** Publish 100 events for the *same* monitorId and 100 across 100 different monitorIds. In Kafka UI, confirm the first set all landed in **one** partition and the second spread across all 12. **Then consume with `concurrency: 3` and log partition + offset — verify per-key order is preserved while different keys process in parallel.** This is the experiment that makes partitioning click.

**5. Consumer** with `enable-auto-commit=false`, `MANUAL_IMMEDIATE` ack, MDC correlation from the message header, and explicit acknowledgment after success.

**6. ⭐ Reproduce the auto-commit data loss.** Set `enable-auto-commit=true` with a 1-second interval. Consume 100 records where record 50 throws and then kill the process. **Restart and confirm records 50–100 were never processed — and that nothing logged an error.** Then set it to `false` with manual ack and confirm redelivery from 50. **That contrast is the most important consumer lesson there is.**

**7. ⭐ Reproduce the poison pill.** Publish a message that always throws. **Watch the partition stall** — publish 10 more good messages after it and confirm none are processed, indefinitely. Then add the `DefaultErrorHandler` with a DLT and confirm: 3 retries, then the poison goes to `.DLT`, then the good messages flow. **Check the DLT message's exception headers in Kafka UI.**

**8. Non-retryable classification.** Publish a malformed message causing `SerializationException`. With `addNotRetryableExceptions`, confirm it goes **straight** to the DLT with no retries. Without it, count 3 pointless retries.

**9. Idempotent consumer, two ways.** First with a `processed_event` dedupe table in the same transaction. Then replace it with `ON CONFLICT DO NOTHING` on `(monitor_id, checked_at)`. **Publish the same event 5 times and confirm exactly one row for both approaches.** Note which one you'd prefer, and why.

**10. Outbox → Kafka.** Wire the Lesson 32 outbox relay to Kafka. **Then `kill -9` the app between the transaction commit and the relay tick, restart, and confirm the message is published.** You've demonstrated at-least-once delivery across a crash — a genuinely impressive thing to have working.

**11. Consumer lag, observed.** Stop the consumer, publish 10,000 messages, and watch lag climb in Kafka UI and in `kafka_consumer_fetch_manager_records_lag_max`. Restart and watch it drain. **Then set up an alert on lag** (Lesson 35) — consumer lag is *the* health metric for a Kafka consumer, because a stalled consumer looks perfectly healthy from every other angle.

**12. Schema evolution, both ways.** Add an optional field with a default → confirm the old consumer still works. Then **rename** a field → confirm the old consumer silently receives `null`. **Sit with the fact that this produced no error anywhere.** Then add a new enum value and confirm a consumer with a non-exhaustive `switch` mishandles it.

**13. Rebalance and `max.poll.interval.ms`.** Set `max.poll.interval.ms=10000` and make processing take 15 seconds. **Watch the consumer get evicted mid-work, rebalance, and reprocess the batch.** That's the "my messages process twice under load" mystery, manufactured.

**14. Test it.** Testcontainers with `KafkaContainer`, plus `@EmbeddedKafka` for lighter tests. Assert: a produced message lands on the right topic and partition, a consumer processes it, a poison message reaches the DLT, and a duplicate produces one row.

### Explain out loud (60s)

1. Why "Kafka is a log, not a queue" explains retention, replay and consumer groups.
2. How keying gives you ordering and parallelism at the same time.
3. The dual-write problem, and the two real solutions.
4. The poison-pill incident: what happens, and the full fix including replay.
5. Exactly-once, stated precisely, including what Kafka transactions don't cover.

---

## 11. Production rules

| Do / Never | Why |
|---|---|
| **Key messages by the entity whose ordering matters** | Same key means same partition means ordered — and you still get parallelism across keys, which is the whole design trick |
| **Over-provision partitions modestly at design time** | Partition count caps consumer parallelism, you can't reduce it, and increasing it remaps keys and breaks ordering across the change |
| **Use `acks=all` with `min.insync.replicas ≥ 2`** | `acks=1` loses messages exactly when a broker fails, which is precisely when you care |
| **Set `enable.idempotence=true` explicitly** | It's the modern default, but stating it documents intent and protects you when configuration elsewhere turns it off |
| **Never block on a send in a request path** | Use the callback; if you truly need "published before I respond", you need the outbox, not a synchronous send |
| **Use an outbox (or Debezium CDC) when a message must not be lost** | Writing the row and the message in one transaction is the only way to make "it happened" and "it will be sent" atomic |
| **Set `enable-auto-commit=false` and acknowledge after processing** | Auto-commit advances the offset on a timer regardless of success, which gives you non-deterministic message loss |
| **Give every consumer a dead-letter topic** | Kafka can't skip ahead, so one poison pill halts its partition indefinitely — the most common Kafka production incident |
| **Classify exceptions and don't retry what can't succeed** | A `SerializationException` will fail identically three times; retrying it just delays the DLT and burns capacity |
| **Build a DLT replay tool** | A dead-letter topic nobody consumes is a data-loss log with extra steps — the point is to fix the bug and reprocess |
| **Enable the cooperative sticky assignor** | Rebalances then pause only the affected partitions instead of stopping all consumption during every rolling deploy |
| **Watch `max.poll.interval.ms` against your batch processing time** | Exceed it and Kafka rebalances while you're still working, so the commit fails and another consumer redoes the batch |
| **Say "exactly-once *processing*", never "exactly-once delivery"** | Delivery can't be exactly-once in a distributed system; at-least-once plus an idempotent consumer is the achievable thing |
| **Write the dedupe record in the same transaction as the business change** | Otherwise you apply the change, crash before recording it, and apply it again — the outbox insight from the consumer's side |
| **Prefer natural idempotency to a dedupe table** | A unique constraint with `ON CONFLICT DO NOTHING` can't be forgotten and costs no extra query |

---

## 12. Interview traps

**Q1. What is Kafka and how does it differ from a queue?**
A distributed, replicated, append-only log. Messages are **retained** rather than removed on consumption, and each consumer group tracks its own offset — so many groups read the same data independently and any group can **replay** by resetting its offset. Ordering is guaranteed within a partition only.

**Q2. What determines the partition, and why does it matter?**
`hash(key) % partitions`. All messages with the same key go to the same partition and are therefore ordered relative to each other. **Choosing the key is the most important Kafka design decision:** keying by `monitorId` gives per-monitor ordering with cross-monitor parallelism.

**Q3. How does consumer parallelism work?**
Consumers in a group split partitions; **partition count caps useful parallelism** — extra consumers idle. You can add partitions but not remove them, and adding them **changes the key→partition mapping**, breaking ordering across the change. So over-provision modestly at design time.

**Q4. Explain the delivery guarantees.**
At-most-once (commit before processing — loses messages), at-least-once (commit after — duplicates possible, the practical default), exactly-once. **Exactly-once *delivery* is impossible in a distributed system** — you can't distinguish a lost ack from a lost message. Exactly-once *processing* is achievable: Kafka transactions for Kafka→Kafka, or at-least-once plus an idempotent consumer once external state is involved.

**Q5. Why must you disable auto-commit?**
Auto-commit commits offsets on a timer regardless of processing success, so a crash after a commit but before processing **silently skips records** with no error. Disable it and acknowledge explicitly after success — that's what makes at-least-once actually hold.

**Q6. What's a poison pill and how do you handle it?**
A message that always fails. Because Kafka preserves ordering, you can't skip it, so the partition stalls indefinitely and every subsequent message is blocked. Handle it with `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`: bounded retries with backoff, then publish to `<topic>.DLT` and advance. Classify permanent exceptions as non-retryable, and **build a replay tool** — an unmonitored DLT is just a data-loss log.

**Q7. How do you publish a message and update a database atomically?**
You can't — that's the dual-write problem. Use `@TransactionalEventListener(AFTER_COMMIT)` (never publishes for rolled-back work, but can lose the message if the process dies after commit), or the **transactional outbox** (write the message to a table in the same transaction; a relay publishes it), or **Debezium CDC** reading the write-ahead log. The outbox gives at-least-once, so consumers must be idempotent.

**Q8. What is `acks=all` and why does it matter?**
The producer waits for all in-sync replicas to acknowledge. With `min.insync.replicas ≥ 2` you don't lose data when a broker dies. `acks=1` loses messages if the leader fails before replication; `acks=0` loses them silently.

**Q9. What does `enable.idempotence` do?**
It attaches a producer id and sequence number so the broker deduplicates retried writes — preventing duplicates when a send succeeds but the ack is lost. It gives exactly-once *producing*, which is distinct from exactly-once *processing*.

**Q10. What's `linger.ms` and why would you increase it?**
It's how long the producer waits to batch records. Raising it to ~10ms adds a little latency and can multiply throughput, because one round trip carries many messages. The default (0) optimizes latency at a throughput cost.

**Q11. What is a rebalance and what problems does it cause?**
Reassignment of partitions when consumers join or leave. Consumption pauses during it, so rolling deploys cause a stall — mitigate with the `CooperativeStickyAssignor` for incremental rebalancing. Also watch `max.poll.interval.ms`: if processing exceeds it, the consumer is presumed dead and evicted **mid-work**, so its records get reprocessed elsewhere — the classic "duplicate processing under load."

**Q12. How do you evolve a message schema safely?**
Only additive, optional changes with defaults; never rename or retype. Consumers must tolerate unknown fields and unknown enum values. Enforce it with a schema registry (Avro/Protobuf) in `BACKWARD` compatibility mode so an incompatible schema is rejected at publish time rather than breaking consumers. And set `spring.json.trusted.packages` to prevent deserialization attacks.

**Q13. Kafka or RabbitMQ?**
Kafka for high-volume event streams needing replay and multiple independent consumers. RabbitMQ for task queues needing rich routing, priorities, delays, or unbounded consumer scaling — and it's simpler to operate. SQS when you're on AWS and want zero operational work. "We need Kafka" is often "we need a queue," and Kafka's operational cost is real.

**Q14. What's the single most important metric for a Kafka consumer?**
**Consumer lag.** A stalled or slow consumer looks perfectly healthy by every other measure — the JVM is fine, no errors, health checks pass — while messages pile up unprocessed. Alert on lag and on its rate of growth.

---

## 13. Retention check

1. Why is Kafka a log rather than a queue, and name two capabilities that follow from it.
2. What happens to ordering if you increase a topic's partition count?
3. Explain the auto-commit data-loss scenario step by step.
4. Why does a poison pill stall an entire partition, and what's the fix?
5. State precisely what Kafka's exactly-once does and does not cover.
6. Give the compatibility rules for schema evolution, including the enum trap.

---

**Next:** [Lesson 38 — Microservices fundamentals](38-microservices-fundamentals.md) — when to split, where the boundaries go, and how to handle data that no longer fits in one transaction.
