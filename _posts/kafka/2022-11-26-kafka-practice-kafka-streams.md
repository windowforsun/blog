--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams"
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
    - Stream Processing
toc: true
use_math: true
---  

## Streams Processing 이란 
`Streams Processing` 이란 이벤트 스트림에 데이터가 도착할 때 마다 처리를 계속 이어가는 애플리케이션을 의미힌다. 
`Kafka` 애플리케이션 중에서는 `Kafka Broker` 에 대해서 `Consumer` 가 무한 루프로 `Polling` 해서 일정 간격마다 처리를 실행하는 애플리케이션이 될 수 있다. 
또한 단일 레코드에 대해서 처리할 수도 있지만 집계 단위를 두고 벌크로 집계를 실행하는 경우도 있을 수 있다.  

### Stateless, Stateful
`Stream Processing` 의 처리는 `Stateless` 와 `Stateful` 로 나눌 수 있다.  

`Stateless` 란 이벤트를 통해 레코드가 전달 됐을 때, 해당 레코드만 처리해서 완료하는 것을 의미힌다. 
예를 들어 전달 받은 `A` 레코드를 `B` 레코드로 변환해 다른 토픽 혹은 데이터 스토어에 전송하는 처리가 있다.  

`Stateful` 은 단일 레코드에 대해서 처리하기 보다는 이벤트를 통해 전달된 레코드를 일정 기간 보관 후, 
다수의 레코드에 대해 집계 혹은 조합해서 결과를 생성하는 처리이다. 
이벤트에 대해서 발생 횟수 합산, 평균, 통계 산출 등등 이 있을 것이다. 
이는 스크림과 데이터 스토어 등을 활용해서 데이터의 질을 높이는 처리도 포함된다.  

`Stateful` 처리는 설명한 것처럼 별도의 데이터 스토어가 필요하다. 
이를 `StateStore` 라고 부르는데 이는 `Stream Processing` 의 구현에 있어서 고려할 포인트가 존재한다.  


### State Store
`Stream Processing` 은 이벤트를 통해 레코드가 전달 될 떄마다 처리가 수행되기 때문에 낮은 대기 시간이 필요하다. 
그러므로 `Redis` 와 같은 `Key-Value` 스토리지를 사용한다 하더라도 성능적으로 충분하지 않을 수 있다. 
그래서 기본적으로 각 처리 노드의 로컬에 데이터 스토어를 보관 유지하는 방식으로 레이턴시를 낮춰 성능을 유지하는 방식을 사용한다.  

대표적으로 `Kafka Streams`, `Apache Flink` 등의 `Stream Processing` 프레임워크에서는 `RocksDB` 가 사용된다. 
`RocksDB` 는 애플리케이션에 통합되는 `Key-Value` 스토리지로 노드의 로컬에 `RocksDB` 를 두어 대기 시간을 최소화 시킨다.  

하지만 위와 같은 방식은 `Stream Processing` 처리 자체가 노드에 크게 의존하게 된다. 
특정 노드가 다운되고 복구가 불가능하게 된다면 데이터가 손실까지 될 수 있다. 
이러한 장애상황을 위해 노드 독립적인 데이터 지속성 메커니즘도 필요하다. 
하지만 이러한 장애상황 대응은 네트워크 통신이 필요하고 이는 오버헤드 발생하기 때문에 어느정도 버퍼링은 피할 수 없다.  

`Kafka` 의 경우를 살펴보자. 
`Kafka` 를 통해 `Stream Processing` 을 구현한다면 확장성을 위해 애플리케이션을 분산해 어려 노드를 사용해 처리가 수행될 수 있다. 
위와 같은 구조에서는 노드 증가에 따라 애플리케이션이 연결돼 사용하는 파티션 할당이 변경될 수 있다. 
`Kafka Consumer` 는 자신이 포함된 `Consumer Group` 에서 파티션과 1:1 매칭 되기 때문에 노드가 증가 되면, 
재배치 작업을 통해 연결된 파티션이 변경될 수 있다. 
이는 `Stateful` 처리에 있어서는 이전에 사용하던 `StateStore` 가 모두 초기화 되는 것이기 때문에 다른 결과를 만들어 낼 수 있다. 
그러므로 노드 전환에 따라 `StateStore` 또한 함께 재배치하는 메커니즘도 필요하다.  

## Kafka Streams
`Kafka Streams` 는 `Apache Kafka` 에서 공식적으로 제공하는 `Stream Processing` 프레임워크로 `Java` 를 기반해서 개발됐다.  

`Stateless` 처리는 비교적 간단하지만, `Stateful` 처리는 `StateStore` 관리가 필요하므로 복잡성이 추가로 요구된다. 
하지만 `Kafka Streams` 프레임워크를 사용하면 추가적으로 발생할 수 있는 복잡성을 간소화할 수 있다는 장점이 있다. 
물론 `Stateful` 뿐만 아니라 `Stateless` 처리 애플리케이션 구현에 있어서도 복잡성을 크게 줄일 수 있다.  

`Kafka Streams` 에 대한 더 자세한 설명은 [공식문서](https://docs.confluent.io/platform/current/streams/concepts.html)
에서 확인 할 수 있다.  

### 특징 
#### 고기능(Powerful)
- 애플리케이션의 높은 확장성, 탄력성, 분산성, 내결합성 구현
- 정확히 단 한번 처리 수행의 처리 시멘틱스 지원
- `Stateless`, `Stateful` 처리 지원
- `windowing`, `joins`, `aggregation` 를 사용한 이벤트 시간 프로세싱
- 스트림과 데이터베이스를 통합하기 위한 `Kafka Streams` 의 대화형 쿼리 지원
- 선언형 함수형 `API` 와 하위 레벨 명령형 `API` 을 통해 높은 제어성과 유연성 제공

### 가벼움(Lightweight)
- 낮은 잔입 장벽
- 소규모, 중규모, 대규모, 특대규모 모두 사용 가능
- 로컬 개발 환경에서 대규모 실 환경으로 원활한 전환
- 처리 클러스터 필요 없음
- `Kafka` 외 다른 외부 의존성 없음

### Java 샘플 코드 

```groovy
dependencies {
    implementation 'org.apache.kafka:kafka-streams:3.2.0'
}
```  

```java
public static void main(String[] args) {
	StreamsBuilder builder = new StreamsBuilder();
	builder
			.stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()))
			.peek((key, value) -> System.out.println("input peek : " + value))
			.mapValues(value -> value.toUpperCase())
			.peek((key, value) -> System.out.println("trans peek : " + value))
			.to("trans-topic", Produced.with(Serdes.String(), Serdes.String()));

	Topology topology = builder.build();
	// or
	// topology.addProcessor(..);

	Properties props = new Properties();
	props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-app");
	props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
	props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
	props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());

	KafkaStreams kafkaStreams = new KafkaStreams(topology, props);
	// non blocking
	kafkaStreams.start();
}
```  

위 샘플 코드는 `input-topic` 을 부터 전달되는 이벤트 레코드를 `UpperCase` 로 변경해서 `trans-topic` 으로 이벤트를 전달하는 스트림이다. 
이렇게 `Kafka Streams` 는 `DSL` 을 사용하거나, 
`Processor API` 의 인터페이스 규칙에 맞게 `Java` 클래스를 구현하는 것만으로 `Kafka` 에 접속하고 필요한 `Consumer`, `Producer` 를 만들어 
필요한 처리를 수행하도록 할 수 있다.  

## Kafka Streams 애플리케이션 모델

### Topology




---  
## Reference
[Kafka Streams vs. Kafka Consumer](https://www.baeldung.com/java-kafka-streams-vs-kafka-consumer)  
[Kafka Streams With Spring Boot](https://www.baeldung.com/spring-boot-kafka-streams)  
[Introducing Kafka Streams: Stream Processing Made Simple](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/)  
[Streams Concepts](https://docs.confluent.io/platform/current/streams/concepts.html#stream)  
[Introduction Kafka Streams API](https://docs.confluent.io/platform/current/streams/introduction.html#introduction-kstreams-api)  
[Streams Architecture](https://docs.confluent.io/platform/current/streams/architecture.html#streams-architecture)  
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#creating-source-streams-from-ak)  

