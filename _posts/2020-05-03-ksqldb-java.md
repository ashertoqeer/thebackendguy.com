---
layout: post
title: "KSqlDB Java"
permalink: "basic-kafka-consumer-java/"
last_modified_at: 2020-04-12T00:00:00
excerpt: "A basic introduction to Kafka KSqlDB using Java"
category: "kafka"
---
#### Intro:
[KSqlDB](https://docs.ksqldb.io/en/latest/){:target="_blank"} is a Kafka consumer application that provides a simple SQL interface on top of Kafka stream processing APIs. It hides away all of the complexity of stream processing and allows clients to directly query for interested events without even knowing about Kafka stream APIs.

#### An Example Setup:
![A diagram of example KSql setup](/assets/img/ksqldb-java-example-setup.png)

As you can see from above sample setup, KSqlDB is totally independent of Kafka cluster and, in fact, it is just another consumer. However, it has its own clients which are NOT Kafka consumers. Instead those clients are directly querying KSqlDB and knows nothing about kafka.

This separation of concerns is extremely powerful. For example if one of the above client is a mobile app and other client is a web admin console and both are interested in same set of events to render some dashboard. If there was no KSqlDB, then both of them have to use stream apis natively which is not only duplication, but also add unnecessary complexity.

With KSqlDB, both clients just need to query using SQL like syntax and get results immediately.

#### Using KSqlDB with Java
Let's create an example demonstration using Java. Make sure you have a Kafka instance running at `localhost:9092`. Let's create a sample producer in Java (Using kafka clients library)

##### Producer
A sample producer to publish a sample product json on kafka topic `sales`. 
```java
public class Producer {

  private static String[] productUpcList = {"231984178341", "783461936482", "983174531846"};
  private static final String TOPIC = "sales";
  // sample json events list
  private static Map<String, String> products = Map.of(
          "231984178341", "{\"upc\":\"231984178341\",\"name\": \"Bread\",\"price\":50, \"category\":\"food\",\"status\":\"SOLD\"}",
          "783461936482", "{\"upc\":\"783461936482\",\"name\": \"Soap\",\"price\":50, \"category\":\"essentials\",\"status\":\"RETURN\"}",
          "983174531846", "{\"upc\":\"983174531846\",\"name\": \"Laptop\",\"price\":50, \"category\":\"tech\",\"status\":\"SOLD\"}"
  );
  public static void main(String[] args) throws InterruptedException {
      // Set producer configuration properties
      final Properties producerProps = new Properties();
      producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
      producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
      producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
      out.println("creating producer");
      // Create a new producer
      try (final KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {
          out.println("created producer");
          int index = 0;
          while (Thread.currentThread().isAlive()) {
              String key = productUpcList[index % 3];
              String product = products.get(key); // pick event from sample json
              producer.send(new ProducerRecord<>(TOPIC, key, product));
              out.println("published: [key:" + key + " value:" + product + "]");
              Thread.sleep(1000); // to mimic realistic producer
              index++;
          }
      }
  }
}
```

It will start posting following events to `sales` topic:
```json
{"upc":"231984178341","name": "Bread","price":50, "category":"food","status":"SOLD"}]
{"upc":"783461936482","name": "Soap","price":50, "category":"essentials","status":"RETURN"}]
{"upc":"983174531846","name": "Laptop","price":50, "category":"tech","status":"SOLD"}]
{"upc":"231984178341","name": "Bread","price":50, "category":"food","status":"SOLD"}]
<continued>
```
So far so good, now we need to run KSqlDB as consumer

#### KSqlDB
The easiest way of running KSqlDB is to run via docker:
```
docker run -d \
  --network host\
  -e KSQL_BOOTSTRAP_SERVERS=localhost:9092 \
  -e KSQL_LISTENERS=http://0.0.0.0:8088 \
  -e KSQL_KSQL_SERVICE_ID=ksql_service_2_ \
  confluentinc/cp-ksql-server:5.4.1 
```

+ `--network host` means use host network as docker network. we need this since kafka is at `localhost:9092`.
+ `-e KSQL_BOOTSTRAP_SERVERS=localhost:9092` telling KSqlDB where Kafka is running.
+ `-e KSQL_LISTENERS=http://0.0.0.0:8088` the endpoint which we would like to use to query KSqlDB.
+ `-e KSQL_KSQL_SERVICE_ID=ksql_service_2_` specifying serviceId

That's it, till now we got a producer publishing events on a kafka topic and a running KSqlDB. Now we need a KSqlDB client to start querying.

#### KSqlDB CLI Client
KSqlDB have a docker based console client, just run following command.

```
docker run --network host -it  confluentinc/cp-ksql-cli http://0.0.0.0:8088
```
Output would be:
```
                  
                  ===========================================
                  =        _  __ _____  ____  _             =
                  =       | |/ // ____|/ __ \| |            =
                  =       | ' /| (___ | |  | | |            =
                  =       |  <  \___ \| |  | | |            =
                  =       | . \ ____) | |__| | |____        =
                  =       |_|\_\_____/ \___\_\______|       =
                  =                                         =
                  =  Streaming SQL Engine for Apache Kafka® =
                  ===========================================

Copyright 2017-2019 Confluent Inc.

CLI v5.4.1, Server v5.4.1 located at http://0.0.0.0:8088

Having trouble? Type 'help' (case-insensitive) for a rundown of how things work!

ksql> 

```
Now we can query using supported Sql syntax. Our producer is publishing events on `sales` topic, lets print those events.

```
ksql> PRINT sales;
```
```json
Format:JSON
{"ROWTIME":1588605554189,"ROWKEY":"783461936482","upc":"783461936482","name":"Soap","price":50,"category":"essentials","status":"RETURN"}
{"ROWTIME":1588605555190,"ROWKEY":"983174531846","upc":"983174531846","name":"Laptop","price":50,"category":"tech","status":"SOLD"}
{"ROWTIME":1588605556191,"ROWKEY":"231984178341","upc":"231984178341","name":"Bread","price":50,"category":"food","status":"SOLD"}
{"ROWTIME":1588605557192,"ROWKEY":"783461936482","upc":"783461936482","name":"Soap","price":50,"category":"essentials","status":"RETURN"}
{"ROWTIME":1588605558193,"ROWKEY":"983174531846","upc":"983174531846","name":"Laptop","price":50,"category":"tech","status":"SOLD"}
{"ROWTIME":1588605559194,"ROWKEY":"231984178341","upc":"231984178341","name":"Bread","price":50,"category":"food","status":"SOLD"}
<continued>
```
Perfect, we got our events getting published by producer. Now move towards next step, querying the data. KSqlDB doesn't directly query topic, instead it has its own abstractions on topic, called Streams and Tables.

You need to use Streams and Tables to query data in KSqlDB.

The Stream and Table both operate on events in a different way. 

Stream is a sequence of events, it is immutable, it only supports inserting new events while existing event can't not be changed.

Table is an aggregation of events, it is mutable, it supports inserting new events (rows), as well as, updating existing events (rows).

Here is a really nice blog post on difference between Streams and Tables [Streams and Tables in Apache Kafka: A Primer](https://www.confluent.io/blog/kafka-streams-tables-part-1-event-streaming/){:target="_blank"}

#### KSqlDB Creating and Querying Stream

Let's create a stream from our `sales` topic and execute some queries.
###### Create a Stream:
```
ksql> CREATE STREAM sales_stream (upc string, name string, price double, category string, status string) WITH (KAFKA_TOPIC='sales', VALUE_FORMAT='JSON');
```
```
 Message        
----------------
 Stream created 
----------------
ksql> 
```
###### Describe Stream:
```
ksql> DESCRIBE sales_stream;
```
```
Name                 : SALES_STREAM
 Field    | Type                      
--------------------------------------
 ROWTIME  | BIGINT           (system) 
 ROWKEY   | VARCHAR(STRING)  (system) 
 UPC      | VARCHAR(STRING)           
 NAME     | VARCHAR(STRING)           
 PRICE    | DOUBLE                    
 CATEGORY | VARCHAR(STRING)           
 STATUS   | VARCHAR(STRING)           
--------------------------------------
For runtime statistics and query details run: DESCRIBE EXTENDED <Stream,Table>;
ksql> 
```
###### Filter out only Return Items:
```
ksql> SELECT * FROM sales_stream where status = 'RETURN' EMIT CHANGES;
```
```
+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+
|ROWTIME                     |ROWKEY                      |UPC                         |NAME                        |PRICE                       |CATEGORY                    |STATUS                      |
+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+----------------------------+
|1588607970163               |783461936482                |783461936482                |Soap                        |50.0                        |essentials                  |RETURN                      |
|1588607973164               |783461936482                |783461936482                |Soap                        |50.0                        |essentials                  |RETURN                      |
|1588607976165               |783461936482                |783461936482                |Soap                        |50.0                        |essentials                  |RETURN                      |
|1588607979166               |783461936482                |783461936482                |Soap                        |50.0                        |essentials                  |RETURN                      |
<continued>
```

That `EMIT CHANGES` indicates that this is a Push query. There are two types of queries in KSqlDB Push queries and Pull queries.

Push queries are NEVER TERMINATING queries, they constantly push results in real time.

Pull queries work like traditional queries, they pull some specific data and then terminate. 

From more on queries: [https://docs.ksqldb.io/en/latest/concepts/queries/](https://docs.ksqldb.io/en/latest/concepts/queries/)

###### Number of Sold items in Real Time:
```
ksql> SELECT upc, name, count(*) as sold FROM sales_stream  group by upc, name, status having status='SOLD'  EMIT CHANGES;
```
```
+--------------------------------------------------------------------+--------------------------------------------------------------------+--------------------------------------------------------------------+
|UPC                                                                 |NAME                                                                |SOLD                                                                |
+--------------------------------------------------------------------+--------------------------------------------------------------------+--------------------------------------------------------------------+
|983174531846                                                        |Laptop                                                              |1                                                                   |
|231984178341                                                        |Bread                                                               |1                                                                   |
|983174531846                                                        |Laptop                                                              |2                                                                   |
|231984178341                                                        |Bread                                                               |2                                                                   |
|983174531846                                                        |Laptop                                                              |3                                                                   |
|231984178341                                                        |Bread                                                               |3                                                                   |
|983174531846                                                        |Laptop                                                              |4                                                                   |
<continued>
```

You could do a lot more with KSqlDB queries, see [Developer Guide](https://docs.ksqldb.io/en/latest/developer-guide/){:target="_blank"}

#### KSqlDB Java Client:
So far, we have used KSqlDB CLI client, its time to get some taste of how to query from a Java application.

KSqlDB provides a Rest endpoint to execute query and return results. From the [docs](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-rest-api/query-endpoint/):

> The /query resource lets you stream the output records of a SELECT statement via a chunked transfer encoding. The response is streamed back until the LIMIT specified in the statement is reached, or the client closes the connection. If no LIMIT is specified in the statement, then the response is streamed until the client closes the connection.

###### KSqlDB Java Client (Java 11)

```java
public class KSqlClient {

  public static void main(String[] args) throws Exception {
      // Sample query
      String queryJson = "{\"ksql\":\"SELECT * FROM sales_stream where status = 'RETURN' EMIT CHANGES;\"}";
      final HttpClient httpClient = HttpClient.newBuilder().version(HttpClient.Version.HTTP_2).build();
      HttpRequest request = HttpRequest.newBuilder()
              .POST(HttpRequest.BodyPublishers.ofString(queryJson))
              .uri(URI.create("http://localhost:8088/query")) // KsqlDB endpoint
              .build();
      HttpResponse<InputStream> response = httpClient.send(request, BodyHandlers.ofInputStream());
      // print response body
      try (BufferedReader br = new BufferedReader(new InputStreamReader(response.body()))) {
          String event = "";
          while ((event = br.readLine()) != null) {
              if (!event.isEmpty()) {
                  log("event: " + event);
              }
          }
      }
  }
}
```
Thanks to chunked transfer encoding, we will get a constantly running sequence of events.

That's all, [Documentation](https://docs.ksqldb.io/en/latest/developer-guide/). 