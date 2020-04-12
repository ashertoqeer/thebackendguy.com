---
layout: post
title: "Basic Kafka Producer Java"
permalink: "basic-kafka-producer-java/"
last_modified_at: 2020-04-12T00:00:00
excerpt: "Basic Kafka Producer code in Java"
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

#### Basic Kafka Producer
```java
import java.util.Arrays;
import java.util.List;
import java.util.Properties;
import java.util.UUID;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

public class Producer {

    private static final String TOPIC = "sales";

    // sample json events list
    private static List<String> products = Arrays.asList(
            "{\"upc\":3,\"name\": \"Bread\",\"price\":50, \"category\":\"food\",\"status\":\"SOLD\"}",
            "{\"upc\":7,\"name\": \"Soap\",\"price\":50, \"category\":\"essentials\",\"status\":\"RETURN\"}",
            "{\"upc\":9,\"name\": \"Laptop\",\"price\":50, \"category\":\"tech\",\"status\":\"SOLD\"}"
    );

    public static void main(String[] args) throws InterruptedException {
        // Set producer configuration properties
        final Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        System.out.println("creating producer");
        // Create a new producer
        try (final KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {
            System.out.println("created producer");
            int index = 0;
            while (Thread.currentThread().isAlive()) {
                String key = UUID.randomUUID().toString().substring(0, 3); // create a random key
                String product = products.get(index % 3); // pick event from sample json list, one by one
                producer.send(new ProducerRecord<>(TOPIC, key, product));
                System.out.println("published: [key:" + key + " value:" + product + "]");
                Thread.sleep(1000); // to mimic realistic producer
                index++;
            }
        }
    }
}
```