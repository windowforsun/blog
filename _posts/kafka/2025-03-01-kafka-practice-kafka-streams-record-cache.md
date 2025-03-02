--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Record Cache"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 성능을 최적화 할 수 있는 Record Cache 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Record Cache
    - State Store
    - Topology
    - Processor API
    - Streams DSL
toc: true
use_math: true
---  

## Kafka Streams Record cache
`Kafka Streams` 에서 `Record Cache` 는 `State Store` 에서 메모리를 활용해 데이터를 임시로 저장하여 디스크 `I/O` 를 줄이고, 
성능을 최적화하는 특성을 의미한다. 
`Stateful` 연산에서(`aggregate()`, `reduce()`, `count()` 등)을 수행할 떄,
상태 저장소로의 데이터 접근을 최적화하는 방법으로 캐시가 사용된다. 
이러한 캐시는 디스크로 직접 쓰기 전에 메모리에 기록해주어 성능을 처리 성능을 향상시키는데 중요한 역할을 한다. 
만약 이러한 캐시가 없다면 `Stateful` 연산은 빈번하게 상태저장소에 접근하게 된다. 
이러한 빈번한 접근은 디스크 기반 `I/O` 를 사용하기 때문에 성능에 큰 영향을 줄 수 있으므로 
`Recrod Cache` 를 적절히 활용하면 비교적 높은 처리 성능을 기대할 수 있다.  

`Kafka Streams` 에서는 이러한 `Record Cache` 를 위해 상태 저장소의 최대 캐시 크기를 지정하는 `statestore.cache.max.bytes` 와 
캐시의 `flush` 시간을 지정하는 `commit.interval.ms` 설정 값을 제공한다. 
`Record Cache` 의 실제 특성은 `Streams DSL` 과 `Processor API` 에서 약간 차이가 있어 구분해서 설명한다.  
