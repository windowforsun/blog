--- 
layout: single
classes: wide
title: "[Kafka] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
toc: true
use_math: true
---  

## Schema Registry with Debezium-ES and Kafka Streams
[Schema Registry](https://windowforsun.github.io/blog/tags/#schema-registry)
에서 알아본 것처럼 `Schema Registry` 는 `Kafka` 기반 메시징을 처리할 때 메시지 스키마 관리와 버전 제어 및 
`Avro` 메시지를 활용해 메시지 크기와 빠른 직렬화를 제공하는 중요한 역할을 한다. 
`Debezium CDC` 를 사용할 때 `Schema` 를 활성화해 메시지를 보다 명시적으로 처리할 수 있는데, 
이때 `Avro` 형식과 `Schema Registry` 를 함께 연동하는 방법에 대해 알아본다. 
그리고 더 나아가 `Kafka Streams` 애플리케이션에서 `Debezium` 에서 등록한 스키마를 활용해 메시지를 처리하는 방법에 대해 알아본다. 

전체 구성과 대략적인 흐름은 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-kafka-streams-debezium-es-1.drawio.png)


1. `Debezium MySQL Source Connector` 에서 `Avro` 와 스키마를 활성화 및 `Schema Registry` 연결 설정을 추가해 실행한다. 
2. `Debezium` 커넥터에서 `Kafka` 로 전송하는 메시지의 스키마를 `Schema Registry` 에 등록한다. 
3. `Kafka Streams` 애플리케이션에서 `Debezium` 에서 등록한 스키마를 활용해 메시지를 처리한다. 
4. `Elasticsearch Sink Connecotor` 에서도 동일하게 `Avro` 와 `Schema Registry` 연결 설정을 추가해 실행한다. 
5. `Elasticsearch` 커넥터는 `Avro` 메시지를 사용해 `Elasticsearch` 에 데이터를 적재한다.  

이후 진행되는 데모의 전체 코드는 [여기]()
에서 확인할 수 있다.  

### Full Configuration
데모 구성에 필요한 모든 구성은 `docker-compose` 를 기반으로 구축한다. 
템플릿 내용은 아래와 같다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: confluentinc/cp-zookeeper:7.0.16
    environment:
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
    ports:
      - "2181:2181"

  kafka:
    container_name: myKafka
    image: confluentinc/cp-kafka:7.0.16
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "29092:29092"

  schemaRegistry:
    container_name: schemaRegistry
    image: confluentinc/cp-schema-registry:7.0.16
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL : 'zookeeper:2181'
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'PLAINTEXT://kafka:9092'
      SCHEMA_REGISTRY_HOST_NAME: schemaRegistry
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
      SCHEMA_REGISTRY_DEBUG: 'true'
    depends_on:
      - zookeeper
      - kafka

  debezium-avro-source:
    container_name: debezium-avro-source
    build:
      context: ./debezium-avro-source
      dockerfile: Dockerfile
    ports:
      - "8083:8083"
    environment:
      CONNECT_GROUP_ID: 'debezium-avro-source'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'debezium-avro-source-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'debezium-avro-source-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'debezium-avro-source-status'
      CONNECT_KEY_CONVERTER: 'io.confluent.connect.avro.AvroConverter'
      CONNECT_VALUE_CONVERTER: 'io.confluent.connect.avro.AvroConverter'
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schemaRegistry:8081'
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schemaRegistry:8081'
      CONNECT_REST_ADVERTISED_HOST_NAME: debezium-avro-source
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
    depends_on:
      - zookeeper
      - kafka
      - schemaRegistry

  es-avro-sink:
    container_name: es-avro-sink
    build:
      context: ./es-avro-sink
      dockerfile: Dockerfile
    ports:
      - "8084:8083"
    environment:
      CONNECT_GROUP_ID: 'es-avro-sink'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'es-avro-sink-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'es-avro-sink-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'es-avro-sink-status'
      CONNECT_KEY_CONVERTER: 'io.confluent.connect.avro.AvroConverter'
      CONNECT_VALUE_CONVERTER: 'io.confluent.connect.avro.AvroConverter'
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schemaRegistry:8081'
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schemaRegistry:8081'
      CONNECT_REST_ADVERTISED_HOST_NAME: es-avro-sink
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
    depends_on:
      - zookeeper
      - kafka
      - schemaRegistry

  es-single:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0
    container_name: es-single
    environment:
      - xpack.security.enabled=false
      - node.name=es-single
      - cluster.name=es-single
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
      - "9300:9300"

  originDB:
    container_name: originDB
    image: mysql:8.0.29
    ports:
      - "3306:3306"
    volumes:
      - ./mysql.cnf:/etc/mysql/conf.d/custom.cnf
      - ./init-sql/:/docker-entrypoint-initdb.d/
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw
```  

데모 진행을 위해 아래 명령으로 전체 구성을 실행한다.  

```bash
$ docker-compose -f docker/docker-compose.yaml up --build
```  

### Debezium Avro Source Connector
`Kafka Connector` 에서 `Avro` 및 `Schema Registry` 를 사용하고자 할때, 
활성화 및 필요한 설정은 아래와 같다.  


항목|Schema Registry 사용|Schema Registry 미사용
---|---|---
데이터 포맷|Avro|JSON, String 등
key.converter|설정|io.confluent.connect.avro.AvroConverter|org.apache.kafka.connect.storage.StringConverter, org.apache.kafka.connect.json.JsonConverter 등
value.converter 설정|io.confluent.connect.avro.AvroConverter|org.apache.kafka.connect.storage.StringConverter, org.apache.kafka.connect.json.JsonConverter 등
schema.registry.url 설정|필요 (http://localhost:8081)|불필요
스키마 포함 여부 (schemas.enable)|true (스키마 포함)|false (스키마 미포함)

`Debezium Avro Source Connector` 를 실행하기 위한 설정은 아래와 같다.  

```json
{
  "name": "debezium.avro.source",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "topic.prefix": "debezium.avro.source",
    "database.hostname": "originDB",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.id": "12223232",
    "database.allowPublicKeyRetrieval": "true",
    "database.server.name": "originDB",
    "database.include.list": "user",
    "database.serverTimezone": "UTC",
    "table.include.list": "user.user_account",
    "database.characterEncoding": "utf8",
    "database.useUnicode": "true",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "debezium.avro.source.history",
    "include.schema.changes": "true",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schemaRegistry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schemaRegistry:8081"
  }
}
```  

커넥터를 실행하고 `Kafka Topic` 및 `Schema Registry` 를 확인하면 아래와 같다.  

```bash
# 커넥터 실행
$ curl -X POST -H "Content-Type: application/json" \
--data @docker/connector-config/debezium-avro-source.json \
http://localhost:8083/connectors | jq

# kafka topics
$ docker exec -it myKafka kafka-topics \
--bootstrap-server localhost:9092 \
--list
__consumer_offsets
_schemas
debezium-avro-source-config
debezium-avro-source-offset
debezium-avro-source-status
debezium.avro.source
debezium.avro.source.history
debezium.avro.source.user.user_account

# kafka topic message
$ docker exec -it myKafka kafka-console-consumer \
--bootstrap-server localhost:9092 \
--topic debezium.avro.source.user.user_account \
--from-beginning
jack2.6.0.Final
mysql(debezium.avro.source�ܼ��lasuser�Բ┳����۠��0user_accountbinlog.000002�r�༥�e���┳����բ��0

# schema registry subjects
$ curl -X GET http://localhost:8081/subjects | jq
[
  "debezium.avro.source-key",
  "debezium.avro.source-value",
  "debezium.avro.source.user.user_account-key",
  "debezium.avro.source.user.user_account-value",
]

# cdc schema detail key and value
$ curl -X GET http://localhost:8081/subjects/debezium.avro.source.user.user_account-key/versions/1 | jq
{
  "subject": "debezium.avro.source.user.user_account-key",
  "version": 1,
  "id": 3,
  "schema": "{\"type\":\"record\",\"name\":\"Key\",\"namespace\":\"debezium.avro.source.user.user_account\",\"fields\":[{\"name\":\"uid\",\"type\":\"int\"}],\"connect.name\":\"debezium.avro.source.user.user_account.Key\"}"
}

$ curl -X GET http://localhost:8081/subjects/debezium.avro.source.user.user_account-value/versions/1 | jq
{
  "subject": "debezium.avro.source.user.user_account-value",
  "version": 1,
  "id": 4,
  "schema": "{\"type\":\"record\",\"name\":\"Envelope\",\"namespace\":\"debezium.avro.source.user.user_account\",\"fields\":[{\"name\":\"before\",\"type\":[\"null\",{\"type\":\"record\",\"name\":\"Value\",\"fields\":[{\"name\":\"uid\",\"type\":\"int\"},{\"name\":\"name\",\"type\":[\"null\",\"string\"],\"default\":null}],\"connect.name\":\"debezium.avro.source.user.user_account.Value\"}],\"default\":null},{\"name\":\"after\",\"type\":[\"null\",\"Value\"],\"default\":null},{\"name\":\"source\",\"type\":{\"type\":\"record\",\"name\":\"Source\",\"namespace\":\"io.debezium.connector.mysql\",\"fields\":[{\"name\":\"version\",\"type\":\"string\"},{\"name\":\"connector\",\"type\":\"string\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"ts_ms\",\"type\":\"long\"},{\"name\":\"snapshot\",\"type\":[{\"type\":\"string\",\"connect.version\":1,\"connect.parameters\":{\"allowed\":\"true,last,false,incremental\"},\"connect.default\":\"false\",\"connect.name\":\"io.debezium.data.Enum\"},\"null\"],\"default\":\"false\"},{\"name\":\"db\",\"type\":\"string\"},{\"name\":\"sequence\",\"type\":[\"null\",\"string\"],\"default\":null},{\"name\":\"ts_us\",\"type\":\"long\"},{\"name\":\"ts_ns\",\"type\":\"long\"},{\"name\":\"table\",\"type\":[\"null\",\"string\"],\"default\":null},{\"name\":\"server_id\",\"type\":\"long\"},{\"name\":\"gtid\",\"type\":[\"null\",\"string\"],\"default\":null},{\"name\":\"file\",\"type\":\"string\"},{\"name\":\"pos\",\"type\":\"long\"},{\"name\":\"row\",\"type\":\"int\"},{\"name\":\"thread\",\"type\":[\"null\",\"long\"],\"default\":null},{\"name\":\"query\",\"type\":[\"null\",\"string\"],\"default\":null}],\"connect.name\":\"io.debezium.connector.mysql.Source\"}},{\"name\":\"op\",\"type\":\"string\"},{\"name\":\"ts_ms\",\"type\":[\"null\",\"long\"],\"default\":null},{\"name\":\"ts_us\",\"type\":[\"null\",\"long\"],\"default\":null},{\"name\":\"ts_ns\",\"type\":[\"null\",\"long\"],\"default\":null},{\"name\":\"transaction\",\"type\":[\"null\",{\"type\":\"record\",\"name\":\"block\",\"namespace\":\"event\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"total_order\",\"type\":\"long\"},{\"name\":\"data_collection_order\",\"type\":\"long\"}],\"connect.version\":1,\"connect.name\":\"event.block\"}],\"default\":null}],\"connect.version\":2,\"connect.name\":\"debezium.avro.source.user.user_account.Envelope\"}"
}

```  

`Debezium Avro Source Connector` 에서 `Kafka` 로 `Avro` 타입으로 메시지를 전송하고 메시지와 매핑되는 스키마를 `Schema Registry` 에 등록한 것을 볼 수 있다.  

### Elasticsearch Avro Sink Connector 
`Elasticsearch` 커넥터도 앞서 `Debezium Avro Source Connector` 와 동일하게 `Avro` 및 `Schema Registry` 설정을 추가하면, 
`Avro` 메시지를 `Elasticsearch` 로 저장할 수 있다. 
설정은 아래와 같다.  

```json
{
  "name": "es-avro-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "debezium.avro.source.user.user_account",
    "connection.url": "http://es-single:9200",
    "connection.username" : "root",
    "connection.password" : "root",
    "transforms": "InsertTimestamp",
    "transforms.InsertTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertTimestamp.timestamp.field": "timestamp",
    "type.name" :"_doc",
    "key.ignore" : "true",
    "key.converter" : "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url" : "http://schemaRegistry:8081",
    "value.converter" : "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url" : "http://schemaRegistry:8081"
  }
}
```  

커넥터를 실행하고, `Elasticsearch` 에 적재된 데이터를 확인하면 아래와 같다.  

```bash
# 커넥터 실행
$ curl -X POST -H "Content-Type: application/json" \
--data @docker/connector-config/es-avro-sink.json \
http://localhost:8084/connectors | jq

# elasticsearch index
$ curl -X GET http://localhost:9200/_aliases\?pretty\=true
{
  "debezium.avro.source.user.user_account" : {
    "aliases" : { }
  }
}

# elasticsearch data
$ curl -X GET http://localhost:9200/debezium.avro.source.user.user_account/_search\?pretty | jq
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "debezium.avro.source.user.user_account",
        "_type": "_doc",
        "_id": "debezium.avro.source.user.user_account+0+0",
        "_score": 1,
        "_source": {
          "before": null,
          "after": {
            "uid": 1,
            "name": "jack"
          },
          "source": {
            "version": "2.6.0.Final",
            "connector": "mysql",
            "name": "debezium.avro.source",
            "ts_ms": 1740306356000,
            "snapshot": "last",
            "db": "user",
            "sequence": null,
            "ts_us": 1740306356000000,
            "ts_ns": 1740306356000000000,
            "table": "user_account",
            "server_id": 0,
            "gtid": null,
            "file": "binlog.000002",
            "pos": 157,
            "row": 0,
            "thread": null,
            "query": null
          },
          "op": "r",
          "ts_ms": 1740306356262,
          "ts_us": 1740306356262071,
          "ts_ns": 1740306356262071000,
          "transaction": null,
          "timestamp": 1740306357296
        }
      }
    ]
  }
}
``` 

`MySQL` 의 테이블 데이터가 `Debezium Connector` 를 통해 `Avro` 로 `Kafka` 에 전성되고, 
최종적으로 `Elasticsearch Connector` 를 통해 `Elasticsearch` 까지 메시지가 잘 적재 됐다.  

### Kafka Streams with Schema Registry
`Debezium Avro Source Connector` 가 `Kafka` 에 저장한 `Avro` 메시지는 `Kafka Streams` 에서도 `Avro` 와 `Schema Registry` 
관련 설정을 추가하면 활용할 수 있다. 
필요한 설정은 아래와 같다.  

- 아래 의존성 추가

```groovy
dependencies {
    implementation 'io.confluent:kafka-streams-avro-serde:7.6.0'
    implementation 'org.apache.avro:avro:1.11.1'
}
```

- `spring.kafka.properties.schema.registry.url` 에 사용할 `Schema Registry` 주소 설정
- `spring.kafka.streams.default.key.serde` 에 `io.confluent.kafka.streams.serdes.avro.GenericAvroSerde` 또는 `io.confluent.kafka.streams.serdes.avro.SpecificAvroSerde` 설정
- `spring.kafka.streams.default.value.serde` 에 `io.confluent.kafka.streams.serdes.avro.GenericAvroSerde` 또는 `io.confluent.kafka.streams.serdes.avro.SpecificAvroSerde` 설정

`Kafka Streams` 애플리케이션에서는 2가지 방식으로 `Avro` 메시지를 사용할 수 있는데 `GenericAvroSerde` 와 `SpecificAvroSerde` 가 있다. 

- `GenericAvroSerde` 는 `Avro` 메시지를 `GenericRecord` 로 처리하는 방식으로 `Avro` 메시지의 스키마를 직접 다루는 방식이다. 별도로 메시지에 대한 스키마 정의가 필요하지 않다. 
- `SpecificAvroSerde` 는 `Avro` 메시지를 `SpecificRecord` 로 처리하는 방식으로 `Avro` 메시지의 스키마를 정의한 클래스를 사용하는 방식이다.  

먼저 `GenericAvroSerde` 를 사용하는 방법은 아래와 같다. 
`defaul.key.serde` 와 `default.value.serde` 설정을 모두 `GenericAvroSerde` 로 해준 뒤, 별도 `Avro` 스키마 정의 없이 
아래와 같이 `Kafka Streams` 처리 코드를 `GenericRecord` 타입으로 정의해 주면 된다.  

```java
@Bean
public KStream<GenericRecord, GenericRecord> stream(StreamsBuilder streamsBuilder) {
    KStream<GenericRecord, GenericRecord> stream = streamsBuilder.stream(this.inboundTopic);

    stream.foreach((key, value) -> log.info("{} {}", key, value));
    stream.foreach((key, value) -> log.info("{} {}", key, value.get("after")));

    return stream;
}
/*
{"uid": 1} {"before": null, "after": {"uid": 1, "name": "jack"}, "source": {"version": "2.6.0.Final", "connector": "mysql", "name": "debezium.avro.source", "ts_ms": 1740306356000, "snapshot": "last", "db": "user", "sequence": null, "ts_us": 1740306356000000, "ts_ns": 1740306356000000000, "table": "user_account", "server_id": 0, "gtid": null, "file": "binlog.000002", "pos": 157, "row": 0, "thread": null, "query": null}, "op": "r", "ts_ms": 1740306356262, "ts_us": 1740306356262071, "ts_ns": 1740306356262071000, "transaction": null}
{"uid": 1} {"uid": 1, "name": "jack"}
*/
```  

`GenericRecord` 를 사용하면 역직렬화 대상이 되는 레코드와 매핑되는 스키마 정의가 필요하지 않기 때문에 간단하게 사용할 수 있다. 
하지만 하위 필드를 조회하거나 처리할 때 명시적인 타입으로 조회가 불가능하고, `Object` 타입이 리턴된다.   

다음으로 `SpecificAvroSerde` 를 사용하는 방법은 아래와 같다. 
`defaul.key.serde` 와 `default.value.serde` 설정을 모두 `SpecificAvroSerde` 로 해준 뒤, 별도 `Avro` 스키마를 정의해야 하는데 
이는 `Debezium` 이 등록한 스키마를 그대로 사용한다.  
