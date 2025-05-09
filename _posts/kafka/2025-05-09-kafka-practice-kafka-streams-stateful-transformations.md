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
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Stateful Transformations
`Kafka Streams` 에서 `Stateful Transformations` 은 입력을 처리하고 출력을 생성할 때 내부 상태(`State`)에 의존하는 변환을 의미한다. 
스트림 프로세서에서 연결된 `State Store` 가 필요하고, 이는 데이터를 임시 저장하고 처리에 필요한 상태를 유지한다. 
예를 들어, 집계 작업에서는 `windowing state store` 를 사용하여 각 윈도우별 최신 집계 결과를 수집할 수 있다. 
조인 작업의 경우 정의된 윈도우 경계 내에서 수신된 모든 레코드를 수집하기 위해 `windowing state store` 가 사용된다.  

그리고 스트림 프로세서에서 처리를 위한 상태를 저장하는 `State Store` 는 내결합성(`fault-tolerance`)를 갖추고 있다. 
이는 장애가 발생할 경우, `Kafka Streams` 는 처리 재개 전에 모든 상태 저장소를 완전히 복원하는 것을 보장한다.  

`Streams DSL` 을 사용해 수행할 수 있는 `Stateful Transformation` 은 아래와 같은 종류가 있다.  

- `Aggregation` : 데이터를 집계하고, 상태를 유지하면서 실시간으로 최신 결과를 계산
- `Joining` : 두 개이상의 스트림을 `Windowed` or `Non-windowed` 기반으로 병합하여 상태를 유지
- `Windowing` : 시간 or 세션 기반의 읜도우를 적용하여 상태별 데이터를 구분하고 집계
- `Custom Processors and Transformations` : 사용자가 정의한 프로세서를 기반으로 상태를 저장하고 처리

아래 그림은 `Streams DSL` 을 통해 사용할 수 있는 `Stateful Transformations` 을 도식화한 내용이다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-stateful-transformations.drawio.png)

이후 소개하는 예제의 전체 코드 내용과 결과에 대한 테스트 코드는 [여기]()
에서 확인 할 수 있다.  
