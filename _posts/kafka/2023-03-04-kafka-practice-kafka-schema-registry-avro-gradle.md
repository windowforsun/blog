--- 
layout: single
classes: wide
title: "[Kafka] Kafka Schema Registry, Java Gradle with Avro Serialize/Deserialize"
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
    - Schema Registry
    - Confluent Schema Registry
    - Avro
    - Gradle
    - Java
toc: true
use_math: true
---  


## Avro Serialize/Deserialize
[Kafka 와 Confluent Schema Registry]()
에서는 `Confluent Schema Registry` 에 대한 개념과 기본적인 사용 방법에 대해서 알아보 았다.  

전 포스트에서 `Avro` 포맷으로 `Serialize/Deserialize` 하는 방법은 아래와 같은 `avro` 포맷 형식을 
코드내 문자열로 만들어 직접 스키마를 구성하는 방법이였다.  

```json
{
  "namespace":"com.windowforsun.test",
  "type":"record",
  "name":"my-test",
  "fields":[
    {"name":"name","type":"string"},
    {"name":"age","type":"int"}
  ]
}
```  

하지만 이러한 방법으로 실제 애플리케이션을 구현하기에는 위 스키마를 클래스 구현체와 매핑해야 한다는 점이다. 
직접 `SpecificRecordBase`, `SpecificRecord` 를 상속 받고 구현해서 매핑지을 수 있지만, 
매번 사용이 필요한 스키마에 대해서 해당 작업을 해주기는 번거러운 작업이다.  


SPECIFIC_AVRO_READER_CONFIG,
specific.avro.reader 
문제 및 언급



그래서 이번 포스트에서는 `Gradle` 기반으로 `Avro` 스키마만 정의해 주면 `Schema Registry` 에 
등록도 해주고 매핑되는 클래스 또한 생성해주는 플러그인과 사용 방법에 대해 알아 본다.  


### Gradle Avro Serialize/Deserialize Plugin





---  
## Reference
[Kafka Clients in Java with Avro Serialization and Confluent Schema Registry](https://thecodinginterface.com/blog/gradle-java-avro-kafka-clients/)  
[davidmc24/gradle-avro-plugin](https://github.com/davidmc24/gradle-avro-plugin)  
