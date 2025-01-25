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

## Kafka Connect HA
`Kafka Connect` 는 데이터 스트림의 수집과 변환을 자동화하는 분산 시스템이다.
이를 활용하면 다양한 데이터 소스를 `Kafka` 로 수집하거나 `Kafka` 에서 다른 데이터 소스로 싱크를 구성 할 수 있다. 
대규모 실시간 데이터 처리 환경에서는 고가용성(`High Availability`)을 보장하여 시스템 중단 없이 안정적인 운영을 유지하는 것이 중요하다.
이번에는 `Kafka Connect` 를 구성하고 사용 할 때 `HA(High Availability)` 를 어떠한 방법으로 보장 받을 수 있는지 알아본다.  

- 무중단 : 데이터 수집 및 전달의 중단 없이 지속적으로 운영될 수 있어야 한다. 
- 장애 복구 : `Worker` 나 `Connector` 가 장애를 겪는 상황에서 빠르게 복구 할 수 있어야 한다. 
- 성능 및 확장성 : 여러 `Worker` 가 작업을 분산 처리함으로써 성능을 향상시키고, 시스템 확장성이 용이하다. 

### Kafka Connect Cluster
`Kafka Connect` 의 고가용성을 위해서는 `Standalone` 보다는 `Distributed` 모드를 사용해 클러스터를 구성해야 한다. 
분산 모드에서는 여러 `Worker` 가 협력하여 데이터를 처리하며, `Worker` 간 로드 밸린성과 장애 복구가 자동으로 이뤄진다. 
이는 여러 `Worker` 가 동일한 클러스터에 속하면, 동일한 작업을 공유하며 하나의 `Worker` 가 실패하더라도 다른 `Worker` 가 작업을 계속 처리 할 수 있게 된다.  

`Kafka Connect Cluster` 를 구성할 때 필수적인 설정 옵션은 아래와 같다. 
더 상세한 옵션과 설명은 [여기](https://docs.confluent.io/platform/current/connect/references/allconfigs.html#distributed-worker-configuration)
에서 확인 할 수 있다.  

- `group.id` : `Kafka Connect Cluster` 의 모든 `Worker` 가 동일한 `gorup.id` 를 사용하여 하나의 클러스터로 동작할 수 있도록한다. 즉 동일한 `group.id` 를 사용해 여러 `Kafka Connect` 를 구성 하면 이는 `group.id` 를 기준으로 하나의 클러스트롤 이루고 작업 분산과 장애처리를 수행한다. 
- `config.storage.topic` : `Kafka Connect` 설정을 저장하는 `Kafka Topic` 이다. 
- `offset.storage.topic` : `Kafka Connect` 가 처리한 데이터의 `offset` 을 자정하는 `Kafka Topic` 이다. 
- `status.storage.topic` : `Kafka Connect` 및 `Worker` 상태 정보를 저장하는 `Kafka Topic` 이다. 
- `config.storage.replication.factor` : `config.storage.topic` 의 복제 팩터로 최소 3이상 설정이 필요하다.
- `offset.storage.replication.factor` : `offset.storage.topic` 의 복제 팩터로 최소 3이상 설정이 필요하다.
- `status.storage.replication.factor` : `status.storage.topic` 의 복제 팩터로 최소 3이상 설정이 필요하다.
- `offset.storage.partitions` : `offset.storage.topic` 의 파티션 수로 데이터가 최소 5이상 많다면 25이상 지정하는게 좋다. 
- `status.storage.partitions` : `status.storage.topic` 의 파티션 수로 비교적 데이터가 적기 때문에 5이상 지정하는게 좋다. 


### Demo
앞서 알아본 내용을 바탕으로 `Kafka Connect Cluster` 를 사용해 고가용성이 고려된 구성을 만들어본다. 
`Kafka Connector` 는 `Debezium Mysql Source Connect` 를 사용한다. 
데모의 전체 내용은 [여기]()
에서 확인 할 수 있다.  

데모 환경을 도식화하면 아래와 같다.  

.. 그림 ..

아래는 데모의 전체 구성 내용이 담겨 있는 `docker-compose.yaml` 의 파일 내용이다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      - exam-net

  kafka:
    container_name: myKafka
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper
    networks:
      - exam-net

  kafka-connect-1:
    container_name: kafka-connect-1
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8083:8083"
    command: sh -c "
      echo 'waiting 10s' &&
      sleep 10 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'exam-cluster'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'exam-cluster-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'exam-cluster-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'exam-cluster-status'
      CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect-1
      # kafka cluster 가 단일로 구성돼 1로 설정, ha 를 위해서는 replication.factor 는 3이상 필요
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_PARTITIONS: '26'
      CONNECT_STATUS_STORAGE_PARTITIONS: '6'
    depends_on:
      - exam-db
      - zookeeper
      - kafka
    networks:
      - exam-net

  kafka-connect-2:
    container_name: kafka-connect-1
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8084:8083"
    command: sh -c "
      echo 'waiting 10s' &&
      sleep 10 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'exam-cluster'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'exam-cluster-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'exam-cluster-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'exam-cluster-status'
      CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect-2
      # kafka cluster 가 단일로 구성돼 1로 설정, ha 를 위해서는 replication.factor 는 3이상 필요
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_PARTITIONS: '26'
      CONNECT_STATUS_STORAGE_PARTITIONS: '6'
    depends_on:
      - exam-db
      - zookeeper
      - kafka
    networks:
      - exam-net

  exam-db:
    container_name: exam-db
    image: mysql:8
    ports:
      - "3306:3306"
    volumes:
      - ./mysql.cnf:/etc/mysql/conf.d/custom.cnf
      - ./init-sql/:/docker-entrypoint-initdb.d/
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw
    networks:
      - exam-net

networks:
  exam-net:
```  

명시적으로 컨테이너는 `kafka-connect-1`, `kafka-connect-1` 로 분리해서 구성했다. 
하지만 환경변수를 사용해 `group.id` 를 비롯해 `config.storage.topic` 등 `Kafka Connect Cluster` 구성에 필요한 내용들은 동일하게 설정했다. 
이는 테스트를 위해 컨테이너를 각 별도로 구성한 것이므로 실제 환경에서는 `Kubernetes` 등과 같은 `Orchestration` 을 사용해 `Replicas` 등을 조절해서 `Kafka Connect Cluster` 를 구성하는 것도 가능하다.  

이제 `docker-compose up --build` 명령으로 전체 구성을 실행한다.  

```bash
$ docker-compose up --build
[+] Building 2.8s (13/13) FINISHED                                                                      0.0s
[+] Running 1/0
 ⠋ Container myZookeeper                                                                                                                                    Creating0.0s 
[+] Running 6/3am-db                                                                                               
 ✔ Container myZookeeper                                                                                                                                    Created0.0s d64) does not match the detected host platform (linux/arm64/v8
 ✔ Container exam-db                                                                                                                                        Created0.0s 
 ! zookeeper The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested 0.0s 
 ✔ Container myKafka                                                                                                                                        Created0.0s 
 ✔ Container kafka-connect-1                                                                                                                                Created0.0s 
 ✔ Container kafka-connect-2                                                                                                                                Created0.1s 
Attaching to exam-db, kafka-connect-1, kafka-connect-2, myKafka, myZookeeper

...
```  
