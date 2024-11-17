--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 3rd"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT 중 ExtractTopic, Filter, TimestampRouter 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - ExtractTopic
    - Filter
    - TimestampRouter
toc: true
use_math: true
---

## Kafka Connect Transforms
[Kafka Connect Transforms(SMT) 1]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-1.md %}),
[Kafka Connect Transforms(SMT) 2]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-2.md %}),
에 이어서 추가적인 `Transforms` 의 사용법에 대해 알아본다. 
테스트를 위해 구성하는 환경와 방식은 위 첫번째 포스팅과 동일하다. 
테스트 및 예제 진행에 궁금한 점이 있다면 이전 포스팅에서 관련 내용을 확인 할 수 있다.  

### ExtractTopic
[ExtractTopic](https://docs.confluent.io/platform/current/connect/transforms/extracttopic.html)
을 사용하면 메시지에서 특정 필드를 추출해 토픽이름으로 지정 할 수 있다. 
하나의 필드만 지정 할 수도 있고, [JSON Path](https://github.com/json-path/JsonPath)
을 사용해서 충첩구조의 지정도 가능하다.  

- Key : `io.confluent.connect.transforms.ExtractTopic$Key`
- Value : `io.confluent.connect.transforms.ExtractTopic$Value`
- Header : `io.confluent.connect.transforms.ExtractTopic$Header`

`extract-topic-input.txt` 파일의 내용은 아래와 같다.  

```
exam-topic-1
exam-topic-2
exam-topic-3
exam-topic-1
exam-topic-2
exam-topic-3
```  

#### ExtractTopic Value
아래 `JSON` 요청은 파일에 있는 내용을 `message` 필드로 래핑 후 해당 필드의 값을 토픽이름으로 지정하는 예시이다. 


```json
{
  "name": "file-source-extract-topic-value",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/extract-topic-input.txt",
    "topic" : "file-source-extract-topic-value-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ExtractTopicValueExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ExtractTopicValueExam.type" : "io.confluent.connect.transforms.ExtractTopic$Value",
    "transforms.ExtractTopicValueExam.field" : "message"
  }
}
```  

결과가 담기는 총 3개의 토픽을 보면 `message` 필드의 값과 동일한 토픽에서 메시지들이 알맞게 출력되는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-1 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=exam-topic-1}
NO_HEADERS      null    Struct{message=exam-topic-1}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-2 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=exam-topic-2}
NO_HEADERS      null    Struct{message=exam-topic-2}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-3 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=exam-topic-3}
NO_HEADERS      null    Struct{message=exam-topic-3}
```  


#### ExtractTopic JSON Path
아래 `JSON` 요청은 중첩 구조를 위해 `HoistField` 를 2번 사용해서 `message2.message` 와 같은 중첩 구조를 만들었다. 
그리고 해당 중첩구조의 값을 토픽의 이름으로 지정하는 예시이다.  

```json
{
  "name": "file-source-extract-topic-jsonpath",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/extract-topic-input.txt",
    "topic" : "file-source-extract-topic-jsonpath-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,HoistFieldExam2,ExtractTopicValueExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.HoistFieldExam2.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam2.field" : "message2",
    "transforms.ExtractTopicValueExam.type" : "io.confluent.connect.transforms.ExtractTopic$Value",
    "transforms.ExtractTopicValueExam.field" : "$[\"message2\"][\"message\"]",
    "transforms.ExtractTopicValueExam.field.format": "JSON_PATH"
  }
}
```  

해당 예시 결과 또한 총 3개의 토픽에 담기게되는데 
`message2.message` 필드의 값과 동일한 토픽에 알맞게 메시지들이 출력되는 것을 확인 할 수 있다.  


```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-1 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-1}}
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-1}}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-2 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-2}}
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-2}}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic exam-topic-3 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-3}}
NO_HEADERS      null    Struct{message2=Struct{message=exam-topic-3}}
```  


### Filter(Confluent)
[Filter(Confluent)](https://docs.confluent.io/platform/current/connect/transforms/filter-confluent.html)
를 사용하면 `filter.condition` 을 만족하는 레코드를 포함하거나 제외시킬 수 있다. 
`filter.condition` 에 작성되는 내용은 [JSON Path](https://github.com/json-path/JsonPath)
술어로 각 레코드에 적용된다. 
`filter.condition` 과 일치하는 경우 `filter.type=include` 라면 해당 레코드는 포함시키고, 
`filter.type=exclude` 라면 해당 레코드는 제외된다. 
그리고 `missing.or.null.behavior` 속성 지정을 통해 레코드가 존재하지 않을 경우의 동작을 지정할 수 있다. 
기본적으론 레코드가 존재하지 않는다면 실패하게 된다. 

- Key : `io.confluent.connect.transforms.Filter$Key`
- Value : `io.confluent.connect.transforms.Filter$Value`

해당 `Transform` 은 `Confluent` 에서 지원하는 내용이므로 사용이 필요하다면 별도로 [설치](https://docs.confluent.io/confluent-cli/current/command-reference/connect/plugin/confluent_connect_plugin_install.html)
를 해줘야 한다.  

`filter-confluent-input.txt` 파일의 내용은 아래와 같다.  

```
100
500
200
600
300
700
400
800
0
```  

#### Filter(Confluent) Value
아래 `JSON` 요청은 값에서 `message` 필드의 값이 `500` 이상인 레코드만 포함하는 예제이다.  

```json
{
  "name": "file-source-filter-confluent",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/filter-confluent-input.txt",
    "topic" : "file-source-filter-confluent-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,CastExam,FilterConfluentExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.CastExam.type" : "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.CastExam.spec" : "message:int32",
    "transforms.FilterConfluentExam.type" : "io.confluent.connect.transforms.Filter$Value",
    "transforms.FilterConfluentExam.filter.condition" : "$[?(@.message > 500)]",
    "transforms.FilterConfluentExam.filter.type" : "include"
  }
}
```  

결과 토픽을 보면 파일에 작성된 전체 값 중 500 이상인 값들은 모두 필터링으로 제외되고, 
이상인 `600`, `700`, `800` 만 출력되는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-filter-confluent-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=600}
NO_HEADERS      null    Struct{message=700}
NO_HEADERS      null    Struct{message=800}
```  
