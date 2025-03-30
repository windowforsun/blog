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

## Schema Registry with Debezium-ES and Kafka Streams
[Schema Registry](https://windowforsun.github.io/blog/tags/#schema-registry)
에서 알아본 것처럼 `Schema Registry` 는 `Kafka` 기반 메시징을 처리할 때 메시지 스키마 관리와 버전 제어 및 
`Avro` 메시지를 활용해 메시지 크기와 빠른 직렬화를 제공하는 중요한 역할을 한다. 
`Debezium CDC` 를 사용할 때 `Schema` 를 활성화해 메시지를 보다 명시적으로 처리할 수 있는데, 
이때 `Avro` 형식과 `Schema Registry` 를 함께 연동하는 방법에 대해 알아본다. 
그리고 더 나아가 `Kafka Streams` 애플리케이션에서 `Debezium` 에서 등록한 스키마를 활용해 메시지를 처리하는 방법에 대해 알아본다. 

전체 구성과 대략적인 흐름은 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-kafka-streams-debezium-es-1.drawio.png)


1. `Debezium MySQL Source Connector` 에서 `Avro` 와 스키마를 활성화 및 `Schema Registry` 연결 설정을 추가해 실행한다. 
2. `Debezium` 커넥터에서 `Kafka` 로 전송하는 메시지의 스키마를 `Schema Registry` 에 등록한다. 
3. `Kafka Streams` 애플리케이션에서 `Debezium` 에서 등록한 스키마를 활용해 메시지를 처리한다. 
4. `Elasticsearch Sink Connecotor` 에서도 동일하게 `Avro` 와 `Schema Registry` 연결 설정을 추가해 실행한다. 
5. `Elasticsearch` 커넥터는 `Avro` 메시지를 사용해 `Elasticsearch` 에 데이터를 적재한다.  

이후 진행되는 데모의 전체 코드는 [여기]()
에서 확인할 수 있다.  
