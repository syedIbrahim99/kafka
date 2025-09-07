# Kafka Manual Installation and Hands-On Guide

## Kafka Installation

**Official documentation link:** [Kafka Quickstart](https://kafka.apache.org/quickstart)

**Kafka tar installation official document link:** [Download Kafka 4.1.0](https://www.apache.org/dyn/closer.cgi?path=/kafka/4.1.0/kafka_2.13-4.1.0.tgz)

**kafka official GitHub link** [Visit Kafka Git Repo](https://github.com/apache/kafka)

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
echo $KAFKA_CLUSTER_ID
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
bin/kafka-server-start.sh config/server.properties
```

* Kafka server started and running in foreground.

---

## Scenario 1: Basic Producer → Consumer

### Create a topic

* PWD - /home/syed/kafka-test/kafka_2.13-4.1.0

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

* PWD - /home/syed/kafka-test/kafka_2.13-4.1.0

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
* This will be running in fore-ground.

* File `test.sink.txt` will be created once the above command is executed.

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

* How the `connect-test` topic automatically expects the `test.txt` and how `test.sink.txt` is created dynamically without we explicitly passing any values for them.

* This is a internal behaviour of `Kafka Connect`.

* nano `config/connect-file-source.properties` default file.

```bash
name=local-file-source
connector.class=FileStreamSource
tasks.max=1
file=test.txt
topic=connect-test
```
* nano `config/connect-file-sink.properties` default file.

```bash
name=local-file-sink
connector.class=FileStreamSink
tasks.max=1
file=test.sink.txt
topics=connect-test
```

* This is how this `connect-test` topic of `kafka Connect` automatically expects the `test.txt` and creates a `test.sink.txt` files.

* And also this `Kafka Connect` scenario is happening with the help of the jar file `connect-file-4.1.0.jar` which have bundled all the source code init it relies in the path `kafka_2.13-4.1.0/libs/connect-file-4.1.0.jar`.

---

## Scenario 3: Kafka Streams (Process Events in Real-Time)

**Official documentation link:** [Run Kafka Streams Demo Application](https://kafka.apache.org/documentation/streams/quickstart)

## Run Kafka Streams Demo Application

### Prepare input topic and start Kafka producer

* PWD - /home/syed/kafka-test/kafka_2.13-4.1.0

```bash
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic streams-plaintext-input

bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic streams-wordcount-output --config cleanup.policy=compact

bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe

bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

bin/kafka-run-class.sh org.apache.kafka.streams.examples.wordcount.WordCountDemo
```

* This command starts the "wordcountDemo" application and runs in fore-ground.

### New terminal:

* pwd - /home/syed/kafka-test/kafka_2.13-4.1.0

```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
>all streams lead to kafka
```

### New terminal:

* pwd - /home/syed/kafka-test/kafka_2.13-4.1.0

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --from-beginning --property print.key=true --property print.value=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
all     2
streams 3
lead    2
to      2
kafka   4
```

## continuation

### New terminal:

* pwd - /home/syed/kafka-test/kafka_2.13-4.1.0

```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
>all streams lead to kafka
>hello kafka streams
```
### New terminal:

* pwd - /home/syed/kafka-test/kafka_2.13-4.1.0

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --from-beginning --property print.key=true --property print.value=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
all     2
streams 3
lead    2
to      2
kafka   4
hello   2
kafka   5
streams 4
```

* In this scenario created two topics named `streams-plaintext-input` `streams-wordcount-output`.

* The second topic is created with the flag `--config cleanup.policy=compact` to stream the latest data.

* Normally Kafka topics keep every message forever (until we delete).

* But here, we don’t need the history of old data, it just reflects the latest data.

* Then ran the `wordcountDemo` starting command, which will be running as a process in `fore-ground`.

* Then ran a `producer` for `streams-plaintext-input` and typed some messaged in new terminal.

* Then ran a `consumer` for `streams-wordcount-output` and got the typed messages in a word count fromat with the help of `wordcountDemo` java program running in a seperate terminal.

* Here one thing we can notice is, the first topic `streams-plaintext-input` and the second topic `streams-wordcount-output` doesn't have any connection between each other and how the message we typed in `streams-plaintext-input` topic's producer is reflected in the consumer of `streams-wordcount-output`.

* It is already hardcoded in the java program which we have executed, so that program will be expecting two topcics from this names, one as a `producer` and other as a `consumer`.

* These all are pre-builted when we installed `kafka`.

* we can see the java program to see the logic by extracting the jar file  in the path `kafka_2.13-4.1.0/libs/kafka-streams-examples-4.1.0.jar`.

* The extracted tar will have the `java` codes as `.class` extension.

* So we can see the codes in the `GitHub` repositories.

**kafka official GitHub link** [Visit Kafka Git Repo](https://github.com/apache/kafka/blob/trunk/streams/examples/src/main/java/org/apache/kafka/streams/examples/wordcount/WordCountDemo.java)

---

## Terminate the Kafka Environment

```bash
# Stop all consumers, producers, and brokers using Ctrl+C
rm -rf /tmp/kafka-logs /tmp/kraft-combined-logs
rm -rf /tmp/connect.offsets    # Deleting the saved offsets (bookmark)
```
