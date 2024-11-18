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

#### Filter(Confluent) Key
아래 `JSON` 요청은 키에서 `message` 필드의 값이 `500` 이하인 레코드만 포함하는 예제이다.  

```json
{
  "name": "file-source-filter-confluent-key",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/filter-confluent-input.txt",
    "topic" : "file-source-filter-confluent-topic-key",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,CastExam,ValueToKeyExam,FilterConfluentExam,ExtractFieldExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.CastExam.type" : "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.CastExam.spec" : "message:int32",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "message",
    "transforms.FilterConfluentExam.type" : "io.confluent.connect.transforms.Filter$Key",
    "transforms.FilterConfluentExam.filter.condition" : "$[?(@.message > 500)]",
    "transforms.FilterConfluentExam.filter.type" : "exclude",
    "transforms.ExtractFieldExam.type" : "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.ExtractFieldExam.field" : "message"
  }
}
```
결과 토픽을 보면 전체 값 중 500 초과하는 값들은 모두 필터링으로 제외되고, 
이하인 값들만 출력되는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-filter-confluent-topic-key \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      100     Struct{message=100}
NO_HEADERS      500     Struct{message=500}
NO_HEADERS      200     Struct{message=200}
NO_HEADERS      300     Struct{message=300}
NO_HEADERS      400     Struct{message=400}
NO_HEADERS      0       Struct{message=0}
```  


### Filter(Kafka)
[Filter(Kafka)](https://docs.confluent.io/platform/current/connect/transforms/filter-ak.html)
를 사용하면 레코드를 주어진 기준 값인 토픽이름 혹은 헤더의 키를 바탕으로 필터링 조건을 정의 할 수 있다.  
그래서 해당 조건을 사용해서 `Transform` 체인을 구성 할때 특정 `Transform` 에 조건을 적용해 조건부 처리를 수행해 `downstream` 으로 보내게 된다. 

- `org.apache.kafka.connect.transforms.Filter`

`Filter(Kafka)` 의 조건은 `predicates` 라는 `JSON` 필드를 사용해서 정의 할 수 있다. 
아래는 조건 정의에 필요한 주요 필드 명이다. 

- `predicates` : 하나 이상의 `Transform` 에 적용 될 수 있는 `predicate` 의 별칭을(`alias`) 정의한다. 
- `predicates.<alias>.type` : 해당 `predicate` 에서 사용 할 클래스 이름을 작성한다. 
- `predicates.<alias>.<predicateClassConfig>` : `type` 에서 작성한 클래스에서 필요로 하는 프로퍼티에 대한 내용을 작성한다. 

`predicates.<alias>.type` 에서 사용할 수 있는 클래스의 종류와 설정에 필요한 프로퍼티는 [여기](https://kafka.apache.org/26/generated/connect_predicates.html)
에서 확인 할 수 있다.

- `org.apache.kafka.connect.transforms.predicates.TopicNameMatches` : 토픽이름이 주어진 정규식과 일치하는지 확인 한다.
- `org.apache.kafka.connect.transforms.predicates.HasHeaderKey` : 메시지 헤더에 설정한 이름으로 된 키가 있는지 확인 한다.
- `org.apache.kafka.connect.transforms.predicates.RecordIsTombstone` : 레코드가 `Tombstone`(null) 인지 확인 한다.


앞서 알아본 `Transform` 등 대부분 `Transform` 에는 `predicate` 라는 필드를 사용해서 위에서 정의한 `predicates` 에 정의한 `alias` 를 적용 할 수 있다. 
그렇게 되면 해당 `Transform` 이 수행되는 조건은 `predicate` 의 만족 여부에 따르게 된다. 
또한 `netage` 라는 필드를 `true`(만족하는 경우 적용), `false`(만족하지 않는 경우 적용)로 설정해 부정과 긍정의 설정도 가능하다.  


#### Filter(Kafka) 1
아래 `JSON` 요청은 `file-source-.*` 의 정규식을 만족하는 토픽만 `downstream` 으로 메시지를 전송하는 예시이다. 
`TopicNameMaches` 로 토픽에 대한 조건을 구성하기 위해서 `ExtractTopic` 을 사용해서 메시지의 값 자체가 토픽 이름이 되도록 했다.  

`filter-kafka-input.txt` 파일의 내용에는 아래와 같이 토픽 이름으로 사용 될 문자열이 작성돼 있다.  

```
file-source-topic
other-source-topic
file-source-topic
other-source-topic
```

```json
{
  "name": "file-source-filter-kafka",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/filter-kafka-input.txt",
    "topic" : "*-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ExtractTopicValueExam,TopicNameFilterExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ExtractTopicValueExam.type" : "io.confluent.connect.transforms.ExtractTopic$Value",
    "transforms.ExtractTopicValueExam.field" : "message",
    "transforms.TopicNameFilterExam.type" : "org.apache.kafka.connect.transforms.Filter",
    "transforms.TopicNameFilterExam.predicate" : "IsFileSourceTopic",
    "transforms.TopicNameFilterExam.negate" : "true",
    "predicates" : "IsFileSourceTopic",
    "predicates.IsFileSourceTopic.type" : "org.apache.kafka.connect.transforms.predicates.TopicNameMatches",
    "predicates.IsFileSourceTopic.pattern" : "file-source-.*"
  }
}
```  

`file-source-topic`, `other-source-topic` 두 결과 토픽을 확인하면 `predicates` 에 정의한 `IsFileSourceTopic` 의 `type` 이 `TopicNameMatches` 이므로 토픽 이름에 대한 확인을 진행한다. 
그리고 토픽 이름확인에 사용할 정규식이 `file-source-.*` 이고, `negate` 가 `true` 이므로 `file-source` 로 시작하는 토픽의 경우에만 메시지가 전달되는 것을 확인 할 수 있다.  


```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=file-source-topic}
NO_HEADERS      null    Struct{message=file-source-topic}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic other-source-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
EMPTY
```

#### Filter(Kafka) 2
아래 `JSON` 요청은 `Filter(Kafka)` 를 사용해서 좀 더 복잡한 구성을 해본 예시이다. 
토픽이름이 `file-source-` 로 시작하는 경우에만 `Timestamp` 필드가 추가된다. 
그리고 토픽이름이 `other` 로 시작하지 않는 경우에만 `downstream` 으로 메시지를 전달한다.  


```json
{
  "name": "file-source-filter-kafka-topic",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/filter-kafka-topic-input.txt",
    "topic" : "*-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ExtractTopicValueExam,InsertFieldTimestampExam,TopicNameFilterExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ExtractTopicValueExam.type" : "io.confluent.connect.transforms.ExtractTopic$Value",
    "transforms.ExtractTopicValueExam.field" : "message",
    "transforms.InsertFieldTimestampExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldTimestampExam.timestamp.field" : "timestamp_field",
    "transforms.InsertFieldTimestampExam.predicate" : "IsFileSourceTopic",
    "transforms.TopicNameFilterExam.type" : "org.apache.kafka.connect.transforms.Filter",
    "transforms.TopicNameFilterExam.predicate" : "IsSourceTopic",
    "transforms.TopicNameFilterExam.negate" : "false",
    "predicates" : "IsFileSourceTopic,IsSourceTopic",
    "predicates.IsFileSourceTopic.type" : "org.apache.kafka.connect.transforms.predicates.TopicNameMatches",
    "predicates.IsFileSourceTopic.pattern" : "file-source-.*",
    "predicates.IsSourceTopic.type" : "org.apache.kafka.connect.transforms.predicates.TopicNameMatches",
    "predicates.IsSourceTopic.pattern" : "other.*"
  }
}
```

`file-source-topic-2`, `db-source-topic-2`, `other-topic-2` 결과 토픽을 확인하면, 구성한 내용과 동일하게 메시지가 출력되는 것을 확인 할 수 있다. 
이전 예제와 동일하게 `message` 의 값 자체를 `ExtractTopic` 을 사용해 토픽으로 설정한다.
그리고 `Timestamp` 필드를 추가하는 `InsertFieldTimestampExam` 라는 `Transform` 에 `IsFileSourceTopic` 라는 `predicate` 를 설정해서,
토픽 이름이 `file-source-.*` 로 시작하는 경우에만 `Timestamp` 가 추가되도록 했다.
그리고 `TopicNameFilterExam` 라는 `Transform` 에 `IsSourceTopic` 라는 `predicate` 를 설정할 때 `negate` 를 `false` 로 설정해서, 
토픽 이름이 `other` 로 시작하지 않는 경우에만 메시지를 처리하도록 했다.  

```bash
docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-topic-2 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=file-source-topic-2,timestamp_field=Tue Jun 18 17:51:59 GMT 2024}
NO_HEADERS      null    Struct{message=file-source-topic-2,timestamp_field=Tue Jun 18 17:51:59 GMT 2024}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic db-source-topic-2 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=db-source-topic-2}
NO_HEADERS      null    Struct{message=db-source-topic-2}

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic other-topic-2 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
EMPTY
```

### TimestampRouter
[TimestampRouter](https://docs.confluent.io/platform/current/connect/transforms/timestamprouter.html)
을 사용하면 메시지를 시간값이 포함된 토픽으로 전달할 수 있다. 
주로 `SinkConnector` 에서 다른 저장소에 일자별 메시지를 저장해야 하는 경우 사용 할 수 있다. 

- `org.apache.kafka.connect.transforms.TimestampRouter`

`timestamp-router-input.txt` 파일 내용은 아래와 같다.  

```
111
222
333
444
```  

아래 `JSON` 요청은 파일 소스로 부터 데이터를 읽은 뒤 이를 `yyy-MM-dd` 접미사가 붙은 토픽으로 메시지를 전달하는 예시이다.  

```json
{
  "name": "file-source-timestamp-router",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/timestamp-router-input.txt",
    "topic" : "file-source-timestamp-router-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "TimestampRouterExam",
    "transforms.TimestampRouterExam.type" : "org.apache.kafka.connect.transforms.TimestampRouter",
    "transforms.TimestampRouterExam.topic.format" : "new-${topic}-${timestamp}",
    "transforms.TimestampRouterExam.timestamp.format" : "yyyy-MM-dd"
  }
}
```  

`Kafka` 토픽을 보면 설정한 것과 같이 `new-${topic}-${timestamp}` 포맷으로 토픽이 존재하는 것을 확인 할 수 있다. 
그리고 해당 토픽에 파일 소스에 있는 데이터들이 담겨 있다.  


```bash
$ docker exec -it myKafka kafka-topics.sh --bootstrap-server localhost:9092 --list
new-file-source-timestamp-router-topic-2024-06-19

$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic new-file-source-timestamp-router-topic-2024-06-19 \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    111
NO_HEADERS      null    222
NO_HEADERS      null    333
NO_HEADERS      null    444
```  





---  
## Reference
[Kafka Connect Single Message Transform](https://docs.confluent.io/platform/current/connect/transforms/overview.html)  
[How to Use Single Message Transforms in Kafka Connect](https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/?session_ref=https://www.google.com/&_gl=1*1vjg9z4*_ga*MjA0NzkyNTk2MC4xNzAxMzgyMDQ3*_ga_D2D3EGKSGD*MTcxNzkzMjc5My44Mi4xLjE3MTc5MzM0NjAuNTQuMC4w&_ga=2.7783026.1766080457.1717921898-2047925960.1701382047)  

