--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect CDC MySQL to Elasticsearch"
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
    - Kafka Connect
    - CDC
    - Debezium
    - MySQL
    - Elasticsearch
toc: true
use_math: true
---  

## CDC MySQK to Elasticsearch
[Kafka Connect Debezium MySQL CDC Source Connector]({{site.baseurl}}{% link _posts/kafka/2023-01-19-kafka-practice-kafka-connect-debezium-mysql-cdc-source-connector.md %})
에서 `MySQL` 의 데이터 변경사항을 `Kafka Connect` 인 `Debezium MySQL Source Connector` 를 사용해서 
`Kafka` 로 전송하는 방법에 대해 알아보았다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-elasticsearch-sink-connector-1.drawio.png)


이번 포스트에서는 `Kafka Topic` 에 있는 `MySQL` 데이터를 
`Elasticsearch Sink Connector` 를 사용해서 `Elasticsearch` 까지 전송해서, 
최종적으로 `MySQL` 의 모든 데이터를 `Elasticsearch` 까지 이동시키는 방법에 대해 알아본다.  

[Kafka Connect Debezium MySQL CDC Source Connector]({{site.baseurl}}{% link _posts/kafka/2023-01-19-kafka-practice-kafka-connect-debezium-mysql-cdc-source-connector.md %})
에서 설명했지만 중요하므로 한번 더 구현하고자 하는 `CDC` 에 대해서 설명하면 아래와 같다. 

- `DataSource`(`MySQL`) 의 `changelog` 를 사용하기 때문에 변경사항이 100% 보장 될 수 있다.
- 쿼리로 데이터 추출하지 않기 때문에 `DataSource` 부하가 적다. 
- 쿼리기반 `Polling` 방식의 경우 최종 데이터 변경만 조회 되기 때문에, 모든 데이터 변경이 어렵지만 `CDC` 는 `changelog` 기반이므로 추적이 가능하다. 
- 스키마 변경에 대한 내용도 추적할 수 있다. 
- 실시간 스트리밍 방식으로 가능하다. 

## 예제
예제는 `docker` 와 `docker-compose` 를 기반으로 진행한다. 
필요한 구성은 아래와 같다. 

- `Zookeeper`
- `Kafka`
- `MySQL`
- `Elasticsearch`
- `Debezium MySQL Source Connector`
- `Elasticsearch Sink Connector`

전체 디렉토리 구조는 아래와 같다. 

```
.
├── docker-compose.yaml
├── elastic-sink-connector
│   └── Dockerfile
├── init-sql
│   └── init.sql
└── mysql.cnf
```  

### Debezium MySQL Source Connector`
[Dockerfile](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-connect-debezium-mysql-cdc-source-connector/#dockerfile)
와 동일하게 구성해 준다.  

### Elasticsearch Sink Connector
`Elasticsearch Sink Connector` 사용을 위해서는 별도 도커 이미지 빌드가 필요하다. 
이미지 생성을 위해 아래와 같은 `Dockerfile` 을 작성해야 한다.  

```dockerfile
FROM confluentinc/cp-kafka-connect-base:6.1.1

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components" \
    CUSTOM_SMT_PATH="/usr/share/java/custom-smt" \
    CUSTOM_CONNECTOR_MYSQL_PATH="/usr/share/java/custom-connector-mysql"

ARG CONNECT_TRANSFORM_VERSION=1.4.0
ARG ELASTIC_SEARCH_VERSION=14.0.5
ARG DEBEZIUM_VERSION=1.5.0

# Download Using confluent-hub
RUN confluent-hub install --no-prompt confluentinc/connect-transforms:$CONNECT_TRANSFORM_VERSION

# Download Elasticsearch sink connector
RUN confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:$ELASTIC_SEARCH_VERSION

# Download Custom Source Connector for transforms
RUN mkdir $CUSTOM_CONNECTOR_MYSQL_PATH && cd $CUSTOM_CONNECTOR_MYSQL_PATH && \
    curl -sO https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/$DEBEZIUM_VERSION.Final/debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip && \
    jar xvf debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip && \
    rm debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip
```  

위 파일이 위치한 경로에서 아래 명령으로 이미지를 빌드한다.  

```bash
$ docker build -t elastic-sink-connector .
```  


### docker-compose
`docker-compose.yaml` 파일에는 구현에 필요한 전체 구성이 담겨져 있다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"

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

  mySourceConnector:
    container_name: mySourceConnector
    image: localhost/debezium-mysql-source-connector
    ports:
      - "8083:8083"
    command: sh -c "
      echo 'waiting 30s' &&
      sleep 30 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'mySourceConnector'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'my-source-connector-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'my-source-connector-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'my-source-connector-status'
      CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_REST_ADVERTISED_HOST_NAME: mySourceConnector
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
    depends_on:
      - originDB
      - zookeeper
      - kafka

  mySinkConnector:
    container_name: mySinkConnector
    image: localhost/elastic-sink-connector
    ports:
      - "8084:8083"
    command: sh -c "
      echo 'waiting 40s' &&
      sleep 40 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'mySinkConnector'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'my-sink-connector-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'my-sink-connector-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'my-sink-connector-status'
      CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_REST_ADVERTISED_HOST_NAME: mySinkConnector
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
    depends_on:
      - originDB
      - zookeeper
      - kafka

  es-single:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.2.0
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

```  

`MySQL` 의 설정 파일인 `mysql.cnf` 내용은 아래와 같다. 

> `mysql.cnf` 파일 권한을 `chmod 755 mysql.cnf` 로 줘야 한다. `777` 이면 정상 반영이 되지 않을 수 있다. 

```
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
log_slave_updates = ON
binlog_row_image = FULL
# 필수는 아님
expire_logs_days = 0
# slave 인 경우에만
#log_slave_updates = 1
```  

`MySQL` 초기 데이터를 구성해 줄 `init-sql/init.sql` 내용은 아래와 같다.  

```sql
create database user;
use user;
create table user_account (
  uid int not null primary key,
  name varchar(255)
);
create table user_role (
   account_id int not null,
   user_role varchar(255) not null,
   primary key (account_id, user_role)
);
insert into user_account(uid, name) values(1, 'jack');
insert into user_role(account_id, user_role) values(1, 'normal');



create database admin;
use admin;
create table admin_account (
   uid int not null primary key,
   name varchar(255)
);
create table admin_role (
    account_id int not null,
    admin_role varchar(255) not null,
    primary key (account_id, admin_role)
);
insert into admin_account(uid, name) values(1, 'susan');
insert into admin_role(account_id, admin_role) values(1, 'developer');
```  

구성에 필요한 모든 설정이 완료된 상태이다. 
이제 `docker-compose.yaml` 파일이 위치한 경로에서 아래 명령으로 전체 구성을 실행 시킨다.  

```bash
$ docker-compose up --build
[+] Running 6/3
 ⠿ Container es-single          Created                                                                                                                                                                                            0.0s
 ⠿ Container originDB           Created                                                                                                                                                                                            0.1s
 ⠿ Container myZookeeper        Created                                                                                                                                                                                            0.1s
 ⠿ Container myKafka            Created                                                                                                                                                                                            0.0s
 ⠿ Container mySinkConnector    Created                                                                                                                                                                                            0.0s
 ⠿ Container mySourceConnector  Created                                                                                                                                                                                            0.0s
Attaching to es-single, myKafka, mySinkConnector, mySourceConnector, myZookeeper, originDB

```  

### 실행 상태 확인
전체 구성이 정상적으로 실행 중인지 아래 명령들로 확인을 진행한다.  

- mySourceConnector(`Debezium MySQL Source Connector`) 확인 
  - `MySQLConnector` 가 존재하는지 확인 한다. 

```bash
$ curl -X GET http://localhost:8083/connector-plugins | jq

[
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.0.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "6.1.1-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "6.1.1-ccs"
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

- mySinkConnector(`Elasticsearch Sink Connector`)
  - `ElasticsearchSinkConnector` 가 존재하는지 확인 한다. 

```bash
$ curl -X GET http://localhost:8084/connector-plugins | jq

[
  {
    "class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "type": "sink",
    "version": "14.0.5"
  },
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.0.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "6.1.1-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "6.1.1-ccs"
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

- es-single(`Elasticsearch`)

```bash
$ curl -X GET http://localhost:9200 | jq

{
  "name": "es-single",
  "cluster_name": "es-single",
  "cluster_uuid": "Bq72RK1jSqyS-J3FwmsXWw",
  "version": {
    "number": "8.2.0",
    "build_flavor": "default",
    "build_type": "docker",
    "build_hash": "b174af62e8dd9f4ac4d25875e9381ffe2b9282c5",
    "build_date": "2022-04-20T10:35:10.180408517Z",
    "build_snapshot": false,
    "lucene_version": "9.1.0",
    "minimum_wire_compatibility_version": "7.17.0",
    "minimum_index_compatibility_version": "7.0.0"
  },
  "tagline": "You Know, for Search"
}

```  

- originDB(`MySQL`)

```bash
$ docker exec -it originDB mysql -uroot -proot -e "select 1";
mysql: [Warning] Using a password on the command line interface can be insecure.
+---+
| 1 |
+---+
| 1 |
+---+

```  

- 만약 특정 구성이 정상적으로 응답하지 못한다면 아래 명령으로 해당 컨테이너만 재실행 한다. 

```bash
$ docker restart <재실행 필요한 컨테이너 이름>
```  


### MySQL to Elasticsearch CDC 구성하기
가장 먼저 `Debezium MySQL Source Connector` 인 `mySourceConnector` 를 사용해서 
`user`, `admin` 데이터베이스에 있는 모든 테이블을 `Kafka` 로 전송하는 `Source Connector` 를 실행 시킨디ㅏ.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
"name": "my-source-connector",
"config": {
"connector.class": "io.debezium.connector.mysql.MySqlConnector",
"tasks.max": "1",
"database.hostname": "originDB",
"database.port": "3306",
"database.user": "root",
"database.password": "root",
"database.server.id": "1",
"database.allowPublicKeyRetrieval" : "true",
"database.server.name" : "originDB",
"database.include.list" : "user, admin",
"database.history.kafka.bootstrap.servers" : "kafka:9092",
"database.history.kafka.topic" : "my-source-connector-history",
"include.schema.changes" : "true"
}
}' \
http://localhost:8083/connectors | jq

{
  "name": "my-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "originDB",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.id": "1",
    "database.allowPublicKeyRetrieval": "true",
    "database.server.name": "originDB",
    "database.include.list": "user, admin",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "my-source-connector-history",
    "include.schema.changes": "true",
    "name": "my-source-connector"
  },
  "tasks": [],
  "type": "source"
}
```  

정상적으로 `Source Connector` 가 실행 됐는지 `REST API` 와 `Kafak Topic` 으로 확인 해준다.  

```bash
$ curl -X GET http://localhost:8083/connectors/my-source-connector/status | jq

{
  "name": "my-source-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "mySourceConnector:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "mySourceConnector:8083"
    }
  ],
  "type": "source"
}

.. originDB 로 시작하는 토픽이 아래와 같이 존재해야 한다. ..
$ docker exec -it myKafka kafka-topics.sh --bootstrap-server localhost:9092 --list

__consumer_offsets
my-sink-connector-config
my-sink-connector-offset
my-sink-connector-status
my-source-connector-config
my-source-connector-history
my-source-connector-offset
my-source-connector-status
originDB
originDB.admin.admin_account
originDB.admin.admin_role
originDB.user.user_account
originDB.user.user_role

```  

그리고 `Elasticsearch Sink Connector` 인 `mySinkConnector` 를 사용해서 
`Kafka` 에 있는 `MySQL` 데이터중 `user` 데이터베이스의 `user_account` 테이블 데이터를 
`Elasticsearch` 로 전송하는 `Sink Connector` 를 실행한다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
"name": "my-sink-connector-user-account",
"config": {
"connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
"tasks.max": "1",
"topics": "originDB.user.user_account",
"connection.url": "http://es-single:9200",
"transforms": "unwrap,key",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.key.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
"transforms.key.field": "uid",
"key.ignore": "false",
"type.name": "user_account"
}
}' \
http://localhost:8084/connectors | jq

{
  "name": "my-sink-connector-user-account",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "originDB.user.user_account",
    "connection.url": "http://es-single:9200",
    "transforms": "unwrap,key",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.key.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.key.field": "uid",
    "key.ignore": "false",
    "type.name": "user_account",
    "name": "my-sink-connector-user-account"
  },
  "tasks": [],
  "type": "sink"
}
```  

위 `Sink Connector` 요청에서 주의해야 할 점은 `transforms.key.field` 필드의 값은 테이블의 컬럼 값인데, 
`PK` 로 구성된 필드가 와야하고, `PK` 가 아닌 필드를 추가하기 위해서는 추가적인 변환처리가(`transforms`) 필요하다.  


실행시킨 `my-sink-connector-user-account` 가 정상 동작 중인지는 `REST API` 요청과 
`Elasticsearch` 의 조회로 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:8084/connectors/my-sink-connector-user-account/status | jq

{
  "name": "my-sink-connector-user-account",
  "connector": {
    "state": "RUNNING",
    "worker_id": "mySinkConnector:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "mySinkConnector:8083"
    }
  ],
  "type": "sink"
}

.. es 인덱스 조회 ..
$ curl -X GET http://localhost:9200/_aliases\?pretty\=true

{
  "origindb.user.user_account" : {
    "aliases" : { }
  }
}

.. es 인덱스에 해당하는 모든 데이터 조회 ..
$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 8,
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
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      }
    ]
  }
}
```  

`uid : 1, name : jack` 인 데이터 하나가 조회되는 걸로 보아 정상적으로 `MySQL to Elasticsearch` 가 구성된 것을 확인 할 수 있다.  


이번엔 다른 데이터베이스인 `admin` 데이터베이스의 `admin_role` 테이블을 데이터를 `Elasticsearch` 로 옮기는 
`my-sink-connector-admin-role` 커넥터를 실행하고 결과를 확인하면 아래와 같다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
"name": "my-sink-connector-admin-role",
"config": {
"connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
"tasks.max": "1",
"topics": "originDB.admin.admin_role",
"connection.url": "http://es-single:9200",
"transforms": "unwrap,key",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.key.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
"transforms.key.field": "account_id",
"key.ignore": "false",
"type.name": "admin_role"
}
}' \
http://localhost:8084/connectors | jq

{
  "name": "my-sink-connector-admin-role",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "originDB.admin.admin_role",
    "connection.url": "http://es-single:9200",
    "transforms": "unwrap,key",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.key.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.key.field": "account_id",
    "key.ignore": "false",
    "type.name": "admin_role",
    "name": "my-sink-connector-admin-role"
  },
  "tasks": [],
  "type": "sink"
}

$ curl -X GET http://localhost:9200/_aliases\?pretty\=true

{
  "origindb.user.user_account" : {
    "aliases" : { }
  },
  "origindb.admin.admin_role" : {
    "aliases" : { }
  }
}

$ curl -X GET http://localhost:9200/origindb.admin.admin_role/_search\?pretty | jq

{
  "took": 7,
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
        "_index": "origindb.admin.admin_role",
        "_id": "1",
        "_score": 1,
        "_source": {
          "account_id": 1,
          "admin_role": "developer"
        }
      }
    ]
  }
}
```  


### 테스트

#### Insert

```bash
mysql> insert into user_account values(2, 'susan');

$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 255,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "susan"
        }
      }
    ]
  }
}
```  


#### Update

```bash
mysql> update user_account set name = 'test' where uid = 2;

$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 129,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "test"
        }
      }
    ]
  }
}
```  

#### Delete
참고로 `Elasticsearch Sink Connector` 에서는 삭제 동작은 지원하지 않는다고 한다. 
[참고](https://github.com/DarioBalinzo/kafka-connect-elasticsearch-source/issues/46)

```bash
mysql> delete from user_account where uid = 1;

$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 129,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "test"
        }
      }
    ]
  }
}
```  

#### DDL

`DDL` 은 기존 `Elasticsearch` 데이터에는 반영되지 않고, 
`DDL` 이후 추가되는 데이터 부터 반영된다.  

```bash
mysql> alter table user_account add column test varchar(255) default '';

$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 129,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "test"
        }
      }
    ]
  }
}

mysql> alter table user_account change name name2 varchar(255);

$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 129,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "test"
        }
      }
    ]
  }
}


mysql> insert into user_account values(3, 'test22', 'test33');
$ curl -X GET http://localhost:9200/origindb.user.user_account/_search\?pretty | jq

{
  "took": 728,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 4,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "origindb.user.user_account",
        "_id": "originDB.user.user_account+0+0",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "2",
        "_score": 1,
        "_source": {
          "uid": 2,
          "name": "test"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "1",
        "_score": 1,
        "_source": {
          "uid": 1,
          "name": "jack"
        }
      },
      {
        "_index": "origindb.user.user_account",
        "_id": "3",
        "_score": 1,
        "_source": {
          "uid": 3,
          "name2": "test22",
          "test": "test33"
        }
      }
    ]
  }
}

```





---  
## Reference
[Streaming data changes in MySQL into ElasticSearch using Debezium, Kafka, and Confluent JDBC Sink Connector](https://medium.com/dana-engineering/streaming-data-changes-in-mysql-into-elasticsearch-using-debezium-kafka-and-confluent-jdbc-sink-8890ad221ccf)  
[Kafka Connect Elasticsearch Connector in Action](https://www.confluent.io/blog/kafka-elasticsearch-connector-tutorial/)  
[Streaming Data Changes from Your Database to Elasticsearch](https://debezium.io/blog/2018/01/17/streaming-to-elasticsearch/)  
[ExtractField](https://docs.confluent.io/platform/current/connect/transforms/extractfield.html#examples)  
[Single Message Transforms for Confluent Platform](https://docs.confluent.io/cloud/current/connectors/transforms/overview.html#single-message-transforms-for-cp)  
[New Record State Extraction](https://debezium.io/documentation/reference/stable/transformations/event-flattening.html)