--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams DSL"
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
    - Kafka Streams DSL
toc: true
use_math: true
---  

## Streams DSL
`Kafka Streams DSL` 은 `Processor API` 를 사용해서 구현 돼있고, 
대부분의 데이터 처리를 `DSL` 을 사용하면 초보자도 간단하게 표현 할 수 있다. 

`Streams DLS` 을 `Processor API` 와 비교하면 아래와 같다. 

- `Streams DSL` 은 `KStream`, `KTable`, `GlobalKTable` 과 같은 스트림과 테이블의 추상화 구현체르 제공한다. 
- `Streams DSL` 은 `stateless transformation`(`map`, `filter`, ..) 와 `stateful transformation`(`count`, `reduce`, `join`, ..) 에 대한 동작을 함수형 스타일로 제공한다. 

`Streams DLS` 을 사용해서 `Topology` 를 구성하는 절차는 아래와 같다. 
1. `Kafka Topic` 에서 데이터를 읽은 `Input Stream` 을 하나 이상 생성한다. 
2. `Input Stream` 을 처리하는 `Transformation` 을 구성한다. 
3. `Ouput Stream` 을 생성해서 결과를 `Kafka Topic` 에 저장한다. 혹슨 `Interactive queries` 를 사용해서 결과를 `REST API` 와 같은 것으로 노출 시킨다. 


### Source Streams 
`Kafka Topic` 에 있는 데이터는 `KStream`, `KTable`, `GlobalTable` 을 사용해서 간편하게 읽어 올 수 있다.  

#### KStream (input topic -> KStream)
`input topic` 역할을 하는 `Kafka Topic` 에 `KStream` 을 생성해서 데이터를 읽어올 수 있다. 
`KStream` 은 분할된 레코드 스트림이다. 
여기서 분할된 레코드 스트림이란 스트림 애플리케이션이 여러 서버에 실행 된다면, 
각 애플리케이션의 `KStream` 은 `input topic` 의 특정 파티션 데이터로 채워진다. 
분할된 `KStream` 을 하나의 큰 집합으로 본다면 `input topic` 의 모든 파티션의 데이터가 처리된다. 

```java

```  







---  
## Reference
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#transform-a-stream)  
