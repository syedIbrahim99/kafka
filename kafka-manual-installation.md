# Kafka Manual Installation and Hands-On Guide

## Kafka Installation

**Official documentation link:** [Kafka Quickstart](https://kafka.apache.org/quickstart)

**Kafka tar installation official document link:** [Download Kafka 4.1.0](https://www.apache.org/dyn/closer.cgi?path=/kafka/4.1.0/kafka_2.13-4.1.0.tgz)

```bash
mkdir kafka-test
cd kafka-test
wget https://dlcdn.apache.org/kafka/4.1.0/kafka_2.13-4.1.0.tgz
tar -xzf kafka_2.13-4.1.0.tgz
cd kafka_2.13-4.1.0
```

**NOTE:** Local environment must have **Java 17+ installed**.

### Install Java 17 on Ubuntu

```bash
sudo apt update
sudo apt install openjdk-17-jdk
java -version
```

### Start Kafka

```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
bin/kafka-server-start.sh config/server.properties
```

* Kafka server started and running in foreground.

---

## Scenario 1: Basic Producer → Consumer

### Create a topic

```bash
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Write events into the topic

```bash
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
>This is my first event
>This is my second event
```

### Read the events

```bash
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```

Output:

```
This is my first event
This is my second event
```

* Without `--from-beginning`, only the latest messages are shown:

```bash
bin/kafka-console-consumer.sh --topic quickstart-events --bootstrap-server localhost:9092
This is my third event
```

---

## Scenario 2: Kafka Connect (Import/Export Data)

### Configure Kafka Connect

```bash
nano config/connect-standalone.properties
# Add line at the bottom:
plugin.path=libs/connect-file-4.1.0.jar
```

### Prepare source file

```bash
echo -e "foo\nbar" > test.txt
```

### Start Kafka Connect

```bash
bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties
```

* File `test.sink.txt` is created.

```bash
more test.sink.txt
foo
bar
```

### Check topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
# Output:
# connect-test
# quickstart-events
```

### Consume messages from connect-test

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
```

Output:

```
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
```

* Add a new line to `test.txt`:

```bash
echo "Another line" >> test.txt
```

* Consumer output:

```
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
{"schema":{"type":"string","optional":false},"payload":"Another line"}
```

---

## Scenario 3: Kafka Streams (Process Events in Real-Time)

### Create output topic

```bash
bin/kafka-topics.sh --create --topic output-topic --bootstrap-server localhost:9092
```

### Prepare Kafka Streams application

```bash
cd ..
mkdir kafka-streams-app
cd kafka-streams-app
mkdir -p src/main/java/org/example
nano src/main/java/org/example/WordCountApp.java
```

**WordCountApp.java:**

```java
package org.example;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Produced;

import java.util.Arrays;
import java.util.Properties;

public class WordCountApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> textLines = builder.stream("quickstart-events");

        KTable<String, Long> wordCounts = textLines
                .flatMapValues(line -> Arrays.asList(line.toLowerCase().split(" ")))
                .groupBy((keyIgnored, word) -> word)
                .count();

        wordCounts.toStream().to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

### Compile Java program

```bash
javac -cp "/home/syed/kafka-test/kafka_2.13-4.1.0/libs/*" -d . src/main/java/org/example/WordCountApp.java
```

* Program should be running in foreground.

### Consume output from Kafka Streams

```bash
bin/kafka-console-consumer.sh \
  --topic output-topic \
  --from-beginning \
  --bootstrap-server localhost:9092 \
  --property print.key=true \
  --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

Sample output:

```
first   1
this    2
is      2
my      2
second  1
event   2
hello   1
kafka   1
syed    1
hello   2
world   1
```

* we can also produce new messages:

```bash
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
>hello world
```

---

## Terminate the Kafka Environment

```bash
# Stop all consumers, producers, and brokers using Ctrl+C
rm -rf /tmp/kafka-logs /tmp/kraft-combined-logs
```
