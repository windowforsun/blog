--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Debezium MySQL CDC Source Connector"
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
toc: true
use_math: true
---  

## CDC
`Application` 를 개별 역할을 수행하는 컴포넌트 별로 나누어 구성하는 `MSA(Micro Service Architecture)` 에서 
`EDA(Event Driven Architecture`) 로 발전하기 위해서는 신뢰성있는 `Event Bus` 를 구축하는 것이 필수이다. 
`Kafka` 를 통해 `Event Bus` 로 사용해서 데이터 인입에 따라 전체 컴포넌트들이 유기적으로 동작 할 수 있는 `CDC` 에 대해 간단하게 알아본다. 

`CDC` 는 `Change Data Capture` 의 약자로 소스가 되는 데이터의 변경을 식별해서 필요한 후속처리를 자동화 하는 기술 또는 설계 기법이자 구조를 의미한다. 
그리고 `CDC` 를 사용 했을 때 장점을 몇가지 더 나열 하면 아래와 같다. 

- `DataSource` 의 `changelog` 를 통해 변경 데이터를 분석 하기 때문에 변경사항이 100% 보장 될 수 있다. 
- 쿼리를 통해 데이터를 추출하지 않기 때문에 `DataSource` 부하가 적다. 
- 쿼리를 통한 `Polling` 방식 데이터 추출의 경우 최종 데이터 변경 내용만 조회 될 수 있어, 모든 데이터 변경 내용 캡쳐가 사실상 어렵다. `CDC` 는 `changelog` 기반으로 하기 때문에 모든 변경 사항 추적이 가능하다. 
- 위 내용을 바탕으로 삭제 및 스키마 변경과 같은 것들도 추적이 가능하다. 
- 실시간 스트리밍 방식으로 처리 할 수 있다. 

본 예제에서는 `Kafka Connect` 와 친화적인 오픈소스 `Debezium` 을 통해 `CDC Source Connectr` 를 간단하게 구성해보고, 
기본 동작에 대해서도 살펴 본다.  

예제 진행은 `Standalone Mode` 가 아닌, `Distribueted Mode` 로 진행되는 점 기억해야 한다. 


### Debezium MySQL Source Connector
`Debezium MySQL Source Connector` 는 `MySQL` 의 `binlog` 를 추적해서 `CDC` 동작을 수행한다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-mysql-cdc-2.png)  

예제 진행을 위해 필요한 디렉토리 구조와 파일은 아래와 같다. 

```
.
├── debezium-mysql-source-connector
│   └── Dockerfile
├── docker-compose.yaml
└── mysql.cnf
```  

`Debezium MySQL Source Connector` 사용을 위해서는 별도 도커 이미지 빌드가 필요하다. 
이미지 생성을 위해 아래와 같은 `Dockerfile` 을 작성해 준다.  

```dockerfile
FROM confluentinc/cp-kafka-connect-base:6.1.1

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components" \
    CUSTOM_SMT_PATH="/usr/share/java/custom-smt" \
    CUSTOM_CONNECTOR_MYSQL_PATH="/usr/share/java/custom-connector-mysql"

ARG CONNECT_TRANSFORM_VERSION=1.4.0
ARG DEBEZIUM_VERSION=1.5.0

# Download Using confluent-hub
RUN confluent-hub install --no-prompt confluentinc/connect-transforms:$CONNECT_TRANSFORM_VERSION

# Download Custom Source Connector
RUN mkdir $CUSTOM_CONNECTOR_MYSQL_PATH && cd $CUSTOM_CONNECTOR_MYSQL_PATH && \
    curl -sO https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/$DEBEZIUM_VERSION.Final/debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip && \
    jar xvf debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip && \
    rm debezium-connector-mysql-$DEBEZIUM_VERSION.Final-plugin.zip

```  

위 파일이 위치한 경로에서 아래 명령을 통해 이미지 빌드를 수행한다.  

```bash
$ docker build -t debezium-mysql-source-connector . 
```  

그리고 `MySQL` 을 `Source` 로 `Source Connector` 와 연결을 위해서는 설정이 필요한데, 
해당 설정이 있는 `MySQL` 설정 파일인 `mysql.cnf` 파일 내용은 아래와 같다.  

```
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
log_slave_updates = ON
binlog_row_image = FULL
# 필수는 아님
expire_logs_days = 0
# slave 인 경우에만 필수
#log_slave_updates = 1
```  

예제를 위한 전체 구성 셋을 `docker` 를 기반으로 올리는 작업만 남아 있다. 
전체 구성 셋이 작성된 `docker-compose.yaml` 파일 내용은 아래와 같다.  

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

  mySourceConnector:
    container_name: mySourceConnector
    image: localhost/debezium-mysql-source-connector
    ports:
      - "8083:8083"
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

  originDB:
    container_name: originDB
    image: mysql:8
    ports:
      - "3306:3306"
    volumes:
      - ./mysql.cnf:/etc/mysql/conf.d/custom.cnf
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw

```  

필요한 구성은 모두 마무리 되었다. 
이제 아래 명령으로 구성한 전체 구성을 실행 시킨다.  

```bash
$ docker-compose up --build
[+] Running 4/3
 ⠿ Container myZookeeper        Created                                                                            0.1s
 ⠿ Container originDB           Created                                                                            0.1s
 ⠿ Container myKafka            Created                                                                            0.1s
 ⠿ Container mySourceConnector  Created                                                                            0.0s
Attaching to myKafka, mySourceConnector, myZookeeper, originDB

.. 생략 ..
```  

`MySQL` 컨테이너인 `originDB` 에 접속해서 `mysql.cnf` 파일을 통해 설정한 값들이 정상적으로 설정 됐는지 확인하면 아래와 같다.  

```bash
.. originDB mysql 로그인 ..
$ docker exec -it originDB mysql -uroot -proot

mysql> SHOW VARIABLES LIKE 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.01 sec)

mysql> SHOW BINARY LOGS;
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       180 | No        |
| mysql-bin.000002 |   3033166 | No        |
| mysql-bin.000003 |       157 | No        |
+------------------+-----------+-----------+
3 rows in set (0.00 sec)

mysql> SHOW GLOBAL VARIABLES LIKE 'read_only';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| read_only     | OFF   |
+---------------+-------+
1 row in set (0.01 sec)

mysql> SHOW GLOBAL VARIABLES LIKE 'binlog_row_image';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| binlog_row_image | FULL  |
+------------------+-------+
1 row in set (0.00 sec)

mysql> SHOW GLOBAL VARIABLES LIKE 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.01 sec)

mysql> SHOW GLOBAL VARIABLES LIKE 'expire%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
1 row in set (0.00 sec)

```  

이어서 초기 데이터 설정을 위해 `user` 데이터베이스를 생성하고 필요한 테이블과 로우를 추가한다.  

```bash
mysql> create database user;
Query OK, 1 row affected (0.02 sec)

mysql> use user;
Database changed

mysql> create table user_account (
    ->    uid int,
    ->    name varchar(255)
    -> );
Query OK, 0 rows affected (0.04 sec)

mysql> create table user_role (
    -> account_id int,
    -> roll varchar(255)
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> insert into user_account(uid, name) values(1, 'jack');
Query OK, 1 row affected (0.01 sec)

mysql> insert into user_role(account_id, role) values(1, 'normal');
Query OK, 1 row affected (0.01 sec)

mysql> select * from user_account;
+------+------+
| uid  | name |
+------+------+
|    1 | jack |
+------+------+
1 row in set (0.00 sec)

mysql> select * from user_role;
+------------+--------+
| account_id | role  |
+------------+--------+
|          1 | normal |
+------------+--------+
1 row in set (0.00 sec)

```  

다음으로 `Debezium MySQL Source Connector` 인 `mySourceConnect` 의 `8083` 포트로  플러그인 목록 요청을 보내 `Kafka Connect` 구성이 잘 됐는지 확인 해본다.  

```bash
$ curl -X GET http://localhost:8083/connector-plugins | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   599  100   599    0     0    451      0  0:00:01  0:00:01 --:--:--   453
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

사용할 수 있는 플러그인에 `io.debezium.connector.mysql.MySqlConnector` 가 있는 걸로 보아 정상적으로 구성된 것을 확인 할 수 있다. 
이어서 `Kafka Connect REST API` 를 사용해서 `Debezium MySQL Source Connector` 를 실행 시킨다.  

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

`Debezium MySQL Source Connector` 실행에 사용한 설정 필드에 대한 설명은 [여기](https://debezium.io/documentation/reference/2.1/connectors/mysql.html#mysql-connector-properties) 
에서 확인 가능하다.  

`Kafka Connect REST API` 를 사용해서 실행한 커넥터가 정상동작 하고 있는지 확인하면 아래와 같다.  

```bash
$ curl -X GET http://localhost:8083/connectors\?expand\=status | jq

{
  "my-source-connector": {
    "status": {
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
  }
}
```  

커넥터도 정상동작이 확인 되었기 때문에 `Kafka Cluster` 에 접속해서 생성된 토픽을 확인하면 아래와 같다.  

```bash
$ docker exec -it myKafka kafka-topics.sh --bootstrap-server localhost:9092 --list
__consumer_offsets
_schemas
my-source-connector-config
my-source-connector-history
my-source-connector-offset
my-source-connector-status
originDB
originDB.user.user_account
originDB.user.user_role
```  

생성된 토픽에 대한 설정은 아래와 같다. 

- `_schemas` : `SchemaRegistry` 에서 커넥터 스키마 정보를 관리하는 토픽
- `my-source-connector-config` : 커넥터 설정 관리 데이터
- `my-source-connector-history` : `MySQL binlog` 의 데이터베이스 스키마 상태(커넥터 재시작시 사용)
- `my-source-connector-offset` : 커넥터 오프셋 관리 데이터
- `my-source-connector-status` : 커넥터 상태 관리 데이터
- `originDB` : 스키마 DDL 변경 데이터
- `originDB.user.user_account` : `user` 데이터베이스 `user_account` 테이블 변경 데이터
- `originDB.user.user_role` : `user` 데이터베이스 `user_role` 테이블 변경 데이터

스키마 `DDL` 변경 데이터 토픽인 `originDB` 를 컨슈머로 조회하면 아래와 같이 생성한 2개 테이블 정보를 확인할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic originDB --from-beginning

1.5.0.Final
mysqloriginDB����true mysql-bin.000003��SET character_set_server=utf8mb4, collation_server=utf8mb4_0900_ai_ci
1.5.0.Final
mysqloriginDB����truuseruser_account mysql-bin.000003userBDROP TABLE IF EXISTS user_account
1.5.0.Final
mysqloriginDB����truuseruser_role mysql-bin.000003user<DROP TABLE IF EXISTS user_role
1.5.0.Final
mysqloriginDB����truuser mysql-bin.000003user<DROP DATABASE IF EXISTS `user`
1.5.0.Final
mysqloriginDB����truuser mysql-bin.000003user�CREATE DATABASE `user` CHARSET utf8mb4 COLLATE utf8mb4_0900_ai_ci
1.5.0.Final
mysqloriginDB����truuser mysql-bin.000003userUSE `user`
1.5.0.Final
mysqloriginDB����truuseruser_account mysql-bin.000003user�CREATE TABLE `user_account` (
  `uid` int DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
                                                                  CREATE*"user"."user_account"utf8mb4uiINTINnameVARCHARVARCHARutf8mb4�
1.5.0.Final
mysqloriginDB����truuseruser_role mysql-bin.000003user�CREATE TABLE `user_role` (
  `account_id` int DEFAULT NULL,
  `role` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
                                                                  CREATE$"user"."user_role"utf8mb4account_iINTINroleVARCHARVARCHARutf8mb4�

```  

그리고 `user` 데이터베이스의 `user_account`, `user_role` 테이블 토픽을 컨슈머로 조회하면 추가한 데이터가 토픽에 들어 있는 것을 확인 할 수 있다.  

```bash

```  


> cdc 토픽 데이터 스키마 어떻하지 ? schema registry 를 안쓰면 토픽 내부 데이터가 너무 크고, schemaregistry 를 쓰면 토픽에 데이터가 잘 안들어가는 듯한 느낌 .. ? ㅠㅠ























---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  
[robcowart/cp-kafka-connect-custom](https://github.com/robcowart/cp-kafka-connect-custom)  
[conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose)  
