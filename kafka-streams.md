# Kafka Streams: Zero to Hero

## Table of Contents
0. [Before You Start: Mental Models](#0-before-you-start)
1. [What is Kafka Streams?](#1-what-is-kafka-streams)
2. [How it Compares to Other Approaches](#2-how-it-compares)
3. [Core Concepts](#3-core-concepts)
4. [Setting Up](#4-setting-up)
5. [Your First Kafka Streams App](#5-your-first-kafka-streams-app)
6. [KStream vs KTable vs GlobalKTable](#6-kstream-vs-ktable-vs-globalktable)
7. [Stateless Operations](#7-stateless-operations)
8. [Stateful Operations](#8-stateful-operations)
9. [Joins](#9-joins)
10. [Windowing](#10-windowing)
11. [State Stores](#11-state-stores)
12. [Processor API (Low-Level)](#12-processor-api)
13. [Error Handling](#13-error-handling)
14. [Testing with TopologyTestDriver](#14-testing)
15. [Deployment & Scaling](#15-deployment--scaling)
16. [Production Best Practices](#16-production-best-practices)
17. [Real-World Scenarios](#17-real-world-scenarios)
18. [Common Beginner Mistakes](#18-common-beginner-mistakes)
19. [Learning Path & Next Steps](#19-learning-path)
20. [Glossary of Terms](#20-glossary)

---

## 0. Before You Start: Mental Models

> **If you take only one thing from this guide, take this section.** These mental models are what separate someone who *uses* Kafka Streams from someone who *understands* it.

### The Coffee Shop Analogy

Imagine a coffee shop:

- **Kafka topic** = the order queue stuck on the wall (orders pile up, never get rewritten)
- **Producer** = customers placing orders
- **Consumer** = a barista picking up orders
- **Kafka Streams** = a smart barista who:
    - Reads orders one by one
    - Transforms them ("decaf" → "regular" if it's after noon)
    - Counts how many lattes per hour (stateful!)
    - Pairs each order with the customer's loyalty info (a join!)
    - Hands the finished drinks back to a *new* queue (output topic)

The key insight: **Kafka Streams doesn't store data; it processes events flowing through Kafka and emits new events.**

### The Conveyor Belt Mental Model

Think of streams as **conveyor belts** carrying boxes (events):

```
INPUT BELT  →  [filter]  →  [transform]  →  [count by type]  →  OUTPUT BELT
   📦📦📦                                                          📦📦
```

- Each box is independent (no batching)
- Operations are stations along the belt
- The belt never stops — events flow continuously
- You can split the belt (branching) or merge two belts (joins)

### Stream vs Table: The Most Important Concept

This trips up almost every beginner. Here's the simplest way to think about it:

**A stream is like a bank's transaction log:**
```
2024-01-01: Alice deposited $100
2024-01-02: Alice withdrew $30
2024-01-03: Alice deposited $50
```
Each line is a fact, an event. You never edit history.

**A table is like the current account balance:**
```
Alice: $120  (computed from the transactions above)
```
This is the *current state* — the result of replaying all transactions.

**Stream-table duality**: every table has an underlying stream of changes (a "changelog"), and every stream can be aggregated into a table. They are two views of the same data.

```
Stream → Table: aggregate / reduce / count
Table → Stream: .toStream()
```

### Why "Streams" Need State

Beginners often ask: "If streams are just events flowing through, why do we need RocksDB and state stores?"

**Answer:** Because some questions can only be answered by remembering past events.
- "How many orders has Alice placed today?" — need a counter per user
- "What was the user's last action?" — need to remember the previous event
- "Average order value over 5 minutes?" — need to keep events in a window

State stores are Kafka Streams' way of remembering things, locally on disk (RocksDB), without ever leaving your app.

### The "Internal Topics" Reality Check

When you run a Kafka Streams app, it creates **internal topics in Kafka** that you didn't ask for:

1. **Repartition topics** — created when you change the key (`selectKey`, `groupBy`, `map`). Required because aggregations need same-key records on the same partition.
2. **Changelog topics** — created for every state store. Used to restore state when an instance crashes.

**Why this matters:** Your "simple" Kafka Streams app might create 10+ extra topics in your broker. Don't be surprised. Don't delete them manually.

### How to Read a Topology

A topology is the blueprint of your app. Print it before deploying:

```java
Topology topology = builder.build();
System.out.println(topology.describe());
```

Output looks like:
```
Topologies:
   Sub-topology: 0
    Source: KSTREAM-SOURCE-0000 (topics: [orders])
      → KSTREAM-FILTER-0001
    Processor: KSTREAM-FILTER-0001 (stores: [])
      → KSTREAM-MAPVALUES-0002
      ← KSTREAM-SOURCE-0000
    ...
```

**Reading tip:** Each "Sub-topology" is a unit of parallelism. Boundaries between sub-topologies = a repartition topic. **Fewer sub-topologies = less network I/O.**

---

## 1. What is Kafka Streams?

Kafka Streams is a **client library** (not a separate cluster) for building real-time streaming applications on top of Apache Kafka.

### Key characteristics
- Runs **inside your application** — no separate processing cluster needed
- Reads from Kafka topics, processes, writes back to Kafka topics
- Exactly-once semantics support
- Fault-tolerant with local state backed by changelog topics
- Scales horizontally by adding more instances

### When to use it
| Use Case | Good Fit? |
|---|---|
| Filter / transform events | ✅ |
| Aggregations per key (counts, sums) | ✅ |
| Joining two event streams | ✅ |
| Complex CEP / ML inference | ⚠️ Consider Flink |
| Batch processing of historical data | ❌ Use Spark |

---

## 2. How it Compares

| Feature | Kafka Streams | Kafka Consumer API | Apache Flink |
|---|---|---|---|
| Deployment | Library in your app | Library in your app | Separate cluster |
| State management | Built-in (RocksDB) | Manual | Built-in |
| Exactly-once | Yes | Manual | Yes |
| Learning curve | Medium | Low | High |
| SQL support | Limited (KSQL) | No | Yes (Table API) |

---

## 3. Core Concepts

### Topology
A **directed acyclic graph (DAG)** of processing steps — sources → processors → sinks.

```
[Kafka Topic A] --> [Filter] --> [Map] --> [Aggregate] --> [Kafka Topic B]
```

### Stream vs Table duality
- **KStream** — unbounded sequence of events (INSERT semantics)
- **KTable** — changelog stream representing the latest value per key (UPDATE semantics)
- Every table has an underlying stream; every stream can become a table

### Tasks and Threads
- Kafka Streams splits partitions into **tasks** (one task per partition)
- Tasks run in **stream threads** within your application
- More instances = more parallelism (up to number of partitions)

### Changelog Topics
- Stateful operations back their state stores with internal Kafka topics
- On restart, state is restored from these changelog topics

---

## 4. Setting Up

### Maven dependency
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.7.0</version>
</dependency>
```

### Gradle
```groovy
implementation 'org.apache.kafka:kafka-streams:3.7.0'
```

### Minimum configuration
```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-app");          // unique per app
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
```

---

## 5. Your First Kafka Streams App

Word count — the "Hello World" of stream processing.

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;
import java.util.Arrays;
import java.util.Properties;

public class WordCount {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> textLines = builder.stream("input-topic");

        KTable<String, Long> wordCounts = textLines
            .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
            .groupBy((key, word) -> word)
            .count(Materialized.as("word-count-store"));

        wordCounts.toStream().to("output-topic",
            Produced.with(Serdes.String(), Serdes.Long()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);

        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));

        streams.start();
    }
}
```

### Lifecycle states
```
CREATED → REBALANCING → RUNNING → PENDING_SHUTDOWN → NOT_RUNNING
                ↑_______________|  (on rebalance)
```

---

## 6. KStream vs KTable vs GlobalKTable

### KStream — event stream
```java
KStream<String, String> stream = builder.stream("events");
// Each record is an independent event
// key="user1", value="click"  → processed as new event
// key="user1", value="click"  → processed as another new event
```

### KTable — materialized view
```java
KTable<String, String> table = builder.table("user-profiles");
// Each record UPDATES the value for that key
// key="user1", value="Alice"  → stored
// key="user1", value="Alice2" → overwrites previous
// key="user1", value=null     → tombstone (delete)
```

### GlobalKTable — replicated across all instances
```java
GlobalKTable<String, String> globalTable = builder.globalTable("country-codes");
// Same as KTable but ALL partitions are loaded into EVERY instance
// Used for lookups/joins where you need full dataset locally
// Smaller datasets only — don't use for large tables
```

### Choosing between them
| Scenario | Use |
|---|---|
| Raw event processing | KStream |
| Latest state per key | KTable |
| Lookup/enrichment table | GlobalKTable |
| Large enrichment dataset | KTable + repartition |

---

## 7. Stateless Operations

These don't require state stores.

### filter / filterNot
```java
KStream<String, Integer> filtered = stream.filter((key, value) -> value > 100);
KStream<String, Integer> excluded = stream.filterNot((key, value) -> value < 0);
```

### map / mapValues
```java
// Changes both key and value (causes repartition if key changes)
KStream<String, Integer> mapped = stream.map((key, value) ->
    KeyValue.pair(key.toUpperCase(), value * 2));

// Changes only value (no repartition)
KStream<String, String> valued = stream.mapValues(value -> value.toString());
```

### flatMap / flatMapValues
```java
// One record → many records
KStream<String, String> words = stream.flatMapValues(sentence ->
    Arrays.asList(sentence.split(" ")));
```

### selectKey
```java
// Re-key the stream (triggers repartition)
KStream<String, Order> rekeyed = orders.selectKey((k, order) -> order.getUserId());
```

### peek
```java
// Side-effect only — doesn't change stream
stream.peek((key, value) -> log.info("Processing: {} → {}", key, value));
```

### branch (split)
```java
// Route to different streams based on predicates
Map<String, KStream<String, Event>> branches = stream.split(Named.as("branch-"))
    .branch((k, v) -> v.getType().equals("click"),   Named.as("clicks"))
    .branch((k, v) -> v.getType().equals("view"),    Named.as("views"))
    .defaultBranch(Named.as("other"));

KStream<String, Event> clicks = branches.get("branch-clicks");
```

### merge
```java
KStream<String, Event> merged = stream1.merge(stream2);
```

---

## 8. Stateful Operations

These maintain state in a local store (RocksDB by default).

### count
```java
KTable<String, Long> counts = stream
    .groupByKey()
    .count(Materialized.as("event-counts"));
```

### reduce
```java
KTable<String, Integer> totals = stream
    .groupByKey()
    .reduce(Integer::sum, Materialized.as("running-totals"));
```

### aggregate
```java
KTable<String, MyStats> stats = stream
    .groupByKey()
    .aggregate(
        MyStats::new,                           // initializer
        (key, newValue, aggregate) -> {         // aggregator
            aggregate.add(newValue);
            return aggregate;
        },
        Materialized.<String, MyStats, KeyValueStore<Bytes, byte[]>>as("my-stats")
            .withValueSerde(myStatsSerde)
    );
```

### groupBy vs groupByKey
```java
// groupByKey — no repartition (key already correct)
stream.groupByKey()

// groupBy — repartition by new key
stream.groupBy((key, value) -> value.getCategory())
```

---

## 9. Joins

### Stream-Stream Join (windowed)
Both streams must have matching records **within the same time window**.

```java
KStream<String, Order> orders = builder.stream("orders");
KStream<String, Payment> payments = builder.stream("payments");

KStream<String, EnrichedOrder> enriched = orders.join(
    payments,
    (order, payment) -> new EnrichedOrder(order, payment),  // joiner
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
    StreamJoined.with(Serdes.String(), orderSerde, paymentSerde)
);
```

### Stream-Table Join (no window needed)
```java
KStream<String, Click> clicks = builder.stream("clicks");
KTable<String, UserProfile> profiles = builder.table("user-profiles");

KStream<String, EnrichedClick> enriched = clicks.join(
    profiles,
    (click, profile) -> new EnrichedClick(click, profile)
);
// Note: table lookup is by key — ensure streams share the same key
```

### Stream-GlobalKTable Join
```java
KStream<String, Order> orders = builder.stream("orders");
GlobalKTable<String, Product> products = builder.globalTable("products");

KStream<String, EnrichedOrder> enriched = orders.join(
    products,
    (orderKey, order) -> order.getProductId(),  // key extractor from stream
    (order, product) -> new EnrichedOrder(order, product)
);
```

### Join types summary
| Type | Left | Right | Window? |
|---|---|---|---|
| `join` | Stream | Stream | Yes (required) |
| `leftJoin` | Stream | Stream | Yes (required) |
| `outerJoin` | Stream | Stream | Yes (required) |
| `join` | Stream | KTable | No |
| `leftJoin` | Stream | KTable | No |
| `join` | KTable | KTable | No |
| `outerJoin` | KTable | KTable | No |

---

## 10. Windowing

Group events into time-bounded buckets for aggregation.

### Tumbling Window — fixed, non-overlapping
```java
// Count events per 1-minute window
stream.groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count()
    .toStream()
    .map((windowedKey, count) ->
        KeyValue.pair(windowedKey.key(), count));
```

### Hopping Window — fixed size, overlapping
```java
// 5-minute window, advances every 1 minute
stream.groupByKey()
    .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30))
        .advanceBy(Duration.ofMinutes(1)))
    .count();
```

### Session Window — activity-based, variable size
```java
// New window if no events for 5 minutes (inactivity gap)
stream.groupByKey()
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5)))
    .count();
```

### Sliding Window — event-time based, continuous
```java
// All events within 10 minutes of each other
stream.groupByKey()
    .windowedBy(SlidingWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10)))
    .count();
```

### Grace period
```java
// Allow late records up to 30 seconds after window closes
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(30))
```

---

## 11. State Stores

### Accessing state store from outside the topology
```java
KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();

// Query local store
ReadOnlyKeyValueStore<String, Long> store = streams.store(
    StoreQueryParameters.fromNameAndType("word-count-store",
        QueryableStoreTypes.keyValueStore())
);

Long count = store.get("hello");

// Iterate all entries
try (KeyValueIterator<String, Long> iter = store.all()) {
    while (iter.hasNext()) {
        KeyValue<String, Long> entry = iter.next();
        System.out.println(entry.key + ": " + entry.value);
    }
}
```

### Custom state store (persistent)
```java
StoreBuilder<KeyValueStore<String, Long>> storeBuilder =
    Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("my-store"),
        Serdes.String(),
        Serdes.Long()
    );

builder.addStateStore(storeBuilder);
```

### In-memory store (non-persistent)
```java
Stores.inMemoryKeyValueStore("my-in-memory-store")
// Lost on restart — no changelog topic written
```

---

## 12. Processor API

Low-level API for full control — use when DSL isn't enough.

```java
public class MyProcessor implements Processor<String, String, String, String> {
    private ProcessorContext<String, String> context;
    private KeyValueStore<String, Long> stateStore;

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
        this.stateStore = context.getStateStore("my-store");

        // Schedule a punctuation (timer)
        context.schedule(
            Duration.ofSeconds(30),
            PunctuationType.WALL_CLOCK_TIME,
            timestamp -> {
                // Called every 30s regardless of events
                stateStore.all().forEachRemaining(entry ->
                    context.forward(new Record<>(entry.key, entry.value.toString(), timestamp))
                );
            }
        );
    }

    @Override
    public void process(Record<String, String> record) {
        Long current = stateStore.get(record.key());
        stateStore.put(record.key(), current == null ? 1L : current + 1);
        // Optionally forward downstream
        context.forward(record.withValue(record.value().toUpperCase()));
    }

    @Override
    public void close() {}
}

// Wire it into topology
Topology topology = new Topology();
topology.addSource("Source", "input-topic")
        .addProcessor("MyProcessor", MyProcessor::new, "Source")
        .addStateStore(storeBuilder, "MyProcessor")
        .addSink("Sink", "output-topic", "MyProcessor");
```

---

## 13. Error Handling

### Deserialization errors
```java
props.put(StreamsConfig.DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG,
    LogAndContinueExceptionHandler.class);  // skip bad records
    // or: LogAndFailExceptionHandler (default — stops app)
```

### Production errors
```java
props.put(StreamsConfig.DEFAULT_PRODUCTION_EXCEPTION_HANDLER_CLASS_CONFIG,
    DefaultProductionExceptionHandler.class);
```

### Custom exception handler
```java
public class MyDeserializationHandler implements DeserializationExceptionHandler {
    @Override
    public DeserializationHandlerResponse handle(
            ProcessorContext context,
            ConsumerRecord<byte[], byte[]> record,
            Exception exception) {
        log.error("Failed to deserialize record from topic={}", record.topic(), exception);
        // Send to dead-letter topic here if needed
        return DeserializationHandlerResponse.CONTINUE;
    }
}
```

### Uncaught exception handler (Kafka Streams 2.8+)
```java
streams.setUncaughtExceptionHandler(exception -> {
    log.error("Uncaught exception", exception);
    return StreamThreadExceptionResponse.REPLACE_THREAD;  // restart the thread
    // or: SHUTDOWN_CLIENT / SHUTDOWN_APPLICATION
});
```

---

## 14. Testing

### TopologyTestDriver — fast, no Kafka needed
```java
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;

class WordCountTest {
    TopologyTestDriver testDriver;
    TestInputTopic<String, String> inputTopic;
    TestOutputTopic<String, Long> outputTopic;

    @BeforeEach
    void setup() {
        StreamsBuilder builder = new StreamsBuilder();
        WordCount.buildTopology(builder);  // extract topology logic to testable method

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:9092");

        testDriver = new TopologyTestDriver(builder.build(), props);

        inputTopic = testDriver.createInputTopic(
            "input-topic", new StringSerializer(), new StringSerializer());
        outputTopic = testDriver.createOutputTopic(
            "output-topic", new StringDeserializer(), new LongDeserializer());
    }

    @AfterEach
    void teardown() {
        testDriver.close();
    }

    @Test
    void testWordCount() {
        inputTopic.pipeInput("key1", "hello world hello");

        Map<String, Long> results = outputTopic.readKeyValuesToMap();
        assertEquals(2L, results.get("hello"));
        assertEquals(1L, results.get("world"));
    }

    @Test
    void testWithTimestamps() {
        Instant t0 = Instant.parse("2024-01-01T00:00:00Z");
        inputTopic.pipeInput("k", "event1", t0);
        inputTopic.pipeInput("k", "event2", t0.plusSeconds(30));
    }
}
```

---

## 15. Deployment & Scaling

### Parallelism model
```
Input topic: 6 partitions
→ Kafka Streams creates 6 tasks
→ Instance 1 (2 threads): handles tasks 0,1,2,3
→ Instance 2 (2 threads): handles tasks 4,5
→ Add Instance 3: Kafka Streams rebalances automatically
```

### Thread count per instance
```java
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
// Max useful = number of input partitions
```

### Scaling out
- Run multiple instances with the **same `application.id`**
- Kafka Streams auto-assigns partitions across instances
- State stores are local — each instance only holds state for its assigned partitions
- Use Interactive Queries to query state across all instances

### Interactive Queries (distributed state)
```java
// Find which instance owns a key
KeyQueryMetadata metadata = streams.queryMetadataForKey(
    "word-count-store", "hello", Serdes.String().serializer());

HostInfo activeHost = metadata.activeHost();
// If activeHost != this instance, proxy the HTTP request to it
```

### Recommended instance configuration
```java
// For production
props.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);  // hot standby for fast failover
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000); // 1s commit interval
```

---

## 16. Production Best Practices

### Configuration
```java
// Exactly-once semantics (requires Kafka 2.5+, 3x performance cost)
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);

// At-least-once (default, better performance)
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.AT_LEAST_ONCE);

// RocksDB tuning for high-throughput
props.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG, MyRocksDBConfig.class);

// State store location (use fast SSD)
props.put(StreamsConfig.STATE_DIR_CONFIG, "/mnt/ssd/kafka-streams");
```

### Topology design rules
1. **Re-keying causes repartition** — `map()`, `selectKey()`, `groupBy()` write to internal repartition topics. Minimize these.
2. **Prefer `mapValues()` over `map()`** when you don't change the key.
3. **Filter early** — push filters upstream to reduce data volume.
4. **Use `GlobalKTable` for small lookup tables** only (fits in memory).
5. **Set grace periods** on windows to handle late data.

### Monitoring key metrics
| Metric | What to watch |
|---|---|
| `records-consumed-rate` | Processing throughput |
| `commit-latency-avg` | State commit health |
| `poll-records-avg` | Consumer poll efficiency |
| `process-rate` | Records processed/sec |
| `record-lag-max` | Consumer lag — rising = trouble |
| RocksDB `block-cache-miss-ratio` | State store efficiency |

### Graceful shutdown pattern
```java
KafkaStreams streams = new KafkaStreams(topology, props);

streams.setStateListener((newState, oldState) -> {
    log.info("State changed: {} → {}", oldState, newState);
});

streams.setUncaughtExceptionHandler(ex ->
    StreamThreadExceptionResponse.REPLACE_THREAD);

Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    streams.close(Duration.ofSeconds(30));  // wait up to 30s for clean shutdown
}));

streams.start();
```

### Common pitfalls
| Pitfall | Fix |
|---|---|
| Changing `application.id` | Don't — it's tied to consumer group and internal topic names |
| Deleting changelog topics | Let Kafka Streams manage them |
| Key is null | Stateful ops require non-null keys |
| Clock skew between producers | Use event-time with `TimestampExtractor` |
| State store grows unbounded | Set `retention.ms` on changelog topics; use windowed stores |

---

## Quick Reference Card

```
StreamsBuilder builder = new StreamsBuilder();

// Sources
KStream<K,V>       builder.stream("topic")
KTable<K,V>        builder.table("topic")
GlobalKTable<K,V>  builder.globalTable("topic")

// Transform (stateless)
.filter()  .filterNot()  .map()  .mapValues()
.flatMap()  .flatMapValues()  .selectKey()
.peek()  .branch()  .merge()

// Aggregate (stateful)
.groupByKey()  .groupBy()
  → .count()  .reduce()  .aggregate()
  → .windowedBy(...)  → .count() / .aggregate()

// Join
stream.join(stream, joiner, JoinWindows...)     // stream-stream
stream.join(table, joiner)                       // stream-table
stream.join(globalTable, keyExtractor, joiner)   // stream-globalTable
table.join(table, joiner)                        // table-table

// Sink
.to("output-topic")
.toTable()
```

---

## 17. Real-World Scenarios

The best way to internalize Kafka Streams is to see how it solves actual problems. Here are common scenarios with full solutions.

### Scenario 1: Real-Time Fraud Detection
**Problem:** Detect users with more than 5 failed login attempts in 1 minute.

```java
KStream<String, LoginEvent> logins = builder.stream("login-events");

logins
    .filter((userId, event) -> event.getStatus().equals("FAILED"))
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count()
    .toStream()
    .filter((windowedKey, count) -> count >= 5)
    .map((windowedKey, count) ->
        KeyValue.pair(windowedKey.key(),
            new FraudAlert(windowedKey.key(), count, windowedKey.window().start())))
    .to("fraud-alerts");
```

**What's happening:**
1. Filter only failed logins
2. Group by user ID (the key)
3. Tumble through 1-minute windows
4. Count failures per user per window
5. Flag windows with ≥5 failures
6. Emit alerts to a downstream topic

### Scenario 2: Order Enrichment with User Profile
**Problem:** Each order has a `userId`. Enrich it with full user details from a slow-changing user profile table.

```java
// User profiles change rarely → KTable
KTable<String, UserProfile> users = builder.table("user-profiles");

// Orders are events → KStream
KStream<String, Order> orders = builder.stream("orders");

// Stream-Table join (no window needed; lookup by key)
KStream<String, EnrichedOrder> enriched = orders.join(
    users,
    (order, user) -> new EnrichedOrder(order, user.getEmail(), user.getCountry())
);

enriched.to("enriched-orders");
```

**Why a KTable?** User profiles are state (latest value per user matters), not events. The join uses the *current* profile when the order arrives.

### Scenario 3: Top-K Trending Products
**Problem:** Track the top 10 best-selling products in the last 5 minutes.

```java
KStream<String, Sale> sales = builder.stream("sales");

sales
    .map((k, sale) -> KeyValue.pair(sale.getProductId(), 1L))
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5))
        .advanceBy(Duration.ofSeconds(30)))  // hopping window
    .count()
    .toStream()
    // Re-key everything to "ALL" so we can rank globally per window
    .map((windowedKey, count) ->
        KeyValue.pair("ALL_PRODUCTS",
            new ProductCount(windowedKey.key(), count, windowedKey.window().start())))
    .groupByKey()
    .aggregate(
        TopK::new,
        (key, productCount, topK) -> topK.add(productCount, 10),  // keep top 10
        Materialized.with(Serdes.String(), topKSerde)
    )
    .toStream()
    .to("trending-products");
```

### Scenario 4: Sessionizing User Activity
**Problem:** Group user clicks into sessions where 30 minutes of inactivity = new session.

```java
KStream<String, Click> clicks = builder.stream("clicks");

clicks
    .groupByKey()
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
    .aggregate(
        Session::new,
        (userId, click, session) -> session.addClick(click),
        (userId, leftSession, rightSession) -> leftSession.merge(rightSession),  // merger
        Materialized.with(Serdes.String(), sessionSerde)
    )
    .toStream()
    .to("user-sessions");
```

**Note:** Session windows need a **merger** function — if two windows overlap, they get merged into one.

### Scenario 5: Materialized View for Read API
**Problem:** Build a query API to look up the current order count per user without hitting the database.

```java
KStream<String, Order> orders = builder.stream("orders");

KTable<String, Long> orderCounts = orders
    .groupByKey()
    .count(Materialized.as("order-counts-store"));

// Inside your REST controller:
@GetMapping("/users/{userId}/order-count")
public Long getOrderCount(@PathVariable String userId) {
    ReadOnlyKeyValueStore<String, Long> store = streams.store(
        StoreQueryParameters.fromNameAndType(
            "order-counts-store",
            QueryableStoreTypes.keyValueStore()));
    Long count = store.get(userId);
    return count == null ? 0L : count;
}
```

This is **CQRS in a box** — your read model is automatically maintained by Kafka Streams.

### Scenario 6: Dead-Letter Queue Pattern
**Problem:** Bad records shouldn't crash the app, but you don't want to silently drop them.

```java
KStream<String, String> raw = builder.stream("raw-events");

KStream<String, Result<Event>> parsed = raw.mapValues(value -> {
    try {
        return Result.ok(parseEvent(value));
    } catch (Exception e) {
        return Result.error(value, e.getMessage());
    }
});

// Split into success and failure streams
Map<String, KStream<String, Result<Event>>> branches = parsed.split()
    .branch((k, v) -> v.isOk(), Branched.as("success"))
    .branch((k, v) -> v.isError(), Branched.as("error"))
    .noDefaultBranch();

branches.get("success")
    .mapValues(Result::value)
    .to("clean-events");

branches.get("error")
    .mapValues(r -> r.errorMessage() + " | original: " + r.rawValue())
    .to("dead-letter-queue");
```

---

## 18. Common Beginner Mistakes

These are the mistakes I've seen (and made) repeatedly. Avoiding them puts you ahead of 90% of beginners.

### Mistake 1: Confusing KStream with KTable
**Symptom:** Aggregations produce wrong results.

**Wrong:**
```java
// Reading user profiles as a stream — every update is treated as a NEW event
KStream<String, UserProfile> users = builder.stream("user-profiles");
```

**Right:**
```java
// Profiles represent state — use a KTable
KTable<String, UserProfile> users = builder.table("user-profiles");
```

**Rule of thumb:** If the topic represents **changes to current state** (CRUD operations, updates), use `table()`. If it represents **independent events** (clicks, payments, logs), use `stream()`.

### Mistake 2: Forgetting That Joins Are Co-Partitioned
**Symptom:** Join produces no output, even though data exists in both topics.

For a join to work, both sides must be:
1. **Same number of partitions**
2. **Same partitioning strategy** (same key, same partitioner)

**Fix:** If they don't match, re-key one side and let Kafka Streams auto-repartition:
```java
KStream<String, Order> orders = builder.stream("orders");
KStream<String, Payment> payments = builder.stream("payments")
    .selectKey((k, payment) -> payment.getOrderId());  // re-key by orderId

// Now partitions are aligned (both keyed by orderId)
KStream<String, EnrichedOrder> joined = orders.join(payments, ...);
```

### Mistake 3: Not Understanding `application.id`
**Symptom:** App reprocesses old data when restarted, or doesn't restore state correctly.

The `application.id`:
- Becomes the consumer group ID
- Becomes the prefix for all internal topics
- **MUST be unique per logical app**, but **MUST be the same across all instances of the same app**

**Common error:** Using a UUID for `application.id` — every restart looks like a "new app" and reprocesses everything.

### Mistake 4: Treating Kafka Streams Like a Database
**Symptom:** Trying to do JOINs across keys, range queries on streams, etc.

Kafka Streams is **partitioned by key**. You cannot:
- Do a "WHERE non_key_field = X" query efficiently
- Join across partitions arbitrarily
- Update a record by primary key (must produce a new event)

**Mental shift:** Think "event log + materialized views" — not "real-time SQL".

### Mistake 5: Ignoring Backpressure & Lag
**Symptom:** App seems healthy but is hours behind real-time.

Always monitor `consumer-lag`. Kafka Streams won't crash if it's slow — it'll just fall behind silently.

**Fix:**
- Add more partitions to the input topic (if too few)
- Add more instances (if CPU-bound)
- Increase `NUM_STREAM_THREADS_CONFIG` (if cores are idle)
- Profile your processing logic — slow processors block the whole task

### Mistake 6: Producing in a `process()` / `transform()`
**Symptom:** Bypassing Kafka Streams' exactly-once and ordering guarantees.

**Wrong:**
```java
stream.process(() -> new Processor() {
    public void process(Record r) {
        kafkaProducer.send(...);  // ❌ external producer breaks guarantees
    }
});
```

**Right:** Use `context.forward()` or write to an output topic via the topology:
```java
stream.to("output-topic");  // Kafka Streams handles transactional writes
```

### Mistake 7: Not Naming Operators
**Symptom:** Adding an operator in the middle of your topology breaks state recovery.

Auto-generated names like `KSTREAM-FILTER-0001` are based on **position in the topology**. If you insert a step, all downstream names shift, **invalidating internal topics**.

**Right:** Always name your operators in production:
```java
stream
    .filter((k, v) -> v.isValid(), Named.as("valid-events"))
    .groupByKey(Grouped.as("group-by-user"))
    .count(Materialized.as("user-counts"));
```

### Mistake 8: Misunderstanding Time
**Symptom:** Windowed aggregations seem to fire at random times.

There are three types of time:
- **Event time** — when the event actually happened (in the payload)
- **Ingestion time** — when Kafka received it
- **Processing time** — when your app processed it

Kafka Streams uses **event time by default**. If your producers don't set timestamps, you'll get unpredictable behavior.

**Fix:** Set timestamps in producers, OR use a custom `TimestampExtractor`:
```java
public class OrderTimestampExtractor implements TimestampExtractor {
    public long extract(ConsumerRecord<Object, Object> record, long partitionTime) {
        Order order = (Order) record.value();
        return order.getCreatedAt().toEpochMilli();
    }
}
```

### Mistake 9: Storing Too Much in State Stores
**Symptom:** Disk fills up, restoration takes hours.

State stores grow unboundedly unless you actively manage them.

**Fix:**
- Use **windowed stores** with retention
- Send tombstones (`null` values) to delete keys in compacted KTables
- Set `cleanup.policy=compact,delete` with a retention on changelog topics

### Mistake 10: Punctuators Doing Heavy Work
**Symptom:** App stalls every N seconds.

Punctuators run **on the same thread** as your `process()` method. If a punctuator iterates a 10M-key state store, your app pauses processing for the duration.

**Fix:** Keep punctuators light. Offload heavy work to async threads if needed (with care for ordering).

---

## 19. Learning Path

A practical order of skills to learn, with milestones to mark "done":

### Week 1: Fundamentals
- [ ] Run a local Kafka cluster (use `docker-compose` with Bitnami images or Confluent Platform)
- [ ] Write a producer & consumer in plain Kafka client API (so you understand what Streams hides)
- [ ] Build the WordCount example end-to-end
- [ ] Print and read your topology with `topology.describe()`
- **Milestone:** Explain to a colleague the difference between KStream and KTable using a real example.

### Week 2: Stateless Operations
- [ ] Use `filter`, `map`, `mapValues`, `flatMap`, `branch`, `merge`
- [ ] Understand when re-keying triggers a repartition topic
- [ ] Build a simple ETL pipeline: read JSON, transform, write to another topic
- **Milestone:** Build an event-router that splits a single input topic into 3 output topics by event type.

### Week 3: Stateful Operations & Joins
- [ ] Use `groupByKey().count()`, `reduce()`, `aggregate()`
- [ ] Do a stream-table join (e.g., enrich orders with user data)
- [ ] Do a stream-stream windowed join (e.g., correlate clicks and purchases)
- [ ] Query a state store via Interactive Queries
- **Milestone:** Build a real-time leaderboard for a fake game with score updates.

### Week 4: Windowing & Time
- [ ] Use tumbling, hopping, and session windows
- [ ] Configure grace periods for late data
- [ ] Set up a custom `TimestampExtractor`
- [ ] Understand how watermarks (stream time) advance
- **Milestone:** Build a metrics dashboard: events-per-minute, p95 latency over rolling 5-min window.

### Week 5: Production Readiness
- [ ] Write tests with `TopologyTestDriver`
- [ ] Set up monitoring (JMX → Prometheus)
- [ ] Configure error handlers (deserialization, production, uncaught)
- [ ] Run multiple instances; observe rebalance behavior
- [ ] Implement graceful shutdown
- **Milestone:** Deploy a Kafka Streams app with at least 3 instances, kill one, verify zero data loss.

### Week 6+: Advanced Topics
- [ ] Use the Processor API for custom logic
- [ ] Configure exactly-once semantics
- [ ] Tune RocksDB for high-throughput
- [ ] Explore ksqlDB as a higher-level alternative
- [ ] Read the [Kafka Streams source code](https://github.com/apache/kafka/tree/trunk/streams) for deep understanding

### Recommended Resources
- 📖 **Book:** "Kafka Streams in Action" by Bill Bejeck (the practical bible)
- 📖 **Book:** "Mastering Kafka Streams and ksqlDB" by Mitch Seymour
- 📺 **Video Series:** Confluent Developer's Kafka Streams 101 course (free, on-demand)
- 📚 **Docs:** [kafka.apache.org/documentation/streams](https://kafka.apache.org/documentation/streams/)
- 💬 **Community:** Confluent Community Slack (#kafka-streams channel)

---

## 20. Glossary

Definitions in plain language — refer back here when terminology gets confusing.

| Term | What it means (plain English) |
|---|---|
| **Topology** | The blueprint/DAG of your stream processing app. |
| **Sub-topology** | A connected piece of the topology. Each sub-topology can run as parallel tasks. |
| **Task** | A unit of work; one task per partition per sub-topology. The smallest unit of parallelism. |
| **Stream Thread** | An OS thread that runs one or more tasks. Configured via `num.stream.threads`. |
| **KStream** | An unbounded sequence of immutable events. Like a transaction log. |
| **KTable** | A continuously updated table representing the latest value per key. Like a database row. |
| **GlobalKTable** | A KTable that is **fully replicated** to every instance — useful for small reference data. |
| **State Store** | A local key-value store (RocksDB or in-memory) used to remember things between events. |
| **Changelog Topic** | An internal Kafka topic that backs up a state store, used for fault recovery. |
| **Repartition Topic** | An internal Kafka topic created when you re-key a stream, ensuring same keys land on same partitions. |
| **Co-partitioning** | When two topics being joined have matching partition counts and key strategies. Required for joins. |
| **Tombstone** | A `null` value sent for a key — signals "delete this key" in compacted topics/KTables. |
| **Watermark / Stream Time** | The maximum timestamp seen so far. Drives when windows close. |
| **Grace Period** | Extra time to wait after a window's end for late events to arrive. |
| **Punctuator** | A scheduled callback (timer) that fires on wall-clock or stream-time intervals. |
| **Interactive Queries** | Reading state stores directly from your app — turns Kafka Streams into a queryable cache. |
| **Exactly-Once Semantics (EOS)** | Guarantee that each event is processed exactly once, even with failures. Costs ~3x throughput. |
| **At-Least-Once** | Default mode; events may be processed multiple times on failure. Cheaper, simpler. |
| **Standby Replica** | A hot copy of a state store on another instance, for fast failover. |
| **Application ID** | The unique name for your Streams app. Becomes the consumer group ID. |
| **Serde** | Serializer + Deserializer — Kafka Streams needs both to read/write each topic. |
| **DSL** | Domain-Specific Language — the high-level fluent API (`filter`, `map`, `groupBy`...). |
| **Processor API** | The low-level API where you write `Processor` classes by hand. |
| **Materialized View** | A KTable that is queryable via Interactive Queries (i.e., backed by a named store). |
| **Headers** | Metadata on each record (key/value pairs) — useful for tracing, routing, schema versioning. |

---

## Final Words: How to Actually Get Good

After all this reading, the truth is that **mastery comes from running things in production and watching them break.** Here's my advice:

1. **Build a toy project end-to-end first.** Don't try to design "the perfect topology" upfront.
2. **Print your topology.** Every. Single. Time. Before you deploy.
3. **Watch consumer lag like a hawk.** It's the #1 indicator of health.
4. **Read changelog topics manually with `kafka-console-consumer`** — seeing the raw state changes will demystify everything.
5. **Embrace the trade-offs.** Exactly-once is slower. Joins repartition. State stores need disks. There are no free lunches.
6. **When in doubt, simplify.** Most "complex" Kafka Streams apps are actually multiple simple apps glued together.

Kafka Streams is one of the most elegant pieces of software in the data world — but only if you respect its model. Treat it as an event-driven library, not a database, and you'll go far.

Happy streaming. 🚀

---

*Guide covers Kafka Streams 3.x — most concepts apply to 2.x with minor API differences.*
*Last updated: 2026-05-03*
