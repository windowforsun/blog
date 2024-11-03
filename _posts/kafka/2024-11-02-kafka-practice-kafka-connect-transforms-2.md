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
