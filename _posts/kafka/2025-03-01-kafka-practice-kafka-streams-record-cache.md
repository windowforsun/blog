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


### Record caches in Streams DSL
`Streams DSL` 에서는 `Topology` 인스턴스에 대한 `Record Cache` 의 전체 메모리 크기를 지정할 수 있다. 
이렇게 설정된 전체 메모리 크기는 아래와 같은 `KTable` 인스턴스에서 사용된다. 

- Source KTable : `StreamsBuilder.table()`, `StreamsBuilder.globalTable()` 을 통해 생성된 `KTable` 인스턴스
- Aggregation KTable : `Aggregating` 연산의 결과로 생성된 `KTable` 인스턴스

이런 `KTable` 인스턴스에서 `Record Cache` 는 아래와 같은 용도로 사용된다.  

- 내부 캐싱 및 출력 레코드 압축 : 내부 상태 저장소에 기록되기 전에 출력 레코드를 캐싱하고 압축한다. 
- 다운스트림으로 전달되기 전에 레코드 캐싱 및 압축 : 상태 프로세서 노드에서 다운 스트림으로 전달되기 전의 레코드를 캐싱하고 압축한다.  

`Record Cache` 를 사용하는 경우와 사용하지 않는 경우 차이를 보이기 위해 다음과 같은 입력 스트림을 예로 든다. 
`<K, V>` 와 같은 구조로 `<A, 1>, <D, 5>, <A, 20>, <A, 300>` 과 같은 입력 입력 스트림이 들어오고, 
집계 동작은 `K` 별 `V` 값을 합산하는 동작이다. 
`Record Cache` 의 특징을 살펴보기 위해 `K` 가 `A` 인 집게 변화만 살펴본다. 
집계 결과의 표현은 `<K, (afterValue, beforeValue)>` 와 같이 한다.  

- `Disable record cache` : `<A, (1, null)>, <A, (21, 1)>, <A, (321, 21)>` 와 같이 캐시가 비활성화된 경우 `A` 의 레코드 수 만큼 집계 변화가 모두 상태 저장소에 기록되고 다운스트림으로 전달된다. 
- `Enable record cache` : 설정에 따라 달라질 수 있지만, 출력 레코드가 캐시 내에서 압축되어, 단일 레코드인 `<A, (321, null)>` 만 남게된다. 해당 레코드는 집계의 내부 상태 저장소에 기록되고 다운스트림으로 전달된다. 

`Record Cache` 의 크기는 `Topology` 별 전역 설정인 `statestore.cache.max.bytes` 설정을 통해 가능하다. (기본값 : 10MB)

```java
Properties streamsConfiguration = new Properties();
streamsConfiguration.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 10 * 1024 * 1024L);
```  

설정한 캐시의 크기 설정 값은 하나의 `Topology` 에서 사용 가능한 캐시의 크기이다. 
`Topology` 가 `T` 개의 스레드로 구성돼 있고, `C` 바이트로 캐시크기가 설정돼 있다면 각 스레드는 `C/T` 바이트를 고르게 할당 받는다. 
이는 스레드 수만큼 개별 캐시가 존재하고 스레드 간 캐시 공유는 되지 않는 다는 의미이다.   


`Record Cache` 는 설정한 캐시 크기가 꽉차면 `LRU` 방식으로 기존 레코드를 제거한다. 
입력 스트림으로 부터 키가 `K1` 인 레코드 `<K1, V1>` 가 처음 들어오면 해당 레코드는 캐시에서 `dirty` 상태로 표시한다. 
그리고 이후 동일한 키 `K1` 인 레코드 `<K1, V2>` 가 동일한 노드에서 처리되면 해당 레코드가 이전에 들어온 `<K1, V1>` 을 덮어쓴다. 
이러한 동작을 `compacting` 이라고 하고, 이는 `Kafka` 의 로그 압축과 유사한 효과를 지닌다. 
하지만 `Record Cache` 의 특성을 수행되는 동작들은 모두 클라이언트 애플리케이션내에서 이뤄진다. 
즉 `Kafka Broker` 와는 상호작용은 수행되지 않는 다는 의미이다. 
이후 `Record Cache` 가 조건에 의해서 `flush` 되면 `<K1, V2>` 레코드는 다운스트림으로 전달되고 로컬 상태 저장소에도 기록된다.  
