---
layout: post
title: "Basic Kafka Consumer Java"
permalink: "basic-kafka-consumer-java/"
last_modified_at: 2020-04-12T00:00:00
excerpt: "Basic Kafka Consumer code in Java"
category: "kafka"
---

[Kafka Clients](https://kafka.apache.org/24/javadoc/index.html?org/apache/kafka/clients/){:target="_blank"} is official Java client for [Apache Kafka](https://kafka.apache.org/){:target="_blank"}. <small>[(compatibility matrix)](https://cwiki.apache.org/confluence/display/KAFKA/Compatibility+Matrix){:target="_blank"}</small>



#### Maven <small>[latest](https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients/){:target="_blank"}</small>
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.4.1</version>
</dependency>
```
#### Basic Kafka Consumer
```java
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

public class Consumer {

    private static final String TOPIC = "sales";

    public static void main(String[] args) {
        // Set consumer configuration properties
        final Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); // for multiple brokers, add comma separated list
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "basic-consumer");

        // Create a new consumer
        try (final KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            // Subscribe to the topic
            consumer.subscribe(Collections.singleton(TOPIC));

            // Continuously read records from the topic
            while (Thread.currentThread().isAlive()) {
                final ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("Received: [key:" + record.key() + " value:" + record.value() + "]");
                }
            }
        }
    }
}

```