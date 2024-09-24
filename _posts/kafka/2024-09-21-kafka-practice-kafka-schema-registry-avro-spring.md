--- 
layout: single
classes: wide
title: "[Kafka] "
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
toc: true
use_math: true
---  

## Avro & Schema Registry
[Schema Registry]({{site.baseurl}}{% link _posts/kafka/2023-02-25-kafka-practice-kafka-schema-registry.md %})
와
[Schema Registry Avro Gradle]({{site.baseurl}}{% link _posts/kafka/2023-03-04-kafka-practice-kafka-schema-registry-avro-gradle.md %})
에서 `Schema Registry` 와 `Avro` 를 `Gradle` 환경에서 사용할 수 있는 방법에 대해 알아보았다. 
이번 포스팅에서는 이를 종합해 `Spring Boot` 에서 활용하는 예시를 알아보고자 하는데, 
그에 앞서 먼저 `Schema Registry` 와 `Avro` 에 대해 다시 한번 정리해보려 한다.  

`Kafka Broker` 를 기반으로 `Producer` 와 `Consumer` 가 스트리밍을 처리할 때, 
메시지의 직렬화/역직렬화는 필수적인 요소이고, 애플리케이션은 중단 없이 계속해서 메시지 처리가 가능해야 한다. 
그리고 그 메시지에는 다양한 데이터가 포함 될 수 있고, 필요에 따라 추가/삭제 또한 가능할 것이다. 
이러한 요구사항을 만족 시킬 수 있는것이 바로 스키마이다. 
스키마를 적용하면 `Producer` 가 직렬화를 통해 토픽에 쓴 메시지가 토픽의 `Consumer` 가 소비하고 역직렬화를 해서 처리 할 수 있도록 보장한다. 
이렇게 `Producer` 와 `Consumer` 는 스키마를 알아야 하고 스키마 변경 관리를 위해서는 `Schema Registry` 가 필요하다. 
여기에 `Apache Avro` 를 사용하면 버전 간 호환성, 타입 처리를 제공하는 직렬화 프레임워크로 끊김 없는 스트림 처리에서 
여러 버전 메시지 스키마르 호환성 높게 사용 할 수 있도록 한다.  


### Avro
`Apache Avro` 는 데이터 직렬화 시스템으로 분산 컴퓨팅 환경에서 효과적이다. 
`Avro` 는 `JSON` 형식의 스키마를 사용하여 스키마를 정의하고, 
스키마를 바탕으로 데이터를 직렬화/역직렬화한다. 
먼저 정의된 스키마 데이터는 이를 저장하고 관리하는 저장소에 보관된다. 
즉 실제 데이터와 스키마가 분리되어 있어 다른 시스템들과 비교했을 때 좀 더 신뢰성과 효율성에 이점이 있다.  

그리고 `Avro` 는 데이터를 바이너르 형태로 직렬화해 저장하기 때문에 
`JSON`, `XML` 과 같은 텍스트 기반 데이터 포멧과 비교해서 크기가 작다. 
이를 토해 네트워크 전송 시간 및 저장공간을 절약할 수 있다.  

다른 특징으로는 스키마의 진화를 지원한다. 
여기서 스키마 진화는 시간이 지남에 따라 스키마에 변경이 발생하는 것을 의미한다. 
구버전 데이터와 신버전 데이터가 호환될 수 있도록 하여 스트리밍 처리환경에서 활용도가 높다.   


### Schema Registry
앞서 메시지를 정의하는 스키마를 사용하면 메시지 변경에 따른 호환성을 높일 수 있다는 것에 대해 알아보았다. 
스키마를 사용하는 방법 중 가장 간단한 방법은 메시지에 해당 메시지와 대응되는 스키마 내용을 포함시키는 것이다. 
하지만 이는 모든 메시지의 크기를 증가시켜, `Producer`, `Consumer`, `Broker` 가 필요로하는 `CPU` 와 메모리를 증가시켜 처리량에 직접적인 영향을 줄 수 있다.  

이러한 문제점을 해결하면서 메시지 스키마를 사용할 수 있는 방법이 바로 `Schema Registry` 이다. 
`Schema Registry` 는 앞서 나열한 구성들과 독립적으로 구성되어 네트워크 통신을 통해 메시지의 스키마를 관리하고 정보를 조회할 수 있다.  


.. 그림 ..

`Kafka` 진영에서 가장 보편적인 `Schema Registry` 는 `Confluent Schema Registry` 이다. 
스키마를 등록하고 조회하기 위한 `REST API` 를 제공한다. 
그러므로 `Producer` 실행 전 명시적으로 사용하는 스키마를 `REST API` 를 사용해서 등록해주거나, 
`CI/CD` 파이프라인의 일부에서 스키마를 등록하는 시나리오를 바탕으로 데이터 스트리밍 환경에서 데이터 일관성과 통합성을 보장 할 수 있다.   

`Schema Registry` 는 필수적인 구성요소는 아니고 구성하지 않더라도 메시지 스트리밍 처리는 구현할 수 있다. 
그렇다면 있을 경우와 그렇지 않은 경우에 어떠한 차이를 보이는지에 대해 좀 더 자세히 알아보자. 

- `Schema Registry` 가 있는 경우
  - 스키마 진화 관리 : `Schema Registry` 는 스키마의 버전 관리를 지원한다. 스키마가 변할 경우 이전 버전과의 호환성을 유지할 수 있다. 이는 스키마가 변경 됐을 때 기존 스키마와 어떻게 호환될지를 정의하고 관리할 수 있다. 
  - 중앙 집중식 스키마 관리 : 모든 스키마 정보를 한 곳에 저장함으로써 모든 애플리케이션에서 동일한 스키마를 사용하고 있음을 보장할 수 있다. 여러 서비스 및 여러 팀 간의 일관성 유지에 큰 도움이 된다. 
  - 자동 스키마 검증 : `Producer` 가 메시지를 전송할 때, `Schema Registry` 는 전송된 메시지가 등록된 스키마와 일치하는지 검증한다. 
  - 효율적인 데이터 역직렬화 : `Consumer` 는 `Schema Registry` 에서 검색하여 받은 스키마로 메시지를 역직렬화 할 수 있다. 이는 스키마가 매번 메시지에 포함되지 않기 때문에 메시지 크기가 작아지고 네트워크 효율성이 높아진다. 
- `Schema Registry` 가 없는 경우
  - 스키마 포함 메시지 : 스키마가 메시지에 포함되어 메시지 크기가 커져 네트워크와 저장공간을 더 많이 사용한다. 
  - 스키마 관리의 복잡성 증가 : 각 애플리케이션 또는 팀에서 독립적으로 스키마를 관리하게 되면, 시스템 전체에서 스키마의 일관성을 유지하기 어렵다. 
  - 스키마 진화의 어려움 : 스키마 변경 시 모든 관련 애플리케이션을 업데이트하고, 변경된 스키마에 대한 호환성을 각각 검증해야 한다. 

### Registering Schemas
`Confluent Schema Registry` 는 내부적으로 `Kafka` 를 저장소로 사용한다. 
이는 `Schema Registry` 에 등록되는 스키마를 `_schemas` 라는 압축된 토픽에 저장하는 것을 의미한다. 
만약 토픽이름을 변경하고 싶다면 `kafkastore.topic` 옵션 수정을 통해 가능하다.  

`Confluent Schema Registry` 는 앞서 언급한 것과 같이 `REST API` 를 사용해서 스키마를 등록한다. 
이는 수동으로 `REST API` 를 호출해서 등록하는 것도 가능하고, `Schema Registry Client` 를 사용하거나, 
`Maven`, `Gradle` 플러그인을 통해 수행하도록 구현할 수도 있다. 
또한 `Kafka Avro` 라이브러리를 사용한다면 `Producer` 가 메시지를 보내기 전에 `Avro Serilizer` 가 메시지의 스키마를 `Schema Registry` 에 확인한다. 
스키마가 존재하지 않을 경우 스키마를 등록하고, 존재한다면 스키마 아이디를 사용해서 메시지를 직렬화한다.  

최종적으로 `Schema Registry` 에 등록된 스키마는 `_schemas` 토픽에 쓰여지고, 
`Shcema Registry` 는 내부 `Consumer` 를 통해 이를 소비해서 로컬 캐시에 저장한다.  


.. 그림 ..


1. `Avro Schema` 는 `REST POST` 를 통해 `Schema Registry` 에 등록된다. 
2. `Schema Registry` 는 새로운 스키마를 내부 `Producer` 를 통해 `_schemas` 토픽에 저장한다. 
3. `Schema Registry` 는 내부 `Consumer` 로 `_schemas` 에 등록된 새로운 스키마를 소비한다. 
4. 소비된 새로운 스키마는 로컬 캐시에 저장된다. 

이후 스키마를 필요로하는 `Producer`, `Consumer` 가 스키마 아이디 혹은 스키마를 조회할 때 로컬 캐시를 통해 응답하게 된다. 
`Confluent Schema Registry` 는 데이터 저장소로 `Kafka` 를 사용하는 만큼 `Kafka` 에서 제공하는 모든 장점을 얻을 수 있다. 
단편적인 예시로 특정 `Broker` 가 실패하더라도, 복제 파티션을 통해 `Scahme Registry` 의 사용은 지속적으로 가능하다. 
또한 `Schema Registry` 가 실패해 재시작 되는 경우에는 `_schemas` 토픽 구독을 통해 등록된 기존 스키마를 로컬 캐시에 재구축하게 된다.  
