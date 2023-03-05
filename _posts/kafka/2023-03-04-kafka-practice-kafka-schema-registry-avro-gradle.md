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
[Kafka 와 Confluent Schema Registry]({{site.baseurl}}{% link _posts/kafka/2023-02-25-kafka-practice-kafka-schema-registry.md %})
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


그래서 이번 포스트에서는 `Gradle` 기반으로 `Avro` 스키마만 정의해 주면 `Schema Registry` 에 
등록도 해주고 매핑되는 클래스 또한 생성해주는 플러그인과 사용 방법에 대해 알아 본다.  


[specific.avro.reader](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-schema-registry/#producer-consumer-%EC%97%B0%EB%8F%99)
이전 포스팅에서 해당 옵션에 `true` 값을 주면 역직렬화 관련 에러가 발생했다. 
이번 포스팅에서 알아보는 `Gradle` 기반 방식을 사용하면 `Producer`, `Consumer` 에서 
특정 `Avro` 스키마 버전을 명시해서 보다 안정적으로 애플리케이션 구현이 가능하다.  

이전 포스팅에서는 `Producer` 가 생산한 스키마와 상관없이 `Consumer` 는 `GenericRecord` 를 사용해서 

### Gradle Avro Serialize/Deserialize Plugin
`Gradle` 기반으로 `Avro` 스키마에 해당하는 클래스를 자동 생성하기 위해서는 [davidmc24/gradle-avro-plugin](https://github.com/davidmc24/gradle-avro-plugin)
이라는 `Gradle Plugin` 을 사용한다. 

플러그인 사용을 위해서 `setting.gradle` 파일에 아래 저장소 설정을 추가해 준다.  

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```  

다음으로 `build.gradle` 내용은 아래와 같다.  

```groovy

```




---  
## Reference
[Kafka Clients in Java with Avro Serialization and Confluent Schema Registry](https://thecodinginterface.com/blog/gradle-java-avro-kafka-clients/)  
[davidmc24/gradle-avro-plugin](https://github.com/davidmc24/gradle-avro-plugin)  
