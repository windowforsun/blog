--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 2"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT 중 '
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Retry
    - Non Blocking
    - Consumer
    - ExtractField
    - ReplaceField
    - TimestampConverter
    - InsertHeader
    - DropHeader
    - HeaderFrom
    - Flatten
toc: true
use_math: true
---

## Kafka Connect Transforms
[Kafka Connect Transforms(SMT) 1]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-1.md %})
에 이어서 추가적인 `Transforms` 의 사용법에 대해 알아본다. 
테스트를 위해 구성하는 환경와 방식은 위 첫번째 포스팅과 동일하다. 
테스트 및 예제 진행에 궁금한 점이 있다면 이전 포스팅에서 관련 내용을 확인 할 수 있다.


### ExtractField
[ExtractField](https://docs.confluent.io/platform/current/connect/transforms/extractfield.html)
는 `Map` ,`Struct` 로 구성된 키 혹은 값에서 지정된 특정 필드를 밖으로 끄집어 낸다. 
그리고 해당 값으로 키 혹은 값을 대체한다. 

- Key : `org.apache.kafka.connect.transforms.ExtractField$Key`
- Value : `org.apache.kafka.connect.transforms.ExtractField$Value`

`extract-field-input.txt` 파일의 내용은 아래와 같다.  

```
111
222
333
444
```

아래 `Json` 요청은 파일로 읽어들인 메시지를 `float` 으로 형 변환 후 `ExtractField` 를 수행해서 
`payload` 의 타입을 `string` 에서 소수 타입으로 변경하는 예제이다.  

```json
{
  "name": "file-source-extract-field",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/extract-field-input.txt",
    "topic" : "file-source-extract-field-topic",
    "value.converter.schemas.enable": "true",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam,CastExam,ExtractFieldExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.CastExam.type" : "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.CastExam.spec" : "message:float64",
    "transforms.ExtractFieldExam.type" : "org.apache.kafka.connect.transforms.ExtractField$Value",
    "transforms.ExtractFieldExam.field" : "message"
  }
}
```  

결과 토픽을 살펴보면 최에 파일을 읽게되면 문자열 타입으로 `"111"` 와 같이 읽어들이 지만 `HoistField`, `Cast`, `ExtractField` 을 함께 사용해서 
`payload` 의 메세지 자체를 `double` 타입을 형변환한 결과를 확인 할 수 있다.   

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-extract-field-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"double","optional":false},"payload":111.0}
NO_HEADERS      null    {"schema":{"type":"double","optional":false},"payload":222.0}
NO_HEADERS      null    {"schema":{"type":"double","optional":false},"payload":333.0}
NO_HEADERS      null    {"schema":{"type":"double","optional":false},"payload":444.0}
```  

### ReplaceField
[ReplaceField](https://docs.confluent.io/platform/current/connect/transforms/replacefield.html)
는 `Struct`, `Map` 내의 필드를 필터링하거나 이름을 변경할 수 있다.  

- Key : `org.apache.kafka.connect.transforms.ReplaceField$Key`
- Value : `org.apache.kafka.connect.transforms.ReplaceField$Value`

`replace-field-input.txt` 파일의 내용은 아래와 같다.  

```
111
222
333
444
```

#### ReplaceField Exclude
아래 `Json` 요청은 `exclude` 를 사용해 특정 메시지 필드를 제거하는 요청의 예시이다. 

```json
{
  "name": "file-source-replace-field-exclude",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/replace-field-exclude-input.txt",
    "topic" : "file-source-replace-field-exclude-topic",
    "value.converter.schemas.enable": "true",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "HoistFieldExam,ReplaceFieldDropExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ReplaceFieldDropExam.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.ReplaceFieldDropExam.exclude": "message"
  }
}
```  

결과 토픽을 확인하면 메시지에서 유일한 필드인 `message` 필드를 `exclude` 로 제거 했기 때문에 스키마와 `payload` 에는 모두 빈값이 들어가 있는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-replace-field-exclude-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"struct","fields":[],"optional":false},"payload":{}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[],"optional":false},"payload":{}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[],"optional":false},"payload":{}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[],"optional":false},"payload":{}}
```  

#### ReplaceField Include
아래 `Json` 요청은 `include` 를 사용해서 특정된 해당 필드만 메시지에 포함하는 요청의 예시이다.  

```json
{
  "name": "file-source-replace-field-include",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/replace-field-include-input.txt",
    "topic" : "file-source-replace-field-include-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,InsertFieldStaticExam,ReplaceFieldIncludeExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.InsertFieldStaticExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldStaticExam.static.field" : "static_field",
    "transforms.InsertFieldStaticExam.static.value" : "static_value",
    "transforms.ReplaceFieldIncludeExam.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.ReplaceFieldIncludeExam.include": "static_field"
  }
}
```  

결과 토픽을 보면 기존에는 `message`, `static_field` 두 개의 필드가 `payload` 에 존재하지만, 
`include` 로 지정된 `static_field` 만 `payload` 에 남은 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-replace-field-include-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":true,"field":"static_field"}],"optional":false},"payload":{"static_field":"static_value"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":true,"field":"static_field"}],"optional":false},"payload":{"static_field":"static_value"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":true,"field":"static_field"}],"optional":false},"payload":{"static_field":"static_value"}}
```  

#### ReplaceField Renames
아래 `Json` 요청은 메시지의 필드명을 변경하는 예시이다.  

```json
{
  "name": "file-source-replace-field-renames",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/replace-field-renames-input.txt",
    "topic" : "file-source-replace-field-renames-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,ReplaceFieldRenamesExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.ReplaceFieldRenamesExam.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.ReplaceFieldRenamesExam.renames": "message:data"
  }
}
```  

결과 토픽을 보면 `message` 필드명이 지정된 값이 `data` 로 변경된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-replace-field-renames-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"data"}],"optional":false},"payload":{"data":"111"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"data"}],"optional":false},"payload":{"data":"222"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"data"}],"optional":false},"payload":{"data":"333"}}
NO_HEADERS      null    {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"data"}],"optional":false},"payload":{"data":"444"}}
```  
