--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 1"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT와 HoistField, ValueToKey, InsertField, Cast, Drop 예제를 살펴보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
    - Kafka Connector
    - Transforms
    - SMT
    - HoistField
    - ValueToKey
    - InsertField
    - Cast
    - Drop
toc: true
use_math: true
---

## Kafka Connect Transforms
`Kafka Connect` 에서 `Transforms` 는 `Single Message Transforms(SMT)` 기능 제공을 의미한다. 
`Kafka Connect` 는 데이터 스트리밍 플랫폼 `Apache Kafka` 의 주요 구성요소로, 
다양한 데이터 소스와 싱크로 데이터를 쉽게 이동할 수 있도록 도와준다. 
이러한 과정에서 요구될 수 있는 데이터 변환 혹은 필터링에 필요한 몇가지 기능을 제공한다.  

`SMT` 는 `Kafka Connect` 에서 단일 메시지 수준에서 변환 작업을 수행하는 기능을 의미한다. 
`SMT` 는 `Kafka` 에 데이터가 작성되기 전이나 데이터 소스에서 읽어 올 때, 
혹은 싱크에게 데이터를 쓰기 전에 변환/필터링이 필요한 경우 사용 할 수 있다.  

`Kafka Connect` 의 개념과 기본적인 사용법은 [여기](https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/?session_ref=https://www.google.com/&_gl=1*1vjg9z4*_ga*MjA0NzkyNTk2MC4xNzAxMzgyMDQ3*_ga_D2D3EGKSGD*MTcxNzkzMjc5My44Mi4xLjE3MTc5MzM0NjAuNTQuMC4w&_ga=2.7783026.1766080457.1717921898-2047925960.1701382047)
를 통해 확인 할 수 있다.  

또한 기본적으로 제공되지 않는 변환/필터링의 경우 [Custom Transforms](https://docs.confluent.io/platform/current/connect/transforms/custom.html)
를 통해 자체 구현 할 수도 있다.  

포스팅에서는 몇가지 `SMT` 의 역할과 동작에 대해서 데모 구성을 통해 사용법과 동작 결과에 대해 알아볼 것이다.  


### Demo
데모 구성은 사용할 `Kafka Connect` 이미지를 직접 빌드하고, 전체 구성을 `docker-compose` 를 사용해서 구동한다. 
아래는 사용할 `Kafka Connect` 이미지를 빌드하는 `Dockerfile` 내용이다.  

```dockerfile
FROM confluentinc/cp-kafka-connect-base:7.0.10

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components,/usr/share/filestream-connectors" \
    CUSTOM_SMT_PATH="/usr/share/java/custom-smt"

ARG CONNECT_TRANSFORM_VERSION=1.4.4

# Download Using confluent-hub
RUN confluent-hub install --no-prompt confluentinc/connect-transforms:$CONNECT_TRANSFORM_VERSION

CMD ["/bin/bash", "-c", "/etc/confluent/docker/run"]
```  

`Kafka Connect` 7버전 부터는 기본적으로 사용할 수 있었던 `plugin` 들을 바로 사용 할 수는 없다. 
그래서 테스트로 `FileStreamSource` 를 사용하기 위해 `CONNECT_PLUGIN_PATH` 에 `/usr/share/filestream-connectors` 경로를 추가해 주었다. 
그리고 `io.confluent.connect` 로 시작하는 `SMT` 사용을 위해 `confluentic/connect-transforms` 도 설치해 주었다.  

위 `Dockerfile` 을 `docker build` 명령을 통해 이미지로 빌드한다.  

```bash
$ docker build -t file-source:7.0.10 .

$ docker image ls | grep file-source
file-source                                             7.0.10                         f02926cddeec   2.31GB
```  

전체 데모 구성을 살행하기 위해 아래와 같은 `docker-compose` 파일을 사용한다.  

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

  file-source:
    container_name: file-source
    image: file-source:7.0.10
    ports:
      - "8083:8083"
    environment:
      CONNECT_GROUP_ID: 'file-source-cluster'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'file-source-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'file-source-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'file-source-status'
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_REST_ADVERTISED_HOST_NAME: file-source
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
    volumes:
      - ./data:/data
```  

데모 구성은 `Kafka` 와 `Zookeeper` 그리고 `Kafka Connect` 인 `file-source` 로 최대한 간단하게 구성했다. 
`file-source` 는 `FileStreamSourceConnector` 를 사용해 예제가 진행된다. 
그래서 `FileStreamSourceConnector` 가 읽을 파일 마운트를 `./data:/data` 와 같이 해주었다.   

위 `docker-compose` 파일을 `docker-compose up --build` 명령으로 실행하고, 
`Kafka Connect` 가 정상 실행됐는지 사용 가능한 플러그인들을 조회하면 아래와 같다.  

```bash
$ docker-compose up --build

$ curl localhost:8083/connector-plugins | jq
[
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

그러면 위와 같이 사용에 필요한 `FileStreamSourceConnector` 가 목록에 있는 것을 확인 할 수 있다. 
여기서 `Kafka Connect` 에게 `HTTP` 요청을 보내 `Kafka Connector` 를 실행 할 수 있다. 


`FileStreamSourceConnect` 가 읽을 `input.txt` 에는 아래와 같은 내용이 작성돼 있다.

```
111
222
333
444
555
```  

그리고 아래는 `FileStreamSourceConnector` 를 실행하는 간단한 예시를 담고 있는 `file-source.json` 의 내용이다. 
현재 `Kafka Connect` 의 `key/value` 의 `Converter` 는 `StringConverter` 로 지정돼 있다. 
아래와 같이 별도로 `Converter` 를 지정해주지 않으면 `Kafka Connect` 에 설정된 기본 값으로 수행된다. 

```json
{
  "name": "file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/input.txt",
    "topic" : "file-source-topic"
  }
}
```  


`file-source.json` 을 사용해서 `Kafka Connect` 에게 `Kafka Connector` 실행 요청을 아래와 같이 전송한다.  

```bash
curl -X POST -H "Content-Type: application/json" \
--data @connector-json/file-source.json \
http://localhost:8083/connectors | jq

{
  "name": "file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/input.txt",
    "topic": "file-source-topic",
    "name": "file-source"
  },
  "tasks": [],
  "type": "source"
}

```  

그리고 실행한 `Kafka Connector` 상태를 확인하면 아래와 같이 정상 동작중인 것을 확인 할 수 있다.  

```bash
$ curl -X GET localhost:8083/connectors/file-source/status | jq

{
  "name": "file-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "file-source-1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "file-source-1:8083"
    }
  ],
  "type": "source"
}
```  

실행한 `Kafka Connector` 의 동작 결과는 `Kafka Topic` 을 구독해서 메시지를 확인하는 식으로 진행한다. 
아래 결과는 `value.converter` 가 `StringConverter` 로 돼있기 때문에 문자열 그대로 값이 토픽으로 전송된 것이다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    111
NO_HEADERS      null    222
NO_HEADERS      null    333
```  

아래와 같이 `value.converter` 를 `JsonConverter` 로 지정해서 스키마를 활성화해 실행할 수도 있다.  

```json
{
  "name": "file-source-json",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/input.txt",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "topic" : "file-source-json-topic"
  }
}
```  

이를 위와 동일한 방법으로 실행하고 토픽의 메시지를 확인해보면 스키마까지 포함된, 
구조화된 구조체(`Struct`) 타입으로 메시지가 전소된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-json-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"string","optional":false},"payload":"111"}
NO_HEADERS      null    {"schema":{"type":"string","optional":false},"payload":"222"}
NO_HEADERS      null    {"schema":{"type":"string","optional":false},"payload":"333"}
```

사용을 완료한 `Kafka Connector` 는 아래 요청으로 삭제할 수 있다.  

```bash
$ curl -X DELETE localhost:8083/connectors/file-source
$ curl -X DELETE localhost:8083/connectors/file-source-json
```

이제 기본으로 제공하는 각 `Transforms` 에는 무엇이 있고 어떻게 사용하며 어떠한 결과가 나오는지 살펴 볼 것이다.   

### HoistField
[HoistField](https://docs.confluent.io/platform/current/connect/transforms/hoistfield.html)
는 데이터에 스키마가 있는 경우 지정한 필드 이름을 사용해서, 
데이터를 구조체(`Struct`)에 래핑한다. 
그리고 데이터 스키마가 없는 경우에는 지정한 필드 이름을 사용해 데이터를 맵(`Map`)에 래핑한다. 
아래와 같이 메시지 키/값을 지정해서 모두 사용 할 수 있다.  

- Key : `org.apache.kafka.connect.transforms.HoistField$Key`
- Value : `org.apache.kafka.connect.transforms.HoistField$Value`

`hoist-field.txt` 의 내용은 아래와 같다.  

```
111
222
333
444
```  

#### HoistField Struct

이를 데이터 스키마가 없는 경우 값을 `message` 라는 필드 이름으로 래핑하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-hoist-field-map",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/hoist-field-input.txt",
    "topic" : "file-source-hoist-field-map-topic",
    "value.converter.schemas.enable": "false",
    "transforms" : "HoistFieldExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message"
  }
}
```  

결과 토픽을 구독해서 메시지를 확인하면 아래와 같다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-hoist-field-map-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=111}
NO_HEADERS      null    Struct{message=222}
```  


#### HoistField Map

이를 데이터 스키마가 없는 경우 값을 `message` 라는 필드 이름으로 래핑하는 `Json` 요청은 아래와 같다.

```json
{
  "name": "file-source-hoist-field-map",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/hoist-field-input.txt",
    "topic" : "file-source-hoist-field-map-topic",
    "value.converter.schemas.enable": "false",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message"
  }
}
```  

결과 토픽을 구독해서 메시지를 확인하면 아래와 같다.

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-hoist-field-map-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"message":"111"}
NO_HEADERS      null    {"message":"222"}
```  

#### HoistField Schemas

이제 데이터 스키마가 있는 경우 값을 `message` 라는 필드 이름으로 래핑하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-hoist-field",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/hoist-field-input.txt",
    "topic" : "file-source-hoist-field-topic",
    "value.converter.schemas.enable": "true",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message"
  }
}
```  

결과 토픽을 구독해서 메시지를 확인하면 아래와 같다.

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-hoist-field-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":false},"payload":{"message":"111"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":false},"payload":{"message":"222"}}
```  

### ValueToKey
[ValueToKey](https://docs.confluent.io/platform/current/connect/transforms/valuetokey.html)
는 메시지 레코드의 값의 필드를 키로 설정한다. 

- Value : `org.apache.kafka.connect.transforms.ValueToKey`

`value-to-key.input.txt` 의 내용은 아래와 같다.  

```
111
222
333
444
```  

파일로 부터 읽은 값을 `message` 라는 레코드로 구조화 시키고 이를 다시 키로 설정하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-value-to-key",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/value-to-key-input.txt",
    "topic" : "file-source-value-to-key-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ValueToKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message"
  }
}
```  

결과 토픽을 확인하면 `message` 레코드의 값이 키 값으로 설정된 것을 확인 할 수 있다.  


```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-value-to-key-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      Struct{message=111}     Struct{message=111}
NO_HEADERS      Struct{message=222}     Struct{message=222}
NO_HEADERS      Struct{message=333}     Struct{message=333}
NO_HEADERS      Struct{message=444}     Struct{message=444}
```  

### InsertField
[InsertField](https://docs.confluent.io/platform/current/connect/transforms/insertfield.html)
는 키 혹은 값에 정적인 값 혹은 제공하는 몇가지 속성을 추가할 수 있도록 한다. 
정적값인 `static` 을 포함해서 `offset`, `partition`, `timestamp`, `topic` 을 추가할 수 있다.  

- Key : `org.apache.kafka.connect.transforms.InsertField$Key`
- Value : `org.apache.kafka.connect.transforms.InsertField$Value`


`insert-field-static.input.txt` 의 내용은 아래와 같다.

```
111
222
333
444
```  

#### InsertField StaticField
정적값을 키와 값에 추가하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-insert-field-static",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/insert-field-static-input.txt",
    "topic" : "file-source-insert-field-static-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ValueToKeyExam,InsertFieldStaticValueExam,InsertFieldStaticKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.InsertFieldStaticValueExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldStaticValueExam.static.field" : "static_value_field",
    "transforms.InsertFieldStaticValueExam.static.value" : "static_value_value",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.InsertFieldStaticKeyExam.type" : "org.apache.kafka.connect.transforms.InsertField$Key",
    "transforms.InsertFieldStaticKeyExam.static.field" : "static_Key_field",
    "transforms.InsertFieldStaticKeyExam.static.value" : "static_Key_value"
  }
}
```  

결과 토픽을 확인하면 아래와 같이 키와 값에 정의한 필드와 값이 추가된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-insert-field-static-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      Struct{message=111,static_Key_field=static_Key_value}   Struct{message=111,static_value_field=static_value_value}
NO_HEADERS      Struct{message=222,static_Key_field=static_Key_value}   Struct{message=222,static_value_field=static_value_value}
NO_HEADERS      Struct{message=333,static_Key_field=static_Key_value}   Struct{message=333,static_value_field=static_value_value}
NO_HEADERS      Struct{message=444,static_Key_field=static_Key_value}   Struct{message=444,static_value_field=static_value_value}
```  

#### InsertField TimestampField
현재 타임스탬프 값을 추가하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-insert-field-timestamp",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/insert-field-timestamp-input.txt",
    "topic" : "file-source-insert-field-timestamp-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ValueToKeyExam,InsertFieldTimestampValueExam,InsertFieldTimestampKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.InsertFieldTimestampValueExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldTimestampValueExam.timestamp.field" : "timestamp_value",
    "transforms.InsertFieldTimestampKeyExam.type" : "org.apache.kafka.connect.transforms.InsertField$Key",
    "transforms.InsertFieldTimestampKeyExam.timestamp.field" : "timestamp_key"
  }
}
```  

결과 토픽을 확인하면 키와 값에 정의한 타임스탬프 필드 이름인 `timestamp_key`, `timestamp_value` 에 현재 타임 스탬프값이 설정된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-insert-field-timestamp-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      Struct{message=111,timestamp_key=Sat Jun 15 09:33:31 GMT 2024}  Struct{message=111,timestamp_value=Sat Jun 15 09:33:31 GMT 2024}
NO_HEADERS      Struct{message=222,timestamp_key=Sat Jun 15 09:33:31 GMT 2024}  Struct{message=222,timestamp_value=Sat Jun 15 09:33:31 GMT 2024}
NO_HEADERS      Struct{message=333,timestamp_key=Sat Jun 15 09:33:31 GMT 2024}  Struct{message=333,timestamp_value=Sat Jun 15 09:33:31 GMT 2024}
NO_HEADERS      Struct{message=444,timestamp_key=Sat Jun 15 09:33:31 GMT 2024}  Struct{message=444,timestamp_value=Sat Jun 15 09:33:31 GMT 2024}
```  

#### InsertField TopicField
`Kafka Connector` 가 목적지로 하는 `Topic` 이름을 추가하는 `Json` 요청은 아래와 같다.  

```json
{
  "name": "file-source-insert-field-topic",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/insert-field-topic-input.txt",
    "topic" : "file-source-insert-field-topic-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ValueToKeyExam,InsertFieldTopicValueExam,InsertFieldTopicKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.InsertFieldTopicValueExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldTopicValueExam.topic.field" : "topic_value",
    "transforms.InsertFieldTopicKeyExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldTopicKeyExam.topic.field" : "topic_key"
  }
}
```  

결과 토픽을 확인하면 키와 값에 `topic_key`, `topic_value` 라는 필드 이름으로 현재 토픽의 이름이 추가된 것을 확인 할 수 있다.  


```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-insert-field-topic-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      Struct{message=111,topic_key=file-source-insert-field-topic-topic}      Struct{message=111,topic_value=file-source-insert-field-topic-topic}
NO_HEADERS      Struct{message=222,topic_key=file-source-insert-field-topic-topic}      Struct{message=222,topic_value=file-source-insert-field-topic-topic}
NO_HEADERS      Struct{message=333,topic_key=file-source-insert-field-topic-topic}      Struct{message=333,topic_value=file-source-insert-field-topic-topic}
NO_HEADERS      Struct{message=444,topic_key=file-source-insert-field-topic-topic}      Struct{message=444,topic_value=file-source-insert-field-topic-topic}
```  

### Cast
[Cast](https://docs.confluent.io/platform/current/connect/transforms/cast.html) 
는 특정 키/값의 필드의 타입을 변경할 수 있다. 
`List`, `Map` 과 값은 복잡한 타입변경은 불가능하고, 
`int`, `string`, `boolean`, `float` 과 같은 원형 타입만 지원 가능하다.  

- Key : `org.apache.kafka.connect.transforms.Cast$Key`
- Value : `org.apache.kafka.connect.transforms.Cast$Value`

`cast-input.txt` 의 내용은 아래와 같다. 

```
111
222
333
444
```  

타입 변환을 정확하게 확인하기 위해 `Schema` 를 활성화하기 위해 `key`, `value` 의 `Converter` 를 `JsonConverter` 로 설정해서 진행한다.   

아래는 `message` 필드의 값을 키에서는 `float` 으로 값은 `int` 로 형변환하는 예제의 `Json` 요청이다.  

```json
{
  "name": "file-source-cast",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/cast-input.txt",
    "topic" : "file-source-cast-topic",
    "value.converter.schemas.enable": "true",
    "key.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam,ValueToKeyExam,CastValueExam,CastKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.CastValueExam.type" : "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.CastValueExam.spec" : "message:int32",
    "transforms.CastKeyExam.type" : "org.apache.kafka.connect.transforms.Cast$Key",
    "transforms.CastKeyExam.spec" : "message:float64"
  }
}
```  

결과 토픽을 확인하면 키에서 `message` 필드의 타입은 `float` 이고, 값에서 `message` 필드의 타입은 `int` 로 형변환이 된 것을 확인 할 수 있다.  

```bash
docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-cast-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"double","optional":false,"field":"message"}],"optional":false},"payload":{"message":111.0}}  {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"message"}],"optional":false},"payload":{"message":111}}
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"double","optional":false,"field":"message"}],"optional":false},"payload":{"message":222.0}}  {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"message"}],"optional":false},"payload":{"message":222}}
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"double","optional":false,"field":"message"}],"optional":false},"payload":{"message":333.0}}  {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"message"}],"optional":false},"payload":{"message":333}}
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"double","optional":false,"field":"message"}],"optional":false},"payload":{"message":444.0}}  {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"message"}],"optional":false},"payload":{"message":444}}
```  

### Drop
[Drop](https://docs.confluent.io/platform/current/connect/transforms/drop.html)
를 통해 메시지의 키 혹은 값은 제거해서 `null` 로 설정 할 수 있다. 
이때 스키마를 사용할 경우 스키마에 어떤 식으로 설정 할지도 선택할 수 있다. 

- Key : `io.confluent.connect.transforms.Drop$Key`
- Value : `io.confluent.connect.transforms.Drop$Value`

`drop-input.txt` 파일 내용은 아래와 같다.  

```
111
222
333
444
```

`null` 로 한 뒤 스키마는 어떻게 처리할 지는 아래 종류 중에 선택 가능하다.  

- `nullify` : 스키마도 `null` 로 만든다. 
- `retain` : 지우고 스키마는 그대로 유지한다. 
- `validate` : `optional` 인 경우 현재 스키마를 유지하고, `optional` 이 아니라면 예외를 발생시킨다. 
- `force_optional` : 스키마를 강제 `optional` 로 설정한다. 

아래 `Json` 요청은 키를 지울 때는 `force_optional` 을 통해 강제 `optional` 로 설정하고, 
값을 지울 때는 `nullify` 를 사용해서 스키마까지 `null` 로 만드는 요청의 예시이다.  

```json
{
  "name": "file-source-drop",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/drop-input.txt",
    "topic" : "file-source-drop-topic",
    "value.converter.schemas.enable": "true",
    "key.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam,ValueToKeyExam,DropKeyExam,DropValueExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.DropKeyExam.type" : "io.confluent.connect.transforms.Drop$Key",
    "transforms.DropKeyExam.schema.behavior" : "force_optional",
    "transforms.DropValueExam.type" : "io.confluent.connect.transforms.Drop$Value",
    "transforms.DropValueExam.schema.behavior" : "nullify"
  }
}
```  

결과 토픽을 확인하면 키의 경우 `force_optional` 로 설정했기 때문에 스키마가 `optional` 이 활성화 된 상태로 존재하고, 
값의 경우 `nullify` 로 설정했기 때문에 `null` 로만 나오는 것을 확인 할 수 있다.  

```bash
docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-drop-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":true},"payload":null}        null
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":true},"payload":null}        null
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":true},"payload":null}        null
NO_HEADERS      {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"message"}],"optional":true},"payload":null}        null
```


---  
## Reference
[Kafka Connect Single Message Transform](https://docs.confluent.io/platform/current/connect/transforms/overview.html)  
[How to Use Single Message Transforms in Kafka Connect](https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/?session_ref=https://www.google.com/&_gl=1*1vjg9z4*_ga*MjA0NzkyNTk2MC4xNzAxMzgyMDQ3*_ga_D2D3EGKSGD*MTcxNzkzMjc5My44Mi4xLjE3MTc5MzM0NjAuNTQuMC4w&_ga=2.7783026.1766080457.1717921898-2047925960.1701382047)  

