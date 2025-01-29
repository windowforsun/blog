--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect High Availability"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect Cluster 를 구성하고 고가용성을 보장하는 방법에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
    - High Availability
    - Distributed
    - Worker
    - Kafka Connector
    - Debezium
    - Mysql
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

모든 구성 실행이 완료 됐다면 아래 명령들로 모든 구성이 정상적으로 됐는지 확인 한다.  

```bash
.. Debezium Mysql Source Connector 에서 CDC 할 테이블 확인 ..
$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "show tables"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------------+
| Tables_in_exam |
+----------------+
| test_table     |
+----------------+

.. Kafka Connect Cluster 에서 구성한 Topic 확인 ..
$  docker exec -it myKafka \
> kafka-topics.sh --bootstrap-server localhost:9092 --list
__consumer_offsets
exam-cluster-config
exam-cluster-offset
exam-cluster-status

.. Kafka Connect Cluster Topic 상세 정보 확인 .. 
$  docker exec -it myKafka \
> kafka-topics.sh --bootstrap-server localhost:9092 --topic exam-cluster-config --describe
Topic: exam-cluster-config      TopicId: pFYK-SVtSZaqgrUoZmbF_g PartitionCount: 1       ReplicationFactor: 1    Configs: cleanup.policy=compact,segment.bytes=1073741824
        Topic: exam-cluster-config      Partition: 0    Leader: 1001    Replicas: 1001  Isr: 1001
        
$  docker exec -it myKafka \
> kafka-topics.sh --bootstrap-server localhost:9092 --topic exam-cluster-offset --describe
Topic: exam-cluster-offset      TopicId: 5ECYbCpTTH28D7vLam-qMg PartitionCount: 26      ReplicationFactor: 1    Configs: cleanup.policy=compact,segment.bytes=1073741824
        Topic: exam-cluster-offset      Partition: 0    Leader: 1001    Replicas: 1001  Isr: 1001
        Topic: exam-cluster-offset      Partition: 1    Leader: 1001    Replicas: 1001  Isr: 1001
        Topic: exam-cluster-offset      Partition: 2    Leader: 1001    Replicas: 1001  Isr: 1001
...

$  docker exec -it myKafka \
> kafka-topics.sh --bootstrap-server localhost:9092 --topic exam-cluster-status --describe
Topic: exam-cluster-status      TopicId: 9vj_-6TEQG-RTcs1z99K3A PartitionCount: 6       ReplicationFactor: 1    Configs: cleanup.policy=compact,segment.bytes=1073741824
        Topic: exam-cluster-status      Partition: 0    Leader: 1001    Replicas: 1001  Isr: 1001
        Topic: exam-cluster-status      Partition: 1    Leader: 1001    Replicas: 1001  Isr: 1001
        Topic: exam-cluster-status      Partition: 2    Leader: 1001    Replicas: 1001  Isr: 1001
...

.. kafka-connect-1 컨테이너 확인 ..
$  curl localhost:8083/connector-plugins | jq
[
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.0.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]

.. kafka-connect-2 컨테이너 확인 ..
$  curl localhost:8084/connector-plugins | jq
[
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.0.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]
```  

모든 구성이 정상적으로 실행된 것을 확인하면 `Kafka Connect` 에 실행 할 `Kafka Connector` 를 아래 요청으로 등록한다. 
사용할 `Kafka Connector` 는 `MySQL` 의 변경사항을 `CDC` 하는 `Debezium Mysql Source Connector` 이다. 
동일한 `group.id` 로 클러스터가 구성돼 있다면 `Kafka Connector` 등록 요청은 `Kafka Connect Cluster` 구성요소 중 하나에 한번만 수행하면된다. 
그러면 `Kafka Connect Cluster` 에서 구성 요소들을 바탕으로 작업을 분배하고 실행하게 된다. 

> Debezium Mysql Source Connector 의 경우 `MySQL` 의 트랜잭션 로그를 추적하는 `Source Connector` 이다 그러므로 `tasks.max` 는 1로만 설정해야 하고, 
> 그 이상 설정하면 에러가 발생한다. 


```bash
curl -X POST -H "Content-Type: application/json" \
--data '{
  "name": "cdc-exam",
  "config": {
    "tasks.max" : "1",
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "exam-db",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.name": "exam-db",
    "database.history.kafka.topic": "cdc-outbox",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.allowPublicKeyRetrieval" : "true",
    "tombstones.on.delete" : "false",
    "table.include.list": "exam.test_table",
    "value.converter" : "org.apache.kafka.connect.storage.StringConverter",
    "key.converter" : "org.apache.kafka.connect.storage.StringConverter"
  }
}' \
http://localhost:8083/connectors | jq

{
  "name": "cdc-exam",
  "config": {
    "tasks.max": "1",
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "exam-db",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.name": "exam-db",
    "database.history.kafka.topic": "cdc-outbox",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.allowPublicKeyRetrieval": "true",
    "tombstones.on.delete": "false",
    "table.include.list": "exam.test_table",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "name": "cdc-exam"
  },
  "tasks": [],
  "type": "source"
}
```  

그 후 `kafka-connect-1`, `kafka-connect-2` 에 모두 등록한 `Connector` 의 상태를 조회하면 아래와 같이 모두 정상적으로 조회되는 것을 확인 할 수 있다.  

```bash
$  curl localhost:8083/connectors/cdc-exam/status | jq
{
  "name": "cdc-exam",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect-1:8083"
    }
  ],
  "type": "source"
}

$  curl localhost:8084/connectors/cdc-exam/status | jq
{
  "name": "cdc-exam",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect-1:8083"
    }
  ],
  "type": "source"
}

```  

상태 조회결과에서 알 수 있듯이 `exam-cluster` 에서 방금 등록한 `Kafka Connector` 는 `kafka-connect-1` 에서 실행 중인 상태이다. 
이제 `CDC` 가 정상 동작하는지 확인을 위해 `test_table` 에 `ROW` 하나를 `INSERT` 하고 타겟이 되는 `Kafka Topic` 을 확인하면 아래와 같다.  

```bash
$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "insert into test_table(value) values('a')"
mysql: [Warning] Using a password on the command line interface can be insecure.

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from test_table"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+
| id | value |
+----+-------+
|  1 | a     |
+----+-------+

$  docker exec -it myKafka kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --topic exam-db.exam.test_table \
> --property print.key=true \
> --property print.headers=true \
> --from-beginning 
NO_HEADERS      Struct{id=1}    Struct{after=Struct{id=1,value=a},source=Struct{version=1.5.0.Final,connector=mysql,name=exam-db,ts_ms=1723372195000,db=exam,table=test_table,server_id=1,file=mysql-bin.000003,pos=375,row=0},op=c,ts_ms=1723372195833}
```  

`Debezium MySQL Source Connector` 가 `test_table` 의 변경사항을 감지하고 이를 `Kafka Topic` 에 정상적으로 전송한 것을 확인 할 수 있다. 
이제 `Kafka Connect Cluster` 의 고가용성 확인을 위해 `Kafka Connector` 가 실행 중인 `kafka-connect-1` 컨테이너를 중지 시킨 후 `kafka-connect-2` 에 상태 조회를 수행하면 아래와 같다.  

```bash
$  docker-compose -f ./docker/docker-compose.yaml stop kafka-connect-1
[+] Stopping 1/1
 ✔ Container kafka-connect-1  Stopped              
 
$  curl localhost:8084/connectors/cdc-exam/status | jq
{
  "name": "cdc-exam",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-2:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect-2:8083"
    }
  ],
  "type": "source"
}
```  

`kafka-connect-1` 컨테이너가 종료되자 `Rebalance` 가 수행되며 `Connector` 를 실행하는 `Worker` 가 `kafka-connect-2` 로 자동 할당된 것을 볼 수 있다. 
실제로 아래와 같은 로그가 발생한다.  

```bash
kafka-connect-1  | [2024-08-11 19:33:55,645] INFO Kafka Connect stopped (org.apache.kafka.connect.runtime.Connect)
kafka-connect-2  | [2024-08-11 19:33:56,019] INFO [Worker clientId=connect-1, groupId=exam-cluster] Rebalance started (org.apache.kafka.connect.runtime.distributed.WorkerCoordinator)
kafka-connect-2  | [2024-08-11 19:33:56,019] INFO [Worker clientId=connect-1, groupId=exam-cluster] (Re-)joining group (org.apache.kafka.connect.runtime.distributed.WorkerCoordinator)
myKafka          | [2024-08-11 10:33:56,021] INFO [GroupCoordinator 1001]: Stabilized group exam-cluster generation 5 (__consumer_offsets-0) with 1 members (kafka.coordinator.group.GroupCoordinator)
kafka-connect-2  | [2024-08-11 19:33:56,022] INFO [Worker clientId=connect-1, groupId=exam-cluster] Successfully joined group with generation Generation{generationId=5, memberId='connect-1-d24c7c49-2e9f-47ea-96e2-4232d3f39d64', protocol='sessioned'} (org.apache.kafka.connect.runtime.distributed.WorkerCoordinator)
myKafka          | [2024-08-11 10:33:56,040] INFO [GroupCoordinator 1001]: Assignment received from leader for group exam-cluster for generation 5. The group has 1 members, 0 of which are static. (kafka.coordinator.group.GroupCoordinator)
kafka-connect-2  | [2024-08-11 19:33:56,041] INFO [Worker clientId=connect-1, groupId=exam-cluster] Successfully synced group in generation Generation{generationId=5, memberId='connect-1-d24c7c49-2e9f-47ea-96e2-4232d3f39d64', protocol='sessioned'} (org.apache.kafka.connect.runtime.distributed.WorkerCoordinator)
kafka-connect-2  | [2024-08-11 19:33:56,041] INFO [Worker clientId=connect-1, groupId=exam-cluster] Joined group at generation 5 with protocol version 2 and got assignment: Assignment{error=0, leader='connect-1-d24c7c49-2e9f-47ea-96e2-4232d3f39d64', leaderUrl='http://kafka-connect-2:8083/', offset=4, connectorIds=[cdc-exam], taskIds=[cdc-exam-0], revokedConnectorIds=[], revokedTaskIds=[], delay=0} with rebalance delay: 0 (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
kafka-connect-2  | [2024-08-11 19:33:56,041] INFO [Worker clientId=connect-1, groupId=exam-cluster] Starting connectors and tasks using config offset 4 (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
kafka-connect-2  | [2024-08-11 19:33:56,043] INFO [Worker clientId=connect-1, groupId=exam-cluster] Starting connector cdc-exam (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
kafka-connect-2  | [2024-08-11 19:33:56,043] INFO [Worker clientId=connect-1, groupId=exam-cluster] Starting task cdc-exam-0 (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
kafka-connect-2  | [2024-08-11 19:33:56,046] INFO Creating task cdc-exam-0 (org.apache.kafka.connect.runtime.Worker)

...
```  

이제 다시 `test_table` 이 `ROW` 를 `INSERT` 하면, 중복 처리 없이 `kafka-connect-1` 에서 실행했던 다음 부터 정상 수행되는 것을 확인 할 수 있다.  

```bash
$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "insert into test_table(value) values('a')"
mysql: [Warning] Using a password on the command line interface can be insecure.

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from test_table"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+
| id | value |
+----+-------+
|  1 | a     |
|  2 | a     |
+----+-------+

$  docker exec -it myKafka kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --topic exam-db.exam.test_table \
> --property print.key=true \
> --property print.headers=true \
> --from-beginning 
NO_HEADERS      Struct{id=1}    Struct{after=Struct{id=1,value=a},source=Struct{version=1.5.0.Final,connector=mysql,name=exam-db,ts_ms=1723372195000,db=exam,table=test_table,server_id=1,file=mysql-bin.000003,pos=375,row=0},op=c,ts_ms=1723372195833}
NO_HEADERS      Struct{id=2}    Struct{after=Struct{id=2,value=a},source=Struct{version=1.5.0.Final,connector=mysql,name=exam-db,ts_ms=1723372668000,db=exam,table=test_table,server_id=1,file=mysql-bin.000003,pos=671,row=0},op=c,ts_ms=1723372668305}
```  
