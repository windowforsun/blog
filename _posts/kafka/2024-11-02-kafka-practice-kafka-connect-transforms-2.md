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

### TimestampConverter
[TimestampConverter](https://docs.confluent.io/platform/current/connect/transforms/timestampconverter.html)
는 `Unix Epoch` 혹은 문자열, `Date`, `Timestamp` 과 같은 다양한 시간을 표현하는 타입간의 타임스탬프 변환을 할 수 있다. 

- Key : `org.apache.kafka.connect.transforms.TimestampConverter$Key`
- Value : `org.apache.kafka.connect.transforms.TimestampConverter$Value`

`timestamp-converter-input.txt` 파일 내용은 아래와 같다. 

```
1696521093000
1696621093000
1696721093000
1696821093000
1696921093000
1697021093000
```  

아래 `Json` 요청은 파일의 내용인 타입스탬프를 읽어 `yyyy-MM-dd` 형태로 변환하고, 
`InsertField` 를 사용해서 현재 타임스탬프 필드인 `timestamp_value` 를 추가한 뒤 
이를 `yyyy-MM-dd HH:mm:ss` 형태로 변환하는 예시이다.  


```json
{
  "name": "file-source-timestamp-converter",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/timestamp-converter-input.txt",
    "topic" : "file-source-timestamp-converter-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,CastExam,TimestampConverter1Exam,InsertFieldTimestampValueExam,TimestampConverter2Exam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.CastExam.type" : "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.CastExam.spec" : "message:int64",
    "transforms.TimestampConverter1Exam.type" : "org.apache.kafka.connect.transforms.TimestampConverter$Value",
    "transforms.TimestampConverter1Exam.format" : "yyyy-MM-dd",
    "transforms.TimestampConverter1Exam.field" : "message",
    "transforms.TimestampConverter1Exam.target.type" : "string",
    "transforms.InsertFieldTimestampValueExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldTimestampValueExam.timestamp.field" : "timestamp_value",
    "transforms.TimestampConverter2Exam.type" : "org.apache.kafka.connect.transforms.TimestampConverter$Value",
    "transforms.TimestampConverter2Exam.format" : "yyyy-MM-dd HH:mm:ss",
    "transforms.TimestampConverter2Exam.field" : "timestamp_value",
    "transforms.TimestampConverter2Exam.target.type" : "string"
  }
}
```  

결과 토픽을 보면 `message` 필드는 특정 타임스탬프 값을 `yyyy-MM-dd` 형태로 변환했고, 
`timestamp_value` 필드는 현재 시간을 `yyyy-MM-dd HH:mm:ss` 형태로 변환한 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-timestamp-converter-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    Struct{message=2023-10-05,timestamp_value=2024-06-15 17:05:54}
NO_HEADERS      null    Struct{message=2023-10-06,timestamp_value=2024-06-15 17:05:54}
NO_HEADERS      null    Struct{message=2023-10-07,timestamp_value=2024-06-15 17:05:54}
NO_HEADERS      null    Struct{message=2023-10-09,timestamp_value=2024-06-15 17:05:54}
NO_HEADERS      null    Struct{message=2023-10-10,timestamp_value=2024-06-15 17:05:54}
```  

### InsertHeader
[InsertHeader](https://docs.confluent.io/platform/current/connect/transforms/insertheader.html)
를 사용하면 메시지 헤더에 특정 필드와 값을 추가할 수 있다. 
`int`, `boolean`, `float` 등 기본형 타입도 가능하고, 
배열 및 맵 형식의 레코드도 추가할 수 있다. 


- `org.apache.kafka.connect.transforms.InsertHeader`

`insert-header-input.txt` 파일 내용은 아래와 같다.  

```
111
222
333
444
```  

아래 `Json` 요청은 메시지 헤더에 `header.int`, `header.boolean`, `header.string`, 
`header.array`, `header.map` 레코드를 추가하는 예시이다.  

```json
{
  "name": "file-source-insert-header",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/insert-header-input.txt",
    "topic" : "file-source-insert-header-topic",
    "value.converter.schemas.enable": "true",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "InsertHeaderIntExam,InsertHeaderBooleanExam,InsertHeaderStringExam,InsertHeaderArrayExam,InsertHeaderMapExam",
    "transforms.InsertHeaderIntExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderIntExam.header" : "header.int",
    "transforms.InsertHeaderIntExam.value.literal" : 1234,
    "transforms.InsertHeaderBooleanExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderBooleanExam.header" : "header.boolean",
    "transforms.InsertHeaderBooleanExam.value.literal" : true,
    "transforms.InsertHeaderStringExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderStringExam.header" : "header.string",
    "transforms.InsertHeaderStringExam.value.literal" : "stringValue",
    "transforms.InsertHeaderArrayExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderArrayExam.header" : "header.array",
    "transforms.InsertHeaderArrayExam.value.literal" : "[\"a\", \"b\",\"c\"]",
    "transforms.InsertHeaderMapExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderMapExam.header" : "header.map",
    "transforms.InsertHeaderMapExam.value.literal" : "{\"str\" :  \"string\", \"num\" :  123, \"list\" :  [-1,-2,-3,-4], \"map\" : {\"key\": \"value\"}}"
  }
}
```  

결과 토픽을 보면 헤더 출력 부분에 헤더에 추가한 필드와 값이 모두 정상적으로 메시지에 담긴것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-insert-header-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
header.int:1234,header.boolean:true,header.string:stringValue,header.array:["a","b","c"],header.map:{"str":"string","num":123,"list":[-1,-2,-3,-4],"map":{"key":"value"}}  null    {"schema":{"type":"string","optional":false},"payload":"111"}
header.int:1234,header.boolean:true,header.string:stringValue,header.array:["a","b","c"],header.map:{"str":"string","num":123,"list":[-1,-2,-3,-4],"map":{"key":"value"}}  null    {"schema":{"type":"string","optional":false},"payload":"222"}
header.int:1234,header.boolean:true,header.string:stringValue,header.array:["a","b","c"],header.map:{"str":"string","num":123,"list":[-1,-2,-3,-4],"map":{"key":"value"}}  null    {"schema":{"type":"string","optional":false},"payload":"333"}
header.int:1234,header.boolean:true,header.string:stringValue,header.array:["a","b","c"],header.map:{"str":"string","num":123,"list":[-1,-2,-3,-4],"map":{"key":"value"}}  null    {"schema":{"type":"string","optional":false},"payload":"444"}
```  

### DropHeaders
[DropHeaders](https://docs.confluent.io/platform/current/connect/transforms/dropheaders.html) 
를 사용하면 헤더에 있는 레코드 중 지정한 레코드를 제거할 수 있다. 

- `org.apache.kafka.connect.transforms.DropHeaders`  

아래 `Json` 요청은 `InsertHeader` 에서 추가한 필드 중 `header.int`, `header.array`, `header.map` 을 헤더에서 지우는 예제이다. 

```json
{
  "name": "file-source-drop-headers",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/drop-headers-input.txt",
    "topic" : "file-source-drop-headers-topic",
    "value.converter.schemas.enable": "true",
    "value.converter" : "org.apache.kafka.connect.json.JsonConverter",
    "transforms" : "InsertHeaderIntExam,InsertHeaderBooleanExam,InsertHeaderStringExam,InsertHeaderArrayExam,InsertHeaderMapExam,DropHeadersExam",
    "transforms.InsertHeaderIntExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderIntExam.header" : "header.int",
    "transforms.InsertHeaderIntExam.value.literal" : 1234,
    "transforms.InsertHeaderBooleanExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderBooleanExam.header" : "header.boolean",
    "transforms.InsertHeaderBooleanExam.value.literal" : true,
    "transforms.InsertHeaderStringExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderStringExam.header" : "header.string",
    "transforms.InsertHeaderStringExam.value.literal" : "stringValue",
    "transforms.InsertHeaderArrayExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderArrayExam.header" : "header.array",
    "transforms.InsertHeaderArrayExam.value.literal" : "[\"a\", \"b\",\"c\"]",
    "transforms.InsertHeaderMapExam.type" : "org.apache.kafka.connect.transforms.InsertHeader",
    "transforms.InsertHeaderMapExam.header" : "header.map",
    "transforms.InsertHeaderMapExam.value.literal" : "{\"str\" :  \"string\", \"num\" :  123, \"list\" :  [-1,-2,-3,-4], \"map\" : {\"key\": \"value\"}}",
    "transforms.DropHeadersExam.type" : "org.apache.kafka.connect.transforms.DropHeaders",
    "transforms.DropHeadersExam.headers" : "header.int,header.array,header.map"
  }
}
```  

결과 토픽을 보면 `InsertHeader` 를 통해 추가한 5개의 헤더 중 `header.boolean`, `header.string` 2개의 헤더만 남은 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-drop-headers-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
header.boolean:true,header.string:stringValue   null    {"schema":{"type":"string","optional":false},"payload":"111"}
header.boolean:true,header.string:stringValue   null    {"schema":{"type":"string","optional":false},"payload":"222"}
header.boolean:true,header.string:stringValue   null    {"schema":{"type":"string","optional":false},"payload":"333"}
header.boolean:true,header.string:stringValue   null    {"schema":{"type":"string","optional":false},"payload":"444"}
```  

### HeaderFrom
[HeaderFrom](https://docs.confluent.io/platform/current/connect/transforms/headerfrom.html)
을 상요하면 메시지의 키, 벨류의 특정 필드를 헤더로 이동하거나 복사 할 수 있다.  

- Key : `org.apache.kafka.connect.transforms.HeaderFrom$Key`
- Value : `org.apache.kafka.connect.transforms.HeaderFrom$Value`

`header-from-input.txt` 파일 내용은 아래와 같다.  

```
111
222
333
444
```  

아래 `Json` 요청은 메시지 값 중 파일에서 읽은 `message` 필드와 `InsertField` 로 추가한 `static_field` 를 헤더로 복사하고, 
메시지 키에 있는 `static_field` 는 헤더로 이동시키는 예제이다.  


```json
{
  "name": "file-source-header-from",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/header-from-input.txt",
    "topic" : "file-source-header-from-topic",
    "value.converter.schemas.enable": "true",
    "transforms" : "HoistFieldExam,InsertFieldStaticExam,HeaderFromValueExam,ValueToKeyExam,HeaderFromKeyExam",
    "transforms.HoistFieldExam.type" : "org.apache.kafka.connect.transforms.HoistField$Value",
    "transforms.HoistFieldExam.field" : "message",
    "transforms.InsertFieldStaticExam.type" : "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertFieldStaticExam.static.field" : "static_field",
    "transforms.InsertFieldStaticExam.static.value" : "static_value",
    "transforms.HeaderFromValueExam.type" : "org.apache.kafka.connect.transforms.HeaderFrom$Value",
    "transforms.HeaderFromValueExam.fields" : "message,static_field",
    "transforms.HeaderFromValueExam.headers" : "header_message,header_static_field",
    "transforms.HeaderFromValueExam.operation" : "copy",
    "transforms.ValueToKeyExam.type" : "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.ValueToKeyExam.fields" : "static_field",
    "transforms.HeaderFromKeyExam.type" : "org.apache.kafka.connect.transforms.HeaderFrom$Key",
    "transforms.HeaderFromKeyExam.fields" : "static_field",
    "transforms.HeaderFromKeyExam.headers" : "header_static_key_field",
    "transforms.HeaderFromKeyExam.operation" : "move"
  }
}
```  

결과 토픽을 확인하면 메시지 값중 파일에서 읽은 `message` 필드는 `header_message` 라는 필드로 헤더에 복사 됐고, 
`InsertField` 를 통해 추가한 `static_field` 는 `header_static_field` 라는 필드로 헤더에 복사 됐다. 
그리고 메시지 키에 있는 `static_field` 필드는 `header_static_key_field` 라는 필드로 헤더로 이동 된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-header-from-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
header_message:111,header_static_field:static_value,header_static_key_field:static_value        Struct{}        Struct{message=111,static_field=static_value}
header_message:222,header_static_field:static_value,header_static_key_field:static_value        Struct{}        Struct{message=222,static_field=static_value}
header_message:333,header_static_field:static_value,header_static_key_field:static_value        Struct{}        Struct{message=333,static_field=static_value}
header_message:444,header_static_field:static_value,header_static_key_field:static_value        Struct{}        Struct{message=444,static_field=static_value}
```  
