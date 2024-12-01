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

## Kafka Streams Join

### Kafka Streams Core Concept
`Kafka` 는 `Message Broker` 로 `Producer` 가 `key-value` 로 구성된 메시지를 `Topic` 으로 전송하고, 
이를 `Consumer` 가 `polling` 을 통해 소비하는 구조이다. 
그리고 각 `Topic` 은 1개 이상의 `Partition` 으로 구성될 수 있고, 
이는 `Kafka Borker` 에 의해 분산돼 관리된다.  
우리는 이러한 기본적인 `Kafka` 의 메시징을 통해 `Kafka Streams API` 의 추상화인 `KStream` 과 `KTable` 2가지 방식으로 메시지를 
처리 할 수 있다.  

`KStream` 은 `Stateless` 성격을 지닌 추상화로 `key-value` 메시지를 `Producer-Consumer` 와 유사한 방식으로 메시지를 처리한다. 
`KStream` 에 전달되는 메시지는 `Topic` 의 전체 메시지일 수도 있고, `filter`, `map`, `flatMap` 등과 같은 
전처리된 메시지일 수 있다.  

`KTable` 은 `Stateful` 성격을 지닌 추상화로 `KStream` 에 전달되는 메시지를 집계하여 구성된다. 
`KTable` 은 `changelog stream` 이라고 달리 부를 수 있는데, 
이는 메시지 키에 대한 최신 값만을 유지하며 새로 들어오는 메시지마다 최신화가 반영된다.  

웹 페이지에서 사용자 계정당 방문을 `Kafka Topic` 으로 보낸다고 해보자. 
`key=ID-value=TIMESTAMP` 와 같은 메시지 구조일 때, 
`KStream` 은 동일한 `ID` 의 사용자가 여러번 방문하면 방문한 수만큼의 메시지로 전달 될 것이다. 
그러므로 `KStream` 에서 카운트 값은 중복방문을 포함한 전체 방문 횟수가 된다. 
반면 `KTable` 은 최신 메시지만 유지하기 때문에, 
`KTable` 에서 카운트 값은 웹 사이틀르 방문한 유니크한 `ID` 의 수가 된다.  

`KStream` 과 `KTable` 은 `Aggregation` 과 `Join` 연산을 위해 `Window` 라는 개념을 사용할 수 있다. 
이는 `KStream` 에 시간범위인 `Window` 를 적용하면 시간 윈도우로 집계된 `KTable` 이 반환되어, 
이러한 두 `KStream` 을 `Join` 연산 할 수 있다. 
하지만 여기서 `KTable` 간의 `Join` 에는 `Window` 기반 조인이 아님을 기억해야 한다.  

`Kafka` 는 이러한 `Window` 연산을 위해 `CreateTime`, `AppendTime` 이라는 타임스탬프를 추가했다. 
`CreateTime` 은 `Producer` 에 의해 설정되는데, 이는 수동 혹은 자동으로 설정될 수 있다. 
`AppendTime` 은 `Kafka Broker` 에 의해 메시지가 추가된 시간을 의미한다. 
이는 `Kafka Broker` 의 `Topic` 의 구성요소로 `AppendTime` 이 이미 설정돼 있는 경우, 
`Broker` 는 새로운 타임스탬프 값으로 덮어쓰게 된다.  

### Join
`Kafka Streams Join` 의 기본적인 개념은 `SQL` 에 차용되어 3가지 조인연산을 제공 하지만, 
`SQL` 의 조인 개념과 완전히 동일하지는 않다.  

.. 그림 ..

- `Inner Join` : 두 소스에서 동일한 키를 가진 레코드가 있을 때 출력값이 발생한다. 
- `Left Join` : 좌측(`Primary`)소스의 모든 레코드에 대해 출력이 발생한다. 우측 레코드에 동일한 키가 있다면 해당 레코드와 조인되고, 없는 경우는 `null` 로 설정된다. 
- `Outer Join` : 두 소스의 모든 레코드에 대해 출력이 발생한다. 두 소스 모두 동일한 키가 있다면 두 레코드가 조인되고, 한 소스에만 키가 있다면 다른 소스는 `null` 로 설정된다.  

위 그림에서 보는 것과 같이 `Kafka Streams Join` 은 `KStream`, `KTable`, `GlobalKTable` 간 조인 연산을 제공한다. 
이를 모두 정리하면 아래와 같다.  

Primary Type(Left Source)|Secondary Type(Right Source)|Inner Join|Left Join|Outer Join
---|---|---|---|---
KStream|KStream|O|O|O
KTable|KTable|O|O|O
KStream|KTable|O|O|X
KStream|GlobalKTable|O|O|X

`KStream` 과 `KTable` 은 동일 타입간은 3가지 조인 연산을 모두 제공한다. 
하지만 `KStream` 과 `KTable` 혹은 `GlobalKTable` 간 조인 연산은 `Outer Join` 은 제공되지 않는다. 
이와 관련된 자세한 내용은 [여기](https://cwiki.apache.org/confluence/display/KAFKA/KIP-99%3A+Add+Global+Tables+to+Kafka+Streams)
에서 확인 할 수 있다. 
이후 다른 포스팅에서 더 자세히 다룰 계획이지만, 
`KTable` 과 `GlobalKTable` 의 차이를 간략하게 나열하면 아래와 같다. 

|구분|KTable|GlobalKTable
---|---|---
|데이터 저장 방식|로컬 상태 저장|전역 상태 저장
|업데이트|`changelog` 기반으로 자신의 `partition` 상태만 업데이트|`changelog` 기반으로 전역 상태 업데이트
|조인 연산|동일한 파티션끼리만 가능|모든 상태가 로컬에 있으므로 제약 없음
|파티셔닝|`Source Stream` 의 파티셔닝 전략을 따름|파티셔닝을 고려할 필요 없음
|메모리 사용|상대적으로 적은 메모리 사용|모든 인스턴스에 전체 데이터를 저장하므로 사용량이 많음

