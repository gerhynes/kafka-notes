# Kafka Notes

Sources:

- [Kafka Docs](https://kafka.apache.org/documentation/)
- [Apache Kafka Fundamentals](https://www.youtube.com/playlist?list=PLa7VYi0yPIH2PelhRHoFR5iQgflg-y6JA)
- [Kafka 101](https://www.youtube.com/playlist?list=PLa7VYi0yPIH0KbnJQcMv5N9iW8HkZHztH)
- [Apache Kafka Series - Learn Apache Kafka for Beginners v3](https://www.udemy.com/course/apache-kafka/)

Kafka is an event-streaming platform used to collect, store and process real-time data streams at scale.

Kafka lets you:

- publish and subscribe to streams of records
- effectively store streams of records in the order in which they were generated
- process streams of records in real time

It's use cases include distributed logging, stream processing and pub-sub messaging.

Kafka combines messaging, storage, and stream processing to allow storage and analysis of both historical and real-time data.

### Motivation and Use Cases

There is a paradigm shift underway in favour of event-driven architectures. The way we build systems had been focused on state. There's been a shift from data as things to data as events.

This leads to

- single platform to connect everyone to every event
- real-time stream of events
- all events stored for historical view

Over 35% of Fortune 500 companies use Kafka.

Kafka is used for:

- real-time fraud detection
- automotive telemetry
- real-time analysis in e-commerce
- 360 views of customers
- payment processing in banking
- IOT medical devices
- online gaming
- online financial services

### Events

Unlike traditional databases, Kafka encourages you to think of events (a customer changing their shipping address, a train unloading cargo) first and things second.

Events can be

- OT reports
- business process change
- user interaction
- microservice output

An event is a combination of notification (when) and state (usually small, < 1MB). State is serialized and represeted in a standard structured format , like JSON, JSON Schema, Avro, Protocol Buffers.

An event is modelled as a key-value pair. Internally Kafka is loosley typed. The serialized object is usually the representation of an application domain object, or some form of raw message input (like the output of a sensor). Keys are often strings or integers. They're usually not a unique identifier but point to an entity in the system, like a user, order, or particular device.

### Topics

Kafka stores data in a log (or topic), an ordered sequence of events. A topic is an ordered collection of events that are stored in a durable way (written to disk and replicated). Topics can store data for a short perod of time, or indefinitely. They can be small or enormous.

You create different topics to hold different kinds of events, and you can create different topics to hold filtered and transformed versions of the same kind of event.

Logs are append only and not indexed. You read by seeking to a specific offset and then scanning sequential log entries from there. Events in a log are immutable. Because logs are simple data structures, Kafka is able to get a lot of performance out of little resources.

Topics can be configured for the messages in it to expire after they've reached a certain age (or once the topic has reached a certain size). Logs are written to disk as files.

The trend in software development has been to move from monoliths to small programs that are easier to think about, version and change. These microservices can talk to each other through Kafka topics.

With data as events in topics that get processed as soon as they happen, it's relatively straightforward to build services that can provide analysis in real time.

By default, topic data is stored for 1 week. This can be configured globally in a cluster or per topic.

If you are only concered wth the current state of an entity and not previous values (for example your profile picture), you can use a compacted topic. It just stores the most recent record associated with a unique key.

### Partitions

Kafka is designed to operate across distributed systems. Partitioning takes a single topic log and breaks it into multiple logs, each of which can live on a separate node in the Kafka cluster. Partitions are strictly ordered and append only.

If a message has no key, these will be distributed round robin among the topic's partitions.

If the message does have a key, that key is used to determine which topic to put that message in. The key is run through a hash function and you take the output of that hash function, mod the number of partitions, and the resulting number is the partition number to write to. This guarantees that messages with the same key always land in the same partition and therefore are always in order.

Each partition can be broken up into multiple segments on disk in the broker. Data is expired per segment. When the newest record in a segment is older than a specified retention period, the segment gets deleted.

There's no correlation between the number of partitions and brokers.

### Records/Messages

Kafka records consist of key-value pairs, optional headers and a timestamp (creation time or injection time). The value is usually structured, being a serialized domain object, and there can be structure to the key.

If you don't provide a timestamp, one will be created. You can explicitly specify the timestamp in the API.

The optional headers store metadata which consumers have access to.

### Brokers

From a physical infrastructure standpoint, Kafka is composed of a network of independent machines/containers called brokers.

Each broker hosts some set of Kafka partitions and handles incoming requests to write new events to those partitions or read events from them. It handles partition storage and pub/sub.

Brokers also handle replication of partitions between each other.

### Replication

Brokers, and their underlying storage, are susceptible to failure. You need to copy partition data to several other brokers (3 is typical) to keep it safe. These copies are called "follower replicas". Whereas the main partition is called the "leader replica".

Every partition that gets replicated has one leader and N-1 followers. When you produce data to the partition, you're really producing it to the leader. In general, writes and reads happen to the leader. After that, the leader and followers work together to get replication done.

You usually don't need to think about replication as a developer building systems on Kafka. Replication is turned on by default and happens automatically. There are settings that can be tuned to produce varying levels of durability guarantee.

If a leader fails, one of the followers will automatically be elected as the new leader.

### Producers

Every component of a Kafka platform that isn't a broker is ultimately a producer, a consumer, or both.

Poducers and consumers are decoupled so they don't need to know about each other. Their speed doesn't affect each other. They can fail or evolve independently.

Java is the native language of Kafka. There are wrappers for the other JVM languages to make the Java library look idiomatic in those languages. Other languages are supported by Confluent or the community.

The `KafkaProducer` class is what lets you connect to the cluster. You give it a map of configuration parameters, such as the address of some brokers in the cluster, any security configuration and any other settings that determine the network behaviour of the producer.

The `ProducerRecord` class holds the topic and the key-value pair you want to send to the cluster.

You call the `send` method on `KafkaProducer` to send messages. The message goes through serialization (where you have already told the library what the type of the key and value are). The partitioner looks for a key and partitions the message accordingly. The partitioner acts as a buffer, and can be tuned to trade off between throughput and latency.

The producer potentially waits for acknowledgement when sending a message. You can have no ack, ack from the leader, or have the followers ack the leader and then the leader ack the producer. It's a tradeoff between latency and security.

Delivery guarantees can be "at most once", "at least once" or "exactly once". Kafka has strong transactional guarantees. Messages may get delivered more than once, but clients can be prevented from processing duplicate messages with failures handled gracefully.

`KafkaProducer` is managing connection pools, doing network buffering, waiting for brokers to acknowledge messages, reransmitting messages when necessary.

The producer makes the decision about which partition to send each message to, whether round robin, by key, or even by a custom configuration scheme.

### Consumers

Each topic can have multiple consumers.

You use a `KafkaConsumer` class to connect to the cluster, pass it a map of confguration parameters, then use that connection to subscribe to one or more topics. This can be a list of topics or a regular expresson that matches some topics (if you're brave).

Consumers live in groups and you can scale out consumers by deploying multiple instances in the same group.

Consumers work from independent offsets, usually as close to the present as possible but you can have a consumer start fom the earliest existing record and catch up over time. The offset of each consumer into each partition is stored in the consumer's memory and also in a special topic in the Kafka cluster named Consumer Offsets.

You set up event handlers (`OnMessage`, `OnError`, `OnConsumeError`) in the consumer to respond when messages happen.

When messages are available on those topics they come back in a collection called `ConsumerRecords`. This contains individual instances of messages in the form of instances of an object called `ConsumerRecord`. The `ConsumerRecord` is the actual key-value pair of a single message.

You call the `subscribe` method on consumer to subscribe to one or more topics and then use `poll` in a loop to check for messages.

`KafkaConsumer` manages connection pooling to the cluster, keeping up to date with cluster metadata, and manages the network protocol.

Kafka is different from legacy message queues in that reading a message does not destroy it. It's still there to be read by any other consumer. It's normal in Kafka for many consumers to read from one topic.

Scaling consumer groups is more or less automatic. A single instance of a consuming application will always receive the messages from all of the partitions in the topic it's subscribed to.

If you add a second instance of the same consuming application, it triggers an automatic rebalancing process in which the Kafka cluster, combined with the client nodes, work together to attempt to distribute partitions fairly between the two instances. This rebalancing repeats each time you add or remove a consumer group instance. This makes each consuming application horizontally and elastically scalable by default. You just need to specify a group id parameter when you're creating the consumer.

In a traditional messaging queue you can often scale the number of consumers but you would usually miss out on ordering guarantees altogether. In Kafka, if your key is not null you still have that ordering by key guarantee, which is preserved as you scale out the consumer group.

Most interesting consumers are stateful and remember something about messages that came before. The consumer API doesn't migrate state. For that, you'd need Kafka Streams or ksqlDB.

### Troubleshooting

Each node will have its own log files. You can also set up centralized logging.

### Security

Kafka supports encryption in transit, as well as authentication and authorization.

There isn't out of the box encryption of data at rest, stored in partitions. You can either apply full disk encryption or write a wrapper around the producer and consumer libraries to handle encryption.

## Kafka Ecosystem

Ideally you shouldn't write infrastructure code yourself, but should rely on the community or infrastructure vendors. This is where the ecosystem comes in.

### Kafka Connect

Kafka Connect is an integration API (a client application with an ecosystem of pluggable connectors) that helps get data into Kafka from other services and back out again. To a Kafka cluster, Connect looks like a producer or a consumer, or both.

Connect is a server process that runs on hardware independent of the Kafka brokers themselves. It's designed to be scalable and fault tolerant. You can have a cluster of connect workers to share the load of moving data in and out of Kafka.

Connect abstracts a lot of the data integration code away from the developer and instead just requires some JSON configuration to run a connector. There are connectors for a wide variety of data sources and sinks.

A Connect worker (one of the nodes in a Connect cluster) runs one or more tasks. Each task is a piece of a connector. A connector is a pluggable component (a JAR file of JVM connect code) that's responsible for interfacing with an external syste. Connector also has a runtime meaning, when the JAR becomes instantiated as a runtime entity.

A connector can be partitioned into multiple pieces, for example if its reading from multiple sources. The current state of each connector is stored in a Kafka topic, allowing for failover to another connector.

A source connector reads data from an external system and produces it to a Kafka topic.

A sink connector subscribes to one or more Kafka topics and then writes the messages to an external system.

Each connector is a source or sink connector but the Kafka cluster only sees a producer or consumer.

One advantage of Connect is the huge ecosystem of connectors available to handle common tasks, such as writing to a relational database. https://www.confluent.io/hub/ is a curated selection of connectors.

Sometimes you will have to write your own connector if one doesn't exist. The difficulty of writing a connector usually lies in the external interface, not the Connect API.

### Confluent REST Proxy

If the language you're working in doesn't have a Kafka libary, you can use the Confluent REST Proxy to access a REST interface to a Kafka cluster.

### Schema Registry

Over time new consumers will be added to topics and topic schemas will evolve. Schema Registry is a standalone server process that runs on a machine external to the Kafka brokers. Its job is to maintain a database of schemas for all the topics in the cluster for which it is responsible.

That database is persisted in an internal Kafka topic and its cached in the Schema Registry for low latency access.

Schema Registry can be run in a redundant high availability configuration.

Schema Registry is also an API that allows producers and consumers to predict whether the message they're about to produce or consume is compatible wth the verson they're expecting, or previous versions.

When a producer is configured to use the Schema Registry, it calls at produe time an API at the Schema Registry REST endpoint. If the schema of the new message is the same as the last produced, or is different but matches the cmpatibility rules defiined for the topic, the produce may succeed. If it vilates the compatabilty rules, the produce will fail in a way that the applcation code can detect. You are made aware f that condition and can handle it, rather than producing data that is going to be incompatible down the line.

If a consumer reads a message that has an incompatible schema from the version that it expects, Schema Registry will tell it not to consume the message.

These schemas have immutable ids and get cached in the producer and consumer to prevent latency.

Schema Registry supports three serialization formats:

- JSON Schema
- Avro
- ProtoBuf

Depending on the format, you may have an Interface Description Language (IDL) where you can describe in a source controllable text file the schema f the objects in question.

### Kafka Streams

Consumers tend to grow in complexity. The Consumer API is relatively simple and so you'll need framework code for things like time windows, late-arriving messages, out of order messages, lookup tables, aggregating by key, etc.

Aggregation and enrichment operations are typically stateful, taking up memory in the program's heap, and becoming a fault tolerance liability. If the processing application goes down, its memory is lost unless it's explicitly persisted elsewhere.

Kafka Streams is a Java API for transforming and enriching streams of data. It gives you easy access to the computational primitives of stream processing: filtering, grouping, aggregating, joining. A stream is an unbounded, continuous real-time flow of records. Records are structured as key-value pairs.

Kafka Streams supports per-record processing, with millisecond latency. It supports stateless processing, stateful processing and windowing operations. The processing runs in your microservice, not in a separate processing cluster. It's elastically scalable and fault-tolerant. It's also deployment agnostic. It lets you divide work among the instances in your processing cluster.

Kafka Streams provides scalable, fault-tolerant support for the potentially large amounts of state that result from stream processing computations. It also handles the distributed state problems that come with consumer groups. It manages state off-heap, persists it to local disk and persists that same state to internal topics in the Kafka cluster.

Because Kafka Streams is a Java library (not new infrastructure), it easily integrates with your other services, such as REST APIs, and turns them into scalable, fault-tolerant stream processing applications. It runs in the context of your existing application and doesn't require special infrastructure.

### ksqlDB

ksqlDB is an event-streaming database optimized for stream processing applications with Kafka. It runs on its own scalable, fault-tolerant cluster adjacent to the Kafka cluster.

It exposes a REST interface (or command line interface) to applications. They can submit new stream processing jobs written in SQL and query the results of those stream processing programs, also in SQL.

There is a Java library that provides an idiomatic Java wrapper for ksql.

ksql also provides an integration with Kafka Connect, letting you connect to external data sources from within the ksqlDB interface. The connector can be embedded inside the ksqlDB cluster, or you can use a standalone Kafka Connect cluster if you already have one.

You can hink of ksqlDB as a standalone SQL-powered stream processing engine that performs continuous processing of event streams and exposes the results to applications in a database-like way.

## Kafka Theory

### Topics, Partitions and Offsets

Kafka **topics** are particular streams of data within your Kafka cluster. A topic is similar to a table in a database (without all the constraints, there is no data verification).

You can have as many topics as you want. A topic is identified by its name.

Topics support any kind of message format: JSON, Avro, binary, etc.

The sequence of messages is called a data stream.

You cannot query topics. Instead, use Kafka Producers to send data and Kafka Consumers to read the data.

Topics are split into **partitions**. Messages within each partition are ordered. Each message within a partition gets an incremental id, called an **offset**.

Kafka topics are immutable. Once data is written to a partition, it cannot be changed.

For example, you could have a fleet of trucks reporting their GPS positions to Kafka every 20 seconds. You can have a topic `trucks_gps` that contains the positions of all trucks. You can choose to create a topic with an arbitrary number of partitions.

This topic could be consumed by a location dashboard to track the trucks and a notification service to notify customers.

- Once the data is written to a partition, it cannot be changed (imutability)
- Data is kept for a limited time (default 1 week but configurable)
- Offsets only have a meaning for a specific partition
  - Offsets are not reused even if previous mesages have been deleted
- Order is guaranted only within a partition, not across partitions
- Data is randomly assigned to a partition unless a key is provided

### Producers and Message Keys

Producers write data to topics (which are made of partitions).

Producers know in advance which partition to write to (and which Kafka broker has it).

In case of Kafka broker failures, Producers will automatically recover.

The load is balanced to many brokers thanks to the number of partitions.

Producers can choose to send a **key** with the message. This can be a string, number, binary, etc.

If key = null, data is sent round robin (partition 0, then 1, then 2, ...)

If key != null, then all messages for that key will always go to the same partition (hashing).

A key is typically sent if you need message ordering for a specific field, such as truck_id.

A Kafka message is made up of

- key (binary) - can be null
- value (binary) - can be null
- compression type (none, gzip, snappy, lz4, zstd)
- headers (optional) - key-value pairs
- partition + offset
- timestamp (system or user set)

#### Kafka Message Serializer

Kafka only accepts bytes as an input from producers and sends bytes out as an output to consumers.

Message serialization means transforming objects/data into bytes. It's performed on both the key and the value of the message.

You will specify which type of serializer to use depending on the data type, such as `IntegerSerializer` or `StringSerializer`.

Kafka Producers come with common serializers to perform this transformation:

- String, including JSON
- Int, Float
- Avro
- Protobuf

A Kafka Partitioner takes a record and determines which partition to send it to.

Key Hashing is the process of determining the mapping of a key to a partition.

In the default Kafka Partitioner, the keys are hashed using the murmur2 algorithm:

```Java
targetPartition = Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1)
```

### Consumers and Deserialization

Consumers read data from a topic (identified by name) - pull model.

Consumers automatically know which broker to read from.

In case of broker failures, consumers know how to recover.

Data is read in order from low to high offset **within each partition**.

Message deserialization involves transforming bytes back into objects/data.

The deserializer needs to know what format the message should be converted into.

Common deserializers include:

- String, including JSON
- Int, Float
- Avro
- Protobuf

The serialization/deserialization type must not change during a topic lifecycle. Instead, create a new topic.

### Consumer Groups and Consumer Offsets

#### Consumer Groups

All the consumers in an application read data as a consumer group.

Each consumer within a group reads from exclusive partitions.

If you have more consumers than partitions, some consumers will be inactive.

It's acceptable to have multiple consumer groups read from the same topic. And within each consumer group a consumer can read from more than one partition.

To create distinct consumer groups, use the consumer property `group.id`.

#### Consumer Offsets

Kafka stores the offsets at which a consumer group has been reading.

The committed offsets are in a Kafka topic names `__consumer_offsets`.

When a consumer in a group has processed data received from Kafka, it will periodically commit offsets. The Kafka broker will write to the `__consumer_offsets` topic rather than the group itself.

If a consumer dies, when it recovers/restarts it will be able to read back from where it left off thanks to the committed consumer offsets.

By default, Java consumers wil automatically commit offsets (at least once).

There are 3 **delivery semantics** if you choose to commit manually:

1. **at least once** (usually preferred)
   - Offsets are committed after the message is processed
   - If the processing goes wrong, the message will be read again
   - This can result in duplicate processing of messages. Make sure your processing is idempotent so processing the messages again won't impact your systems
2. **at most once**
   - Offsets are committed as soon as messages are received
   - If the processing goes wrong, some messages will be lost and won't be read again
3. **exactly once**
   - For Kafka to Kafka workflows, use the Transactional API (easy with the Kafka Streams API)
   - Fof Kafka to external systems workflows, use an idempotent consumer

### Brokers and Topics

A Kafka cluster is composed of multiple **brokers** (servers).

Each broker is identified with its ID (an integer).

Each broker contains certain topic partitions.

After connecting to any broker (called a bootstrap broker), you will be connected to the entire cluster (Kafka clients have smart mechanics for this).

A good number to get started is 3 brokers, but some big clusters have over 100 brokers.

Brokers allow for horizontal scaling, since topics and partitions can be spread across multiple brokers.

Every Kafka broker is also called a "bootstrap server". This means that you only need to connect to one broker and the Kafka clients will know how to be connected to the entire cluster.

When a Kafka client makes a connection (and metadata) request to a broker, the broker will return a list of all the brokers in the cluster. The client can then connect to the necessary broker(s).

### Topic Replication

Topics should have a replication factor greater than 1 (usually 2 or 3). This way, if a broker is down, another broker can serve the data.

So, for example, a topic with two partitions and a replication factor of 2 will have each of its partitions exist on two brokers.

At any time only one broker can be a **leader** for a given partition.

Producers can only send data to the broker that is the leader of a partition. The other brokers will replicate the data.

Therefore, each partition has one leader and multiple in-sync replicas (ISRs).

Producers can only write to the leader broker of a partition by default.

Consumers will read from the leader broker of a partition by default.

Since Kafka 2.4, it's possible to configure consumers to read from the closest replica. This may help to improve latency and also decrease network costs if using the cloud.

### Producer Acknowledgements and Topic Durability

Producers can choose to receive acknowledgements of data writes:

- `acks=0` - producer won't wait for acknowledgement (possible data loss)
- `acks=1` - producer will wait for leader acknowledgement (liomited data loss)
- `acks=all` - producer will wait for leader and replicas acknowledgement (no data loss)

For a topic replication factor of 3, topic data durability can withstand losing 2 brokers.

As a rule, for a replication factor of N, you can permanently lose up to N-1 brokers and still recover your data.

### Zookeeper

Zookeeper manages brokers, keeping a list of them.

Zookeeper helps in performing leader election for partitions.

Zookeeper sends notifications to Kafka in case of changes, such as new topics, a broker dying, a broker coming up, topics being deleted.

Kafka 2.x can't work without Zookeeper.

Kafka 3.x can work without Zookeeper (KIP-500), using Kafka Raft instead.

Kafka 4.x will not have Zookeeper.

Zookeeper by design operates with an odd number of servers (1, 3, 5, 7).

Zookeeper has a leader that writes. The rest of the servers are followers and read.

The oldest versions of Kafka (pre 0.10) used to store consumer offsets on Zookeeper. They are now stored in the internal `__consumer_offsets` topic.

If you are managing Kafka brokers, you should still use Zokeeper until Kafka 4.0 when Zookeeper-less Kafka will be production ready.

Over time, the Kafka clients and CLI have been migrated to leverage the brokers as a connection endpoint instead of Zookeeper.

Since Kafka 0.10, consumers store offsets in Kafka and must not connect to Zookeeper, as it has been deprecated as a connection endpoint.

Since Kafka 2.2, the `kafka-topics.sh` CLI command references Kafka brokers and not Zookeeper for topic management (creation, deletion, etc) and the Zookeeper CLI argument is deprecated.

All the APIs and commands that were previously leveraging Zookeeper have been migrated to use Kafka instead, so that when clusters are migrated to be without Zookeeper the change will be invisible to clients.

Zookeeper is less secure than Kafka and so Zookeeper ports should only be opened to allow traffic from Kafka brokers, and not Kafka clients.

### Kafka KRaft

In 2020, the Apache Kafka project started to work to remove the Zookeeper dependency from Kafka (KIP-500).

Zookeeper has scaling issues when clusters have > 100,000 partitions.

By removing Zookeeper, Kafka can:

- Scale to millions of partitions, and become easier to maintain and set up
- Improve stability, making it easier to monitor, support and administer
- Have a single security model for the whole system
- Start with a single process
- Shut down and recover much faster

Kafka 3.x implements the Raft protocol (KRaft) in order to replace Zookeeper.

## Starting Kafka

### Start Kafka with Zookeeper

You need to start Zookeeper prior to starting Kafka.

You can start Zookeeper with

```bash
/usr/local/bin/zookeeper-server-start /usr/local/etc/zookeeper/zoo.cfg
```

Then, in another terminal window, start Kafka with

```bash
/usr/local/bin/kafka-server-start /usr/local/etc/kafka/server.properties
```

If you close either window, it will stop Kafka.

#### Start Kafka in KRaft Mode

You can start Kafka in KRaft mode (development only)

First, generate a new ID for your cluster.

```bash
~/kafka_2.13-3.0.0/bin/kafka-storage.sh random-uuid
```

Next, format your storage directory. Use the ID you just generated.

```bash
~/kafka_2.13-3.0.0/bin/kafka-storage.sh format -t <uuid> -c ~/kafka_2.13-3.0.0/config/kraft/server.properties
```

This will format the directory in `log.dirs` in the `config/kafka/server.properties` file (by default `/tmp/kraft-combined-logs`).

Now you can launch the broker itself in daemon mode by running `kafka-server-start.sh` and passing in the properties:

```bash
~/kafka_2.13-3.0.0/bin/kafka-server-start.sh ~/kafka_2.13-3.0.0/config/kraft/server.properties
```

## Kafka CLI

The Kafka CLI comes bundled with the Kafka binaries.

If you set up the `$PATH` variable, you can invoke the CLI from anywhere on your machine.

If you installed Kafka using binaries, the command will be `kafka-topics.sh` or `kafka-topics` if using Homebrew/apt.

Use the `--bootstrap-server` option, not `--zookeeper`.

```bash
kafka-topics --bootstrap-server localhost:9092
```

### Kafka Topic Management

To list topics, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --list
```

To create a topic (with default partitions and replication factor), use:

```bash
kafka-topics --bootstrap-server localhost:9092 --create --topic TOPIC_NAME
```

To create a topic with multiple partitions, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --create --topic TOPIC_NAME --partitions 3
```

To create a topic with a replication factor higher than 1, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --create --topic TOPIC_NAME --partitions 3 --replication-factor 2
```

Note: You cannot have a higher replication factor than the number of brokers.

To describe a topic, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --topic TOPIC_NAME --describe
```

To describe all the topics in a cluster, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --describe
```

To delete a topic, use:

```bash
kafka-topics --bootstrap-server localhost:9092 --delete --topic TOPIC_NAME
```

### Kafka Console Producer CLI

You can produce to a topic from the CLI using `kafka-console-producer`.

Running `kafka-console-producer` by itself will bring up its docs.

To create a producer, run

```bash
kafka-console-producer --bootstrap-server localhost:9092 --topic TOPIC_NAME
```

You can then enter messages from the command line until you stop the `kafka-console-producer` with `ctrl + C`.

You can set different console producer properties, such as `acks=all`, by passing the `--producer-property` option.

If you produce to a topic that doesn't exist, you'll get a warning, but the topic will be created. It's better to create a topic in advance with the number of partitions you want.

By default, the values sent from the console producer have a null key and will be distributed evenly between all partitions.

To produce with keys, set `parse.key` to true and set a key seperator. If you then try to produce a message without specifying a key, you will get an exception.

```bash
kafka-console-producer --bootstrap-server localhost:9092 --topic TOPIC_NAME --property parse.key=true --property key.seperator=:
```

### Kafka Console Consumer CLI

You can consume from a topic from the CLI using `kafka-console-consumer`.

Running `kafka-console-consumer` by itself will bring up its docs.

To create a consumer, run

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic TOPIC_NAME
```

When you start a consumer, by default it will start reading messages from the end of the topic onwards. It won't read messages that were previously sent to the topic.

To read all messages from the beginning of the topic, specify `--from-beginning`

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic TOPIC_NAME --from-beginning
```

The messages will be in order **within** each partition, but may seem out of order if they were sent round robin to different partitions.

To display the key, value, and timestamp of each message, set these on the `print` property.

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic TOPIC_NAME --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.timestamp=true --property print.key=true --property print.value=true
```

### Kafka Consumers in Groups

You can run a consumer group from the command line. You can view the documentation for this by running

```bash
kafka-consumer-groups
```

If you run the following more than once with the same group name, it will create multiple consumers in the same consumer group.

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic TOPIC_NAME --group GROUP_NAME
```

Writes will be spread across the consumers in a group, up to the number of partitions.

If multiple consumer groups are set up to read from the same topic, the same message will be written to each group.

### Kafka Consumer Groups CLI

To list consumer groups, use

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --list
```

To describe a consumer group, use

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group GROUP_NAME
```

You will see

- `CURRENT-OFFSET`- how far the consumer has read into the topic
- `LOG-END-OFFSET` - the total messages in the topic
- `LAG` - the difference between these two offsets

The `CONSUMER-ID` will be specified if there is a consumer consuming the topic.

If you create a consumer without specifying a consumer group, a temporary consumer group will be created called `console-consumer-UNIQUE_ID`.

### Resetting Offsets

You can reset the offsets for a consumer group using the `--reset-offsets` options.

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --group GROUP_NAME --reset-offsets --to-earliest --execute --all-topics
```

This can be applied to all topics with `--all-topics` or one topic with `--topic`.

If you specify `--to-earliest` you will get a `NEW-OFFSET` of 0.

You cannot reset offsets if a consumer is currently running.

You can use the `--shift-by` option to reset by a specific number of messages. This can be a positive integer to go forwards, or a negative integer to go backwards.

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --group GROUP_NAME --reset-offsets --shft-by -2 --execute --topic TOPIC_NAME
```

## Kafka Programming

The official SDK for Kafka is the Java SDK.

There are community-supported SDKs for Scala, C, C++, Go, Python, JavaScript, .NET, C#, Rust, Kotlin, Haskell, Ruby and more.

You can create a Kafka Java project with either Gradle or Maven.

### Java Producer

To create a Producer, you need to:

1. create Producer Properties
2. create the Producer
3. create a ProducerRecord
4. send the data
5. flush and close the Producer

```java
public class ProducerDemo {
private static final Logger log = LoggerFactory.getLogger(ProducerDemo.class);

public static void main(String[] args) {
	log.info("Started Logging");

	Properties properties = new Properties();

	properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
	        properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
	        properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

	KafkaProducer<String, String> producer = new KafkaProducer<>(properties);

	ProducerRecord<String, String> producerRecord = new ProducerRecord<>("demo_java", "hello world");

	producer.send(producerRecord);

	producer.flush();

	producer.close();
    }
}
```

### Java API Callbacks

To confirm the partition and offset the message was sent to, you use callbacks.

```java
producer.send(producerRecord, new Callback() {
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        if (exception == null) {
            log.info("Received new metadata. \n" +
                    "Topic: " + metadata.topic() + "\n" +
                    "Partition: " + metadata.partition() + "\n" +
                    "Offset: " + metadata.offset() + "\n" +
                    "Timestamp: " + metadata.timestamp());
        } else {
            log.error("Error while producing: ", exception);
        }
    }
});
```

### Sticky Partitioner

The default behaviour of a producer is to send messages round robin between partitions.

But if you send several messages very quickly, the producer will batch messages together to the same partition to be more efficient.

If a partition or key is specified in the message, that partition will be used. If no key or partition is specified, the producer will use the sticky partition that changes once the batch is full.

### Java Producer with Keys

If you specify a key, every message with the same key will be sent to the same partition.

```java
for(int i = 0; i < 10; i++) {
    String topic = "java_demo";
    String value = "hello world " + i;
    String key = "id_" + i;

    ProducerRecord<String, String> producerRecord = new ProducerRecord<>(topic, key, value);

    producer.send(producerRecord, new Callback() {
        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (exception == null) {
                log.info("Received new metadata. \n" +
                        "Topic: " + metadata.topic() + "\n" +
                        "Key: " + producerRecord.key() + "\n" +
                        "Partition: " + metadata.partition() + "\n" +
                        "Offset: " + metadata.offset() + "\n" +
                        "Timestamp: " + metadata.timestamp());
            } else {
                log.error("Error while producing: ", exception);
            }
        }
    });
}
```

### Java Consumer

Setting up a consumer is similar to a producer.

To poll for messages from a topic, you set up an infinite loop and call the consumer's `poll()` method with a timeout duration.

If records are received during this time, you can do something with them. Either way, the consumer goes back to polling the topic.

```java
public class ConsumerDemo {
    private static final Logger log = LoggerFactory.getLogger(ConsumerDemo.class);
    public static void main(String[] args) {
        log.info("Started Logging");

        String bootstrapServers = "127.0.0.1:9092";
        String groupId = "my-second-application";
        String topic = "demo_java";

        // create consumer config
        Properties properties = new Properties();
        properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // create consumer
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);

        // subscribe to topic(s)
        consumer.subscribe(Collections.singletonList(topic));
        // consumer.subscribe(Arrays.asList(topic)); // multiple topics

        // poll for new data
        while(true){
            log.info("Polling");

            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, String> record : records) {
                log.info("Key: " + record.key() + ", Value: " + record.value());
                log.info("Partition: " + record.partition() + ", Offset: " + record.offset());
            }
        }
    }
}
```

### Consumer - Graceful Shutdown

To gracefully shut down a consumer:

1. Get a reference to the current thread
2. Add a shutdown hook
3. Call `consumer.wakeup()`
4. Join the main thread
5. Handle the expected `WakeupException`
6. Close the consumer

```java
// get a reference to the current thread
final Thread mainThread = Thread.currentThread();

// add the shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread() {
    public void run() {
        log.info("Detected a shutdown, calling consumer.wakeup()");
        consumer.wakeup();

        // join the main thread to allow the execution of the code in the main thread
        try {
            mainThread.join();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
});

try {
    // subscribe to topic(s)
    consumer.subscribe(Collections.singletonList(topic));
    // consumer.subscribe(Arrays.asList(topic)); // multiple topics

    // poll for new data    while(true){
        log.info("Polling");

        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            log.info("Key: " + record.key() + ", Value: " + record.value());
            log.info("Partition: " + record.partition() + ", Offset: " + record.offset());
        }
    }
} catch (WakeupException e) {
    // ignore this expected exception
    log.info("Wakeup exception");
} catch (Exception e) {
    log.error("Unexpected exception");
} finally {
    // gracefully close the connection, committing offsets if needed
    consumer.close();
    log.info("Consumer closed gracefully");
}
```

### Java Consumer inside a Consumer Group

If you run two instances of the same Consumer, they will both join the same consumer group and read from their respective partitions.

They will be assigned partitions and have their offsets set to the committed offsets.

If you shut down one consumer, it will leave the group and the remaining consumers will rebalance the partitions between themselves.

### Consumers - Incremental/Cooperative Rebalance

Moving partitions between consumers is called a **rebalance**.

Reassignment of partitions happens when a consumer leaves or joins a group. It also happens if an administrator adds new partitions into a topic.

The default behaviour is an **eager rebalance**.

- All consumers stop and give up their membership of partitions
- They rejoin the consumer group and get a new partition assignment
- For a short period of time the entire consumer group stops processing messages
- Consumers don't necessarily get back the same partition they used to have

Recently, Kafka added the option of a **cooperative/incremental rebalance**.

- A small subset of partitions are reassigned from one consumer to another
- Consumers that don't have reassigned partitions can still process uninterrupted
- This can go through several iterations to find a stable assignment (hence "incremental")
- This avoids "stop the world" events where all consumers stop processing data

In the Kafka Consumer there is a `partition.assignment.strategy` setting.

The eager strategies are:

- `RangeAssignor` - assigns partitions on a per-topic basis, can lead to imbalance (previous default)
- `RoundRobin` - assigns all partitions across topics in a round-robin fashion, optimal balance
- `StickyAssignor` - balanced like RoundRobin, then minimizes partition movements when consumers join/leave the group

The newer strategy is:

- `CooperativeStickyAssignor` - identical to StickyAssignor but supports cooperative rebalances so consumers can keep consuming from the topic

The default in Kafka 3 is now:

- `[RangeAssignor, CooperativeStickyAssignor]` - uses the RangeAssignor by default, but allows upgrading to the CooperativeStickyAssignor with just a single rolling bounce that removes the RangeAssignor from the list

In Kafka Connect, cooperative rebalancing is enabled by default.

In Kafka Streams, cooperative rebalancing is enabled by default using the StreamsPartitionAssignor.

#### Static Group Membership

By default, when a consumer leaves a group its partitions are revoked and reassigned.

If it rejoins, it will have a new "member ID" and new partitions will be assigned to it.

But, if you specify `group.instance.id` as part of the consumer config, it makes the consumer a **static member**.

Upon leaving, the consumer has up to `session.timeout.ms` to rejoin and get back its partitions (otherwise they will be reassigned), without triggering a rebalance.

This is helpful when consumers maintain local state and cache (to avoid rebuilding the cache).

### Consumer - Auto Offset Commit Behaviour

In the Java Consumer API when you poll regularly offsets are regularly committed.

This enables at-least-once reading scenarios by default (under certain conditions).

Offsets are committed when you call `poll()` and `auto.commit.interval.ms` has elapsed. For example, `enable.auto.commit=true` and `auto.commit.interval.ms=5000` means the consumer will commit offsets every 5 seconds.

Make sure messages are all successfully processed before you call `poll()`again. If you don't, you will not be in an at-least-once reading scenario.

In the rare case where you disable `enable.auto.commit`, you will have a seperate thread to let you commit once in a while by calling `commitSync()` or `commitAsync()` (advanced).

### Producer Acknowledgements (acks)

Producers can choose to receive acknowledgements of data writes to brokers:

- `acks=0` - producer won't wait for acknowledgement (possible data loss)
- `acks=1` - producer will wait for leader acknowledgement (liomited data loss)
- `acks=all` - producer will wait for leader and replicas acknowledgement (no data loss)

#### acks=0

When `acks=0` producers consider messages as "written successfully" the moment the message was sent, without waiting for the broker to accept it.

If the broker goes offline or an exception happens, you won't know and will lose data.

This is still useful where it's ok to potentially lose messages, such as metrics collection.

This produces the highest thoughput setting because the network overhead is minimal.

#### acks=1

When `acks=1`, producers consider messages as "written successfully" when the message was acknowledged only by the leader broker.

This was the default from Kafka 1.0 - 2.8.

Leader response is requested but replication is not a guarantee as it happens in the background.

If the leader broker goes offline unexpectedly and replicas haven't yet replicated the data, you will have data loss.

If an ack is not received, the producer may retry the request.

#### acks=all (acks=-1)

When `acks=all`, producers consider messages as "written successfully" when the message is accepted by all in-sync replicas (ISRs).

This is the default for Kafka 3.0+.

`acks=all` goes with the setting `min.insync.replicas`.

The leader replica for a partition checks to see if there are enough in-sync replicas to safely write the message (controlled by the broker setting `min.insync.replicas`).

- `min.insync.replicas=1` - only the broker leader needs to successfully ack (default)
- `min.insync.replicas=2` - at least the broker leader and one replica need to ack (recommended)

If there are not enough replicas available, the leader broker will respond with this information to the producer and trigger an exception.

It's generally preferable to not accept writes than to risk losing data.

### Kafka Topic Availability

Availability (considering a replication factor of 3)

- `acks=0` or `acks=1` - if one partition is up and considered an ISR, the topic will be available for writes
- `acks=all`
  - `min.insync.replicas=1` (default) - the topic must have at least one partition up as an ISR (that includes the leader) and so you can tolerate two brokers being down
  - `min.insync.replicas=2` - the topic must have at least 2 ISR up, and so you can tolerate at most one broker being down (in the case of a replication factor of 3), and you have the guarantee that for every write the data will be written at least twice

It wouldn't make sense to set a `min.insync.replicas` to the same as the replication factor (such as 3 and 3), since you couldn't tolerate any broker going down.

When `acks=all`, with a `replication.factor=N` and `min.insync.replicas=M` you can tolerate N-M brokers going down for topic availability purposes.

`acks=all` and `min.insync.replicas=2` is the most popular combination for data durability and availability and allows you to withstand at most the loss of **one** broker.

### Producer Retries

In case of transient failures, developers are expected to ahndle exceptions, otherwise data will be lost.

Transient failures could include `NOT_ENOUGH_REPLICAS` due to the `min.insync.replicas` setting.

There is a "retries" setting:

- defaults to 0 for Kafka <= 2.0
- defaults to 2,147,483,647 for Kafka >= 2.1

The `retry.backoff.ms` setting is 100ms by default.

### Producer Timeouts

If retries > 0, for example 2,147,483,647, retries are bounded by a timeout.

Since Kafka 2.1 you can set `delivery.timeout.ms` (defaults to 120,000ms or 2 mins).

If you are not using an idempotent producer (not recommended - old Kafka):

- In case of retries, there is a chance that messages will be sent out of order (if a batch has failed to be sent)
- If you rely on key-based ordering, that can be an issue

For this, you can set the settings which control how many producer requests can be made in parallel: `max.in.flight.requests.per.connection`

- Defaults to 5
- Set it to 1 if you need to ensure ordering (may impact throughput)

In Kafka >= 1.0 idempotent producers are a better solution.

### Idempotent Producers

The producer can introduce duplicate messages in Kafka due to network errors.

In Kafka >= 0.11, you can define an idempotent producer which won't introduce duplicates on network error. Kafka will detect the duplicate message and won't commit it twice.

Idempotent producers are a must to guarantee a stable and safe pipeline. They are the default since Kafka 3.0, and are strongly recommended.

They come with:

- `retries=Integer.MAX_VALUE` (2,147,483,647)
- `max.in.flight.requests=1` (Kafka <= 0.11) or
- `max.in.flight.requests=5` (Kafka >= 1.0 - higher performance while ordering kept, KAFKA-5494)
- `acks=all`

These settings are applied automatically after your producer has started if not already manually set.

```java
properties.setProperty(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
```

### Safe Producer Settings

Since Kafka 3.0 the producer is "safe" by default:

- `acks=all`
- `enable.idempotence=true`

With Kafka 2.8 and lower, the producer by default comes with:

- `acks=1`
- `enable.idempotence=false`

Use a safe producer whenever possible and always use upgraded Kafka Clients.

If you're not yet using Kafka 3.0+, at least set the following:

- `acks=all`
- `min.insync.replicas=2`
- `enable.idempotence=true`
- `retries=MAX_INT`
- `delivery.timeout.ms=120000`
- `max.in.flight.requests.per.connection=5`

```java
// set safe producer configs (Kafka <= 2.8)
properties.setProperty(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
properties.setProperty(ProducerConfig.ACKS_CONFIG, "all"); // same as setting -1
properties.setProperty(ProducerConfig.RETRIES_CONFIG, Integer.toString(Integer.MAX_VALUE));
```

### Kafka Message Compression

Producers usually send text-based data, such as JSON.

In this case, it's important to apply compression to the producer to enable faster transmission and more efficient storage on disk.

Compression can be enabled at the Producer level and doesn't require any configuration change in the Brokers or Consumers.

`compression.type` can be `none` (default), `gzip`, `lz4`, `snappy`, zstd (Kafka 2.1).

Compression is more effective the bigger the batch of messages being sent to Kafka.

The compressed message batch has the following advantages:

- Much smaller producer request size (compression ration up to 4x)
- Faster to transfer data over the network -> less latency
- Better throughput
- Better disk utilization in Kafka (stored messages on disk are smaller)

Disadvantages are minor:

- Producers must commit some CPU cycles to compression
- Consumers must commit some CPU cycles to decompression

Consider `snappy` or `lz4` for optimal speed/compression ratio.

Consider tweaking `linger.ms` and `batch.size` to have bigger batches, and therefore more compression and higher throughput.

Always enable compression in production.

Compression can also happen at the broker (all topics) or topic level.

`compression.type=producer` (default) - the broker takes the compressed batch from the producer client and writes it directly to the topic's log file without recompressing the data.

`compression.type=none` - all batches are decompressed by the broker (inefficient)

`compression.type=lz4` (for example): - If it matches the producer's settings, data is stored on disk as is - If it's a different compression setting, batches are decompressed by the broker and then recompressed using the compression algorithm specified

Warning: if you enable broker-side compression, it will consume extra CPU cycles

### `linger.ms` and `batch.size` Producer Settings

By default, Kafka producers try to send records as soon as possible.

It will have up to `max.in.flight.requests.per.connection=5`, meaning up to 5 message batches being in flight (being sent between the producer in the broker) at most.

After this, if more messages must be sent while others are in flight, Kafka is smart and will start batching them before the next batch is sent.

This smart batching helps increase throughput while maintaining low latency. An added benefit is that batches have a higher compression ratio and so better efficiency.

Two settings to influence the batching mechanism are:

- `linger.ms` (default 0) - how long to wait until you send a batch. Adding a small number, for example 5ms, helps add more messages in the batch at the expense of latency
- `batch.size` - if a batch is filled before `linger.ms` it wil be sent. You can increase the batch size if you want larger batches

The default batch size is 16KB, which represents the maximum number of bytes that will be included in a batch.

Increasing batch size to 32 or 64KB can help increase the compression, throughput, and efficiency of requests.

Any message that is bigger than the batch size will not be batched and will be sent immediately.

A batch is allocated per partition, so make sure you don't set it to a number that's too high, otherwise you'll waste memory.

Note: You can monitor the average batch size using Kafka Producer Metrics.

To get a high thoughput Producer:

- Increase `linger.ms` and the producer will wait a few milliseconds for the batches to fill up before sending them
- If you are sending full batches and have memory to spare, you can increase `batch.size` and send larger batches
- Introduce some producer-level compression for more efficiency when sending

```java
// set high throughput configs
properties.setProperty(ProducerConfig.LINGER_MS_CONFIG, "20");
properties.setProperty(ProducerConfig.BATCH_SIZE_CONFIG, Integer.toString(32 * 1024));
properties.setProperty(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
```

### Default Partitioner and Sticky Partitioner

When a key is not null, the record goes through partitioner logic which decides how a record gets assigned to a partition.

This process of determining the mapping of a key to a partition is called key mapping.

In the default Kafka partitioner, the keys are hashed using the murmur2 algorithm.

```Java
targetPartition = Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1)
```

The same key will go to the same partition. Adding partitions to a topic will completely alter the formula.

It's preferable not to override the behaviour of the partitioner (instead create a new topic), but it's possible using `partitioner.class`.

When key is null, the producer has a default partitioner that varies:

- Round Robin - Kafka 2.3 and below
- Sticky Partitioner - for Kafka 2.4 and above

Sticky Partitoner improves the performance of the producer, expecially when throughput is high and the key is null.

With Kafka <= 2.3, when there's no partition and no key specified, the default partitioner sends data in a round-robin fashion. This results in more batches (one per partition) and smaller batches (1 message per batch). Smaller batches lead to more requests as well as higher latency.

From Kafka 2.4+ Sticky Partitioner is the default partitioner, with significant performance improvements. The producer sticks to a partition until the batch is full of `linger.ms` has elapsed. After sending the batch, the partition that is sticky changes. This leads to larger batches and reduced latency. Over time, records are still spread evenly across partitions.

### `max.block.ms` and `buffer.memory`

If the producer produces faster than the broker can take, the records will be buffered in memory.

`buffer.memory=33554432` - 32MB (the size of the send buffer)

That buffer will fill up over time and empty back down when the throughput to the broker increases.

If the buffer is full (32MB), the `send()` method will start to block and won't return immediately.

`max.block=60000` - the time the `send()` method will block before throwing an exception.

Exceptions are thrown when:

- The producer has filled its buffer
- The broker is not accepting any new data
- 60 seconds has elapsed

If you hit an exception, it usually means your brokers are down or overloaded and they can't respond to requests.

### Consumer Delivery Semantics

**At Most Once** - Offsets are committed as soon as the message batch is received. If the processing goes wrong, the message will be lost (it won't be read again).

**At Least Once** - Offsets are committed after the message is processed. If the processing goes wrong, the message will be read again.This can result in duplicate processing of messages. Make sure your processing is **idempotent**, so that processing messages again won't impact your system.

**Exactly Once** - Can be achieved for Kafka => Kafka workflowsusing Transactional API (easy with Kafka Streams API). For Kafka => sink workflows, use an idempotent consumer.

For most applications you should use at least once processing and ensure your transformations/processing are idempotent, for example always checking the id of the record.

If the message content itself doesn't contain a unique id, you can create one by combining the Kafka topic, partition and offset values, which together will be unique.

### Consumer Offset Commit Strategies

There are two common patterns forcommitting offsets in a consumer application:

- `enable.auto.commit=true` and synchronous processing of batches (easy)
- `enable.auto.commit=false` and manual commit of offsets (medium)

In the Java Consumer API, offsets are regularly committed. At-least-once reading scenariosare enabled by default.

Offsets are committed when you call `poll()` and `auto.commit.interval.ms` has elapsed.

You need to make sure all messages are successfully processed before you call `poll()` again. If you don't, you will not be in an at-least-once reading scenario. In that rare case, you must disable `enable.auto.commit` and from time to time call `commitSync()` or `commitAsync()` with the correct offsets manually (advanced).

With auto-commit enabled, offsets will be committed automatically for you at regular intervals every time you call `poll()`.

If you don't use synchronous processing, you will be in at-most-once behaviour because offsets will be committed before your data is processed.

```java
while (true) {
	List<Records> batch = consumer.poll(Duration.ofMillis(100));
	doSomethingSynchronous(batch);
}
```

If you want to disable auto-commit and stilldo synchronous processing of batches, you control when you commit ooffsets and the condition for committing them. For example, accumulating records into a buffer and then flushing the buffer to a database and then committing offsets asynchronously.

```java
while (true) {
	batch += consumer.poll(Duration.ofMillis(100));
	if (isReady(batch)) {
		doSonethingSynchronous(batch);
		consumer.commitAsync();
	}
}
```

Another option is to disable auto-commit and store offsets externally (avanced).

- You need to assign partitions to your consumer at launch manually using the `seek()`API.
- You need to model and store your offsets in a database table for example
- You need to handle the cases where rebalances happen (ConsumerRebalanceListener interface)

For example, if you need exactly-once processing and can't find a way to do idempotent processing, then you "process data" and "commit offsets" as part of a single transaction with the target database.

### Consumer Offset Reset Behaviour

A consumer is expected to read from a log continuously.

Your consumer can however go down.

By default, Kafka has a retention perod of 7 days. If your consumer is down for more than 7 days, the offsets are "invalid".

The behaviour for the consumer is to then use:

- `auto-offset.reset=latest` - will read from the end of the log
- `auto-offset.reset=earliest` - will read from the start of the log
- `auto-offset.reset=latest` - will throw an exception if no offset is found

Additionally, consumer offsets can be lost:

- If a consumer hasn't read new data in 1 day (Kafka < 2.0)
- If a consumer hasn't read new data in 7 days (Kafka >= 2.0)

This can be controlled by the broker setting `offset.retention.minutes`.

To replay data for a consumer group:

- Take all the consumers from a specific group down
- Use the `kafka-consumer-groups` command to set the offset to what you want
- Restart the consumers

Summary:

- Set proper data retention period and offset retention period
- Ensure the data offset reset behaviour is what you expect/want
- Use replay capability in case of unexpected behaviour

### Consumer Internal Threads

Consumers in a group talk to a Consumer Group Coordinator broker.

To detect consumers that are down, there is a "heartbeat" mechanism and a "poll" mechanism.

The heartbeat thread is the consumers sending periodic messages to the coordinator broker to tell it they're still alive.

`heartbeat.intervals.ms `(defaults 3 seconds)

- Controls how often to send heartbeats
- Usually set to 1/3 of `session.timeout.ms`

`session.timeout.ms` (default 45 seconds Kafka 3.0+, previously 10 seconds)

- Heartbeats are periodically sent to the broker
- If no heartbeat is sent during this period, the consumer is considered dead
- You can set the timeout lower for faster consumer rebalances

The poll thread is what is requesting data from the various brokers. This is used to detect a data processing issue with a "stuck" consumer.

`max.poll.interval.ms` (default 5 mnutes)

- Maximum amount of time between two `poll()` calls before declaring the consumer dead
- This is relevant for Big Data frameworks like Spark in case the processing takes time

`max.poll.records` (default 500)

- Controls how many records to receive per poll request
- Increase it if your messages are very small and have a lot of available RAM
- Good to monitor how many records are polled per request
- Lower it if it takes you too much time to process records

To avoid issues, consumers are encouraged to process data fast and poll often.

`fetch.min.bytes` (default 1)

- Controls how much data (at least) you want to pull on each request
- Can help improve throughput by decreasing the number of requests at the cost of latency

`fetch.max.wait.ms` (default 500)

- The maximum amount of time the Kafka broker will block before answering the fetch request if there isn't sufficient data to immediately satisfy the requirement given by `fetch.min.bytes`
- This means that until the requirement of `fetch.min.bytes` is satisfied, you will have up to 500ms of latency before the fetch request returns data to the consumer, introducing a potential delay in order to be more efficient in requests

`max.partition.fetch.bytes` (default 1MB)

- The maximum amount of data per partition the server will return
- If you read from 100 partitions, you'll need a lot of RAM

`fetch.max.bytes` (default 55MB)

- The maximum data returned for each fetch request
- If you have available memory, try increasing `fetch.max.bytes` to allow the consumer to read more data in each request

These are advanced settings. Change them only if your consumer already maxes out on throughout.

### Consumer Replica Fetching - Rack Awareness

Consumers by default will read from the leader broker for a partition.

If you have brokers and consumers spread across multiple data centers, you may have higher latency and need to pay network charges.

Since Kafka 2.4, it's possible to configure consumers to read from the closest replica. This can improve latency and reduce network costs if using a cloud service provider.

Consumer rack awareness requires the following:
Broker settings:

- Must be Kafka 2.4+
- `rack.id` must be set to the data centre ID (for example, `rack.id=usw2-az1` in AWS)
- replica.selector.class must be set to `org.apache.kafka.common.replica.RackAwareReplicaSelector`
  Consumer settings:
- Set `client.rack` to the data center ID the consumer is launched on

## Kafka Extended APIs

Kafka Consumers and Producers have exiisted for a long time and are considered relatvely low-level.

Kafka and its ecosystem have introduced some new higher-level APIs to solve specific sets of problems:

- Kafka Connect solves External Source => Kafka and Kafka => External Sink
- Kafka Streams solves transformations Kafka => Kafka
- Schema Registry helps with using Schemas in Kafka

### Kafka Connect

Many Kafka applications need to import data from the same sources (common databases, messaging queues, IOT devices) and store dat in the same sinks (common databases, S3, ElasticSearch).

You create a Connect cluster (containing connect workers) to ingest data from a source and send it to your Kafka cluster. The same connect workers can be used to deploy sink connectors to read data from your Kafka cluster and send it to a sink.

Source Connectors get data from commopn data sources.

Sink Connectors publish data to common data stores.

This makes it easier for developers to get their data reliably into/out of Kafka, becoming part of your ETL pipeline.

This makes it easier to scale from small pipelines to company-wide pipelines by adding compute capability to the Connect cluster.

There is an ecosystem of reusable code.

Connectors achieve fault tolerance, idempotence, distribution, and ordering.

### Kafka Streams

Kafka Streams is an easy data processing and transformation library within Kafka.

For example, it can perform:

- data transformations
- data enrichment
- fraud detection
- monitoring and alerting

A Kafka Streams application is written as a standard Java application. You don't need to create a seperate cluster. It's highly scalable, elastic and fault tolerant.

It provides exactly-once capabilities since it is a Kafka to Kafka workflow. It processes one record at a time, with no batching.

The Kafka Streams application reads from a topic, performs its calculations, and outputs the resulting data to other topics.

### Kafka Schema Registry

One reason that Kafka is efficient is that it takes bytes as an input, sends bytes as an output, and carries out no data verification.

But what if:

- the producer sends bad data?
- a field gets renamed?
- the data format changes?

The Consumers will break because they won't be able to deserialize messages.

You need data to be self describable and able to evolve over time without breaking downstream consumers. This requires schemas and a schema registry.

Having Kafka brokers verify the messages they receive would break much of what makes Kafka good.

- Kafka doesn't parse or even read your data (no CPU overhead)
- Kafka takes bytes as an input without even loading them into memory (zero copy)
- Kafka distributes bytes
- As far as Kafka is concerned, it doesn't care if your data is an integer, string, or something else

The Schema Registry needs to be a seperate component and Producers and Consumers need to be able to talk to it.

The Schema Registry needs to be able to reject bad data before it is sent to Kafka.

A common data format must to be agreed upon:

- it needs to support schemas
- it needs to support schema evolution
- it needs to be lightweight

Apache Avro is the default data format used but Protobuf and JSON Schema are also supported.

The Schema Registry will:

- Store and retrieve Schemas for Producers and Consumers
- Enforce backwards/forwards/full compatability on topics
- Decrease the size of the payload sent to Kafka

Using a Schema Registry has a lot of benefits, but you need to:

- Set it up properly
- Make sure it's highly available
- Change the Producer and Consumer code

Apache Avro as a format has a learning curve.

The Confluent Schema Registry is free and source-available.

## Kafka Real-World Architectures

### Partition Count and Replication Factor

Partition count and replication factor are the two most important factors when creating a topic. They impact performance and the durability of the overall system.

It's best to get these parameters right the first time.

- If the partition count increases during a topic lifecycle, you will break your key ordering guarantees.
- If the replication factor increases during a topic lifecycle, you put more pressure on your cluster, which can lead to unexpected performance decreases

#### Partition Count

Each partition can handle a throughput of a few MB/s (measure it for your setup)

More partitions implies:

- Better parallelism, better throughput
- Ability to run more consumers in a group to scale (max as many consumers per group as partitions)
- Ability to leverage more brokers if you have a large cluster
- But, more elections to perform for Zookeeper (if using Zookeeper)
- But, more files open on Kafka

**Guidelines**: Partitions per topic:

- Small cluster (<6 brokers): 3 x # brokers
- Bigger cluster (> 12 brokers): 2 x # brokers
- Adjust for number of consumers you need to run in parallel at peak throughput
- Adjust for producer throughput (increase if super high throughput or projected increase in the next two years)

Don't systematically create topics with 1000 partitions.

Every Kafka cluster will have different performance, so **test**.

#### Replication Factor

Replication factor should be at least 2, usually 3, max 4.

The higher the replication factor (N):

- Better durability of your system (N-1 brokers can fail)
- Better availability of your system (N-min.insync.replicas if producer acks=all)
- But, more replication means higher latency if acks=all
- But, more disk space on your system (50% more if RF is 3 instead of 2)

Guidelines: Replication factor:

- Set to 3 to get started (must have at least 3 brokers)
- If replication performance is an issue, get a better broker instead of less RF
- Never set to 1 in production

#### Cluster Guidelines

Total number of partitions in the cluster:

- Kafka with Zookeeper: max 200,000 partitions - Zookepper scaling limit
  - Maximum of 4000 partitions per broker recommended (soft limit)
- Kafka with KRaft: potentially millions of partitions

If you need more partitions in your cluster, add brokers instead.

If you need more than 200,000 partitions in your cluster (which will take time to get there), follow the Netflix model and create more Kafka clusters.

Overall, you don't need a topic with 1000 partitions to achieve high throughput. Start at a reasonable number, test the performance, and go from there.

### Topic Naming Conventions

Naming a topic is a free-for-all but it's better to enforce guidelines in your cluster to ease management. You are free to come up with your own guidelines.

For example:

```
message_type.dataset_name.data_name.data_format
```

Message type:

- logging - for logging data (slf4j, syslog)
- queuing - for classic queuing use cases
- tracking - for tracking events suhc as user clicks, page views, ad views
- etl/db - for ETL and CDCX uses cases such as database feeds
- streaming - for intermediate topics created by stream processing pipelines
- push - for data that's being pushed from offline (batch computation) environments into online environments
- user - for user-specific data such as scratch and test topics

The dataset name is analogous to a database name in traditional RDBMS systems. It's used as a category to group topics together.

The data name is analogous to a table name in traditional RDBMS systems, though it's fine to include further dotted notation if developers wish to impose their own hierarchy within the dataset namespace.

The data format, for example .avro, .json, .txt, .protobuf, .csv, .log.

Use snake case.

### Case Studies

#### Media Streaming App

Media streaming app with the following capabilities:

- Make sure the user can resume the video where they left off
- Build a user profile in real time
- Recommend the next show to the user in real time
- Store all the data in the analytics store

Video player (while playing) -> Video position service (provider) -> Show position topic (Kafka) -> Resuming service (consumer) -> Video player (while starting).

A real time Recommendation Engine (Kafka Streams) could use show position to inform recommendations. The recommendations could be consumed by a Recommendations service which sends data to the client app.

An Analytics consumer (Kafka Connect) could read from the show position and recommendations topic and send the data to an analytics store for further processing (such as Hadoop).

show_position topic:

- Can have multiple producers
- Should be highly distributed if high volume > 30 partitions
- Key could be user_id

recommendations topic:

- The Kafka Streams recommendation engine may source data from the analytical store for historical training
- May be a low volume topic
- Key could be user_id

#### Taxi App

An on-demand taxi service with the following capabilities:

- The user should match with a close by driver
- The pricing should surge if the number of drivers is low or the number of users is high
- All the position data before and during the trip should be stored in an analytics store so that the cost can be computed accurately

User app -> User Position service (producer) -> user_position topic (Kafka)

Taxi Driver app -> Taxi Position service (producer) -> taxi_position topic (Kafka)

A Surge Pricing computational model (Kafka Streams) can use user_position data and taxi_position data to calculate prices and output to a surge_pricing topic.

The surge_pricing topic can feed information to a Taxi Cost service (consumer) which updates the user app.

An Analytics consumer (Kafka Connect) can read data from the user_position, taxi_position, and surge_pricing topics and store it in an Analytics Store (such as S3).

taxi_position and user_position topics:

- Can have multiple producers
- Should be highly distributed if high volume > 30 partitions
- Key could be user_id and taxi_id respectively
- Data is quite ephemeral and soesn't need to be kept for a long time

surge_pricing topic:

- The computation of surge pricing comes from the Kafka Streams application
- Surge pricing may be regional and therefore that topic may be high volume
- Other topics such as weather or events can be included in this Kafka Streams application

#### Social Media App

A social media app with the following capabilities:

- Users should be able to post, like and comment
- Users should see the total number of likes and comments per post in real time
- A high volume of data is expected on launch day
- Users should be able to see trending posts

Seperate topics for posts, likes, comments, and posts_with_counts.

User posts -> Posting Service (producer) -> posts topic (Kafka)

User likes/comments -> Like/Comment Service (producer) -> likes and comments topics (Kafka)

The computation of total likes and comments per post can be handled in Kafka Streams outputting to a posts_with_counts topic.

Trending posts in the past hour could be calculated by a seperate Kafka Streams application.

A Refresh Feed Service (consumer) and Trending Feed Service (consumer) could consum from the posts_with_counts and trending_posts topics before updating the client app.

This follows Command Query Responsibility Segregation (CQRS) by decoupling commands from queries.

posts topic:

- Can have multiple producers
- Should be highly distributed if high volume > 30 partitions
- Key could be user_id
- Probably want a high retention period for dat in this topic

likes, comments topics:

- Can have multiple producers
- Should be highly distributed as the volume of data will be much greater
- Key could be post_id

The data itself in Kafka should be formatted as events:

- User_123 created post_456 at 2pm
- User_234 liked post_456 at 3pm
- User_123 deleted post_456 at 6pm

### Banking App

A digital bank that allows real-time banking for users. It wants to deploy a new capability to alert users in the event of large transactions.

- The transaction data already exists in a database
- Thresholds can be defined by users
- Alerts must be sent in real time to users

Transactions Database -> Kafka Connect Source CDC (Change Data Capture) Connector -> bank_transactions topic (Kafka)

Users set thresholds in app -> App Threshold Service (producer) -> user_settings topic (Kafka)

A Kafka Streams application could be used to detect large transactions in real time, reading from the bank_transactions and user_settings topics and writing to a user_alerts topic.

A Notifcation Service (consumer) could read from the user_alerts topic and notify users via the client app.

bank_transactions topic:

- Kafka Connect Source is a great way to expose data from existing databases
- There are many CDC (change data capture) connectors for popular databases, such as PostgreSQL, Oracle, MySQL, SQLServer, MongoDB

Kafka Streams application:

- When a user changes their settings, alerts won't be triggered for past transactions

user_settings topic:

- It's better to send events to the topic (user_123 enabled threshold $1000 at 12pm 12 July 2022) rather than sending the state of the user (user_123, threshold $1000)

### Big Data Ingestion

Kafka was originally created to handle Big Data ingestion.

It's comon to have generic connectors or solutions to offload data from Kafka to stores such as HDFS, S3, or ElasticSearch.

It's also common to have Kafka server as a speed layer for real time applications, while having a slow layer which helps with data ingestion into stores for later analysis.

Kafka as a front to Big Data ingestion is a common pattern in Big Data to provide an ingestion buffer in front of some stores.

Kafka can write directly to real-time tools such as Spark or FLink, for realt-ime analytics and alerts. Or via Kafka Connect to batching processing with Hadoop, S3, RDBMS for data science, reporting, auditing, long term storage.

### Logging and Metrics Aggregation

One of the first use cases for Kafka was to ingest logs and metrics for various applications.

This kind of deployment usually wants high throughput, and has less restrictions regarding data loss, replication etc.

Application logs can end up in logging solutions such as Splunk, CloudWatch, ELK

App -> Log forwarder (producer) -> application_logs topic (Kafka) -> Kafka Connect Sink -> Splunk

## Enterprise Kafka

### Kafka Cluster Setup - High Level Architecture

You want multiple brokers in different data centers (racks) to distribute your load.

You also want a cluster of at least 3 Zookeeper instances (if using Zookeeper).

For example:

| us-east-1a     | us-east-1b     | us-east-1c     |
| -------------- | -------------- | -------------- |
| Zookeeper 1    | Zookeeper 2    | Zookeeper 3    |
| Kafka Broker 1 | Kafka Broker 2 | Kafka Broker 3 |
| Kafka Broker 4 | Kafka Broker 5 | Kafka Broker 6 |

It's not easy to set up a cluster:

- You want to isolate each Zookeeper and Broker on seperate servers
- You need to implement monitoring
- You need to master operations
- You need an excellent Kafka admin

This is why there are several managed Kafka as a Service offerings from various companies, such as AWS MSK, Confluent Cloud, Aiven, CloudKarafka, Instaclustr, Upstash. This removes operational burdens in terms of setup, updates, and monitoring.

To know how many brokers you need in your cluster, you need to compute your throughput, data retention, and replication factor, then test for your use case.

Other components that need to be properly set up:

- Kafka Connect Clusters
- Kafka Schema Registry (run 2 for high availability)
- UI tools for ease of administration
- Admin tools for automated workflows

Automate as much of your infrastructure as possible once you've understood how processes work.

### Kafka Monitoring and Operations

Kafka exposes metrics through JMX.

These metrics are highly important for monitoring Kafka and ensuring the systems are behaving correctly under load.

Common places to host Kafka metrics include:

- ELK (ElasticSearch + Kibana)
- Datadog
- NewRelic
- Confluent Control Centre
- Prometheus

Some of the most important metrics are:

- **Under Replicated Partitions** - The number of partitions that are having problems with ISRs (in-sync replicas). May indicate a high load on the system
- **Request Handlers** - Utilization of threads for IO, network, overall utilization of a Kafka broker
- **Request Timing** - How long it takes to reply to requests. Lower is better, as latency will be improved.

Kafka operations teams need to be able to perform:

- Rolling restart of brokers
- Updating configurations
- Rebalancing partitions
- Increasing/decreasing replication factor
- Adding, removing, replacing a broker
- Upgrading a cluster without downtime

Managing your own cluster comes with all these responsibilities and more.

### Kafka Security

Without security:

- Any client can access your Kafka cluster
- Clients can publish to or consume from any topic
- All the data being sent is fully visible on the network

Someone could:

- Intercept data being sent
- publish bad data
- Steal data
- Delete topics

Encryption in Kafka ensures that the data exchanged between clients and brokers is secret to routers along the way, similar to HTTPS.

You can enable SSL-based encryption and expose a port on the Kafka broker for encrypted data.

There is a negliglbe performance cost if you are using Java JDK 11+.

Authentication in Kafka ensures that only clients that can prove their identity can connect to the Kafka cluster. Encrypted authentication data is sent by the Kafka Client and authenticated by the Kafka broker.

- SSL Authentication - clients authenticate to Kafka using SSL certificates
- SASL/Plaintext:
  - clients authenticate using username/password (weak but easy to set up)
  - Must enable SSL encryption broker side as well
  - Changes in passwords require broker reboot (good for dev only)
- SASL/SCRAM
  - Username/password with a challenge (salt), more secure
  - Must enable SSL encryption broker-side as well
  - Authentication data is in Zookeeper (until removed) - add without restarting brokers
- SASL/GSSAPI (Kerberos)
  - Works with Kerberos, such as Microsoft Active Directory (strong but hard to set up)
  - Very secure and enterprise friendly
- SASL/OAUTHBEARER
  - Leverages OAuth2 tokens for authentication

Once a client is authenticated, Kafka can verify its identity. It still needs to be combined with authorisation so that Kafka knows which client can do what.

Access Control Lists (ACL) must be maintained by admins to onboard new users.

The best support for Kafka security is with the Java clients, although other clients based on librdkafka now have good support for security.

### Kafka Multicluster and Replication

Kafka can only operate well in a single region.

It's common for enterprises to have Kafka clusters across the world, with some level of replication between them.

A replication application at its core is just a consumer and a producer.

There are different tools to perform replication:

- Mirror Maker 2 - open-source Kafka Connector that ships with Kafka
- Netflix uses Flink - they wrote their own application
- Uber uses uReplicator - addresses performance and operations issues with Mirror Maker 1
- Comcast has their own open-source Kafka Connect Source
- Confluent has their own Kafka Connect Source (paid)

Replication doesn't preserve offsets, just data. Data at an offset in one cluster is not the same as the data at the same offset in another cluster.

#### Active/Active Replication

Active/Active Replication would be two Kafka clusters where both clusters can be written to with two way replication between the clusters.

Advantages:

- Ability to serve users from a nearby data center, typically with performance benefits
- Redundency and resilience. Since every data center has all its functionality, if one is unavailable, you can direct users to the other.

Disadvantages

- It's challenging to avoid conflicts when data is read and updated asynchronously in multiple locations.

#### Active/Passive Replication

Active/Active Replication would be two Kafka clusters where producers are only producing to the active cluster, which is being replicated to another (passive) cluster. Consumers can read from either cluster.

Advantages:

- Simplicity in setup and suitability for pretty much any use case
- No need to worry about access to data, handling conflicts, and other architectural complexities
- Good for cloud migrations

Disadvantages:

- Waste of good cluster
- It's not currently possible to perform cluster failover in Kafka without either losing data or having duplicate events

### Advertised Listeners - Communication between Client and Kafka

Advertised Listeners are some of the most important settings in Kafka.

A Kafka client can initially access a broker using its public IP address but needs its Advertised Listener address to continue communicating.

If the Kafka Client is on the same private network, this will work. Otherwise, it won't be able to find the IP.

If the Advertised Listener is the same as the public IP, things will work, but after a reboot the Kafka public IP will change. The client will not be able to find a network route.

If your clients are on your private network, set `advertised.listeners` to either:

- the internal private IP
- the internal private DNS hostname
  Your clients should be able to resolve the internal IP or hostname.

If your clients are on a public network, set `advertised.listeners` to either:

- The external public IP
- The external public hostname pointing to the public IP
  Your clients must be able to resolve the public hostname

## Advanced Kafka

### Topic Configurations

Brokers have defaults for all topic configuration parameters. These parameters impact performance and topic behaviour.

Some topics may need different values than the defaults, such as:

- Replication factor
- Number of partitions
- Message size
- Compression level
- Log cleanup policy
- Min insync replicas

Run `kafka-configs` to view the available configuration options.

To view the dynamic configs for a topic, run:

```bash
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name TOPIC_NAME --describe
```

You can alter a topic's configuration using the `--alter` and `--add-config` options.

```bash
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name TOPIC_NAME --alter --add-config min.insync.replicas=2
```

You can remove a topic's dynamic configuration using the `--alter` and `--delete-config` options.

```bash
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name TOPIC_NAME --alter --delete-config min.insync.replicas
```

### Partitions and Segments

Partitions are made of segments (files). Each segment will have a range of offsets.

The last segment is the active segment and is being written to. There is only one active segment at any one time.

Two important segment settings:

- `log.segment.bytes` - the max size of a single segment in bytes (default 1GB)
- `log.segment.ms` - the time Kafka will wait before committing the segment if not full (default 1 week)

Segments come with two indexes (files):

- An offset to position index, which helps Kafka find where to read from to find a message
- A timestamp to offset index, which helps Kafka find messages with a specific timestamp

A smaller `log.segment.bytes` means:

- More segments per partition
- Log Compaction happens more often
- But, Kafka must keep more files opened (may encounter: Error: Too many open files)

Ask yourself how fast will I have new segments based on throughput.

A smaller `log.segment.ms` means:

- You set a max frequency for log compaction (more frequent triggers)
- Maybe you want daily compaction instead of weekly

Ask yourself how often do I need log compaction to happen?

### Log Cleanup Policies

Many Kafka clusters make data expire, according to a policy called log cleanup.

Policy 1: `log.cleanup.policy=delete` (Kafka default for al user topics)

- Delete based on age of data (default is a week)
- Delete based on max size of log (default is -1 == infinite)

Policy 2: `log.cleanup.policy=compact` (Kafka default for topic \_\_consumer_offsets)

- Delete based on keys of your messages
- Will delete old duplicate keys **after** the active segment is committed
- Infinite time and space retention

Deleting data from Kafka allows you to:

- Control the size of the data on the disk
- This limits maintenance work on the cluster if data size is stable

How often does log cleanup happen?

- Log cleanup happens when partition segments are created
- Smaller/more segments means that log cleanup will happen more often
- Log cleanup shouldn't happen too often as it takes CPU and RAM resources
- The cleaner checks for work every 15 seconds (`log.cleaner.backoff.ms`)

`log.retention.hours`:

- Number of hours to keep data for (default is 168 - one week)
- Higher number means more disk space
- Lower number means less data is retained (but if your consumers are down for too long, they can miss data)
- Other parameters allowed: `log.retention.ms`, `log.retention.minutes` (smaller unit has precedence)

`log.retention.bytes`:

- Max size in bytes for each partition (default is -1 - infinite)
- Useful to set to keep size of log under a threshold

There are two common pairs of options:

- One week of retention
  - `log.retention.hours=168` and `log.retention.bytes=-1`
- Infinite time of retention bounded by 500MB threshold
  - `log.retention.hours=-1` and `log.retention.bytes=524288000`

### Log Compaction

Log compaction ensures that your log contains at least the last known value for a specific key within an partition.

It's very useful if you just require a snapshot of the current state of a topic rather than its full history. Comparable to a table in a relational database where you only have the current state of the table, not all the changes previously made.

The idea is that you only keep the latest update for a key in your log. For example, the most recent salary for an employee.

When log compaction happens, only the most recent value for a specific key is kept. It does not change the offset, it just deletes keys when there are newer keys available.

Log compaction guarantees:

- Any consumer reading from the tail of a log (most current data) will still see all the messages sent to the topic.
- The ordering of messages is kept, log compaction only removes some messages, but does not reorder them
- The offset of a message is immutable (it never changes). Offsets are just skipped if a message is missing
- Deleted records can still be seen by consumers for a grace epriod of `delete.retention.ms` (default is 24 hours)

Log compaction doesn't prevent you from pushing duplicate data to Kafka, or reading duplicate data from Kafka:

- De-duplication is done after a segment is committed
- Your consumers will still read from the tail as soon as the data arrives

Log compaction can fail from time to time:

- It is an optimization and the compaction thread might crash
- Make sure you assign enough memory to it and that it gets triggered
- Restart Kafka if log compaction is broken

You can't currently trigger log compaction using an API call.

`log.cleanup.policy=compact` is impacted by:

- `segment.ms` (default 7 days) - max amount of time to wait to close an active segment
- `segment.bytes` (default 1GB) - max size of a segment
- `min.compaction.lag.ms` (default 0) - how long to wait before a message can be compacted
- `default.retention.ms` (default 24 hours) - wait before deleting data marked for compaction
- `min.cleanable.dirty.ratio` (default 0.5) - higher = less, more efficient cleaning, lower = more cleaning but less efficient

### Unclean Leader Election

If all your in sync replicas go offline, but you still have out of sync replicas up, you have the option to:

- Wait for an ISR to come back online (default)
- Enable `unclean.leader.election.enable=true` and start producing to non-ISR partitions

If you enable `unclean.leader.election.enable=true`, you improve availability but you will lose data because other messages on ISR will be discarded when they come back online and replicate data from the new leader.

It's a dangerous setting and should only be used if its full implications are understood. Use cases include metrics and log collection and other cases where data loss is somewhat acceptable, at the trade off of availability.

### Large Messages in Kafka

Kafka has a default of 1MB per message in topics, as large messages are considered inefficient and an anti-pattern.

Two approaches to sending large messages:

1. Using an external store, such as HDFS, S3, Google Cloud Storage, and sending a reference to that message to Kafka
2. Modifying Kafka parameters: you must change broker, producer, and consumer settings

Large messages using external store:

- Store the large message outside of Kafka
- Send a metadata reference of that message to Kafka
- Write custom code at the producer/consumer level to handle this pattern

Sending large messages in Kafka:

- Topic-wise, Kafka-side, set max message size to 10MB
  - Broker-side modify `message.max.bytes`
  - Topic-side modify `max.message.bytes`
- Broker-wise, set max replication fetch size to 10MB
  - `replica.fetch.max.bytes=10485880` (in `server.properties`)
- Consumer-side, increase the fetch size of the consumer or it will crash
  - `max.partition.fetch.bytes=10485880`
- Producer-side, increase the max request size
  - `max.request.size=10485880`
