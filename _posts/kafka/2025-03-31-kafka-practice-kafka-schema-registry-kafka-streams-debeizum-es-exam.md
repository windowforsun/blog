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
