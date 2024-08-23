--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Spring Boot"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 의 특징과 장점 그리고 Spring Boot 기반 구현 예시에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Topology
    - Kafka Streams Topology
    - Consumer
    - Producer
    - Processor
    - Processor API
    - Streams DSL
toc: true
use_math: true
---  

## Kafka Streams
기존 스트림 링크 건뒤 Spring 기반 데모를 위해 한번 더 정리한다고 하면서 또 설명하기
[여기]() 
에서 `Kafka Streams` 에 대한 기본적인 개념에 대해서 알아보았다. 
`Kafka Streams Spring Boot Demo` 를 구성하기 앞서 한번 더 추가적인 내용을 집고 넘어가고자 한다.  

`Kafka Streams` 는 메시지 스트리밍 구현을 위한 다양한 `API` 와 메시지 처리, 변환, 집계와 같은 기능을 포함한다. 
그리고 필요에따라 유연하게 확장 가능한 구성과 메시지 처리의 신뢰성 그리고 유지 관리에 이점을 가져다 줄 수 있다. 
또한 실시간으로 무한히 들어오는 메시지를 낮은 지연시간을 바탕으로 빠른 처리를 가능하게 한다.

`Kafka Streams API` 는 `Consumer` 와 `Producer` 를 사용해 `Kafka Broker` 에 존재하는 `Topic` 의 메시지를
실시간으로 스트리밍하고, 처리/변환/집계 후, 다른 토픽으로 쓰는 역할을 수행한다.  
추가적으로 처리가 필요한 데이터가 `DB` 나 다른 외부 저장소에 있다면 `Kafka Connect API` 를 사용해서 외부 데이터를 `Kafka` 와 연결하는 파이프라인을 구성 할 수 있다.   

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-1.drawio.png)



### 특징

#### Processor Topology
`Kafka Streams` 는 `Processor Topology` 라는
하나의 스트림을 구성하는 프로세서들의 집합 개념을 통해 메시지를 처리한다.
`Processor` 에는 크게 `Source`, `Stream`, `Sink` 라는 3가지 타입의 종류가 있다.

- `Source Processor` : 토픽에서 메시지를 소비하는 역할을 하고, 소비한 데이터는 하나 이상의 `Stream` 혹은 `Sink` 프로세서에게 전달된다.
- `Stream Prcessor` : 실제 메시지를 처리하는 역할을 한다. 메시지 변환, 집계, 필터링을 수행하는데 이는 여러 단계로 구성된 `Stream Processor Chain` 을 메시지가 통과하며 수행된다. 최종 메시지는 `Sink` 프로세서에게 전달 된다.
- `Sink Processor` : `Processor Topology` 의 최종 결과를 다른 토픽에 쓰는 역할을 한다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-2.drawio.png)


`Stateful Processor` 라고 불리는 상태를 가진 프로세서는 다른 `Streams` 혹은 `Tables` 데이터와 결합하거나,
`Windowing` 을 바탕으로 데이터 그룹화를 통한 집계를 수행하는 프로세서를 의미한다.
이러한 프로세서는 추가적인 저장소를 사용해서 상태를 유지하는 특성을 가지고 있다.

그리고 하나의 `Processor Topology` 를 기준으로 각 메시지는 순차적으로 처리되는 특성을 갖는다.
이는 앞선 메시지가 처리 완료되기 전까지 다음 메시지 처리는 수행되지 않음을 의미한다.
`Topology` 는 `Sub-Topology` 를 포함 할 수 있는데,
`Sub-Topology` 와 다른 `Sub-Topology` 는 병렬로 메시지를 처리한다.
