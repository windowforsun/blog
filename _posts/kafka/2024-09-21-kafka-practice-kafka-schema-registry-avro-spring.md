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
