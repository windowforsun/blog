--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams 기본 개념"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에 대한 설명과 특징에 대해 알아보자'
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
public class Main {
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
}
```  

위 샘플 코드는 `input-topic` 을 부터 전달되는 이벤트 레코드를 `UpperCase` 로 변경해서 `trans-topic` 으로 이벤트를 전달하는 스트림이다. 
이렇게 `Kafka Streams` 는 `DSL` 을 사용하거나, 
`Processor API` 의 인터페이스 규칙에 맞게 `Java` 클래스를 구현하는 것만으로 `Kafka` 에 접속하고 필요한 `Consumer`, `Producer` 를 만들어 
필요한 처리를 수행하도록 할 수 있다.  

### Kafka Streams 구현
`Kafka Streams` 구현은 `Streams API` 를 사용해서 파이프라인 로직(토폴로지)를 구현하는 것을 의미한다. 
실제 구현 방식은 아래 2가지로 추상화 돼 있다. 

- `Streams DSL`
- `Processor API`

#### Streams DSL
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#streams-dsl)
은 토폴로지를 쉽게 구축하기 위해 제공하는 `DSL` 이다. 
`Streams DSL` 을 사용하면 `Lambda` 를 메서드 체인으로 연결하는 인터페이스 방식으로 스트림 애플리케이션 구현이 가능하다.  


#### Processor API
[Processor API](https://docs.confluent.io/platform/current/streams/developer-guide/processor-api.html#kstreams-processor-api)
은 `Kafka Streams` 의 `low level API` 로 위 `Streams DSL` 또한 `Processor API` 로 구현 됐다.  

`Processor API` 는 클래스 정의로 사용할 수 있으며, 
`init` 메서드 `ProcessorContext` 객체를 이용해서 `Kafka Streams` 정보를 얻어 `StateStore` 참조 혹은 후속 처리에 데이터를 전달 할 수 있다.  

## Kafka Streams 애플리케이션 모델
`Kafka Streams` 는 `Source Node`, `Processor Nodes`, `Sink Node` 를 조합해서 
`Topology` 라는 단위를 만들어 애플리케이션을 구축한다.  

### Topology
`Topology` 는 수학에서 위상 기하학을 나타내는 용어로, 
물건의 형상이 가지는 설징에 초점을 맞춘 것을 의미힌다. 
그리고 네트워크 그래프가 어떤 구성을 하고 있는지 형상 패턴을 나타내는 네트워크 토폴리지도 있다. 
`Kafka Streams` 의 `Topology` 는 앞서 설명한 네트워크 토폴리지와 동일하다고 생각하면 된다.  

`Kafka Streams` 는 역할을 갖는 노드들을 연결해서 그래프 구조로 `Topology` 를 구성하고 동작한다. 
`Topology` 는 2개 이상의 하위 토폴로지에 의해 형성돌 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/concept-kafka-stream-1.jpeg)

위와 같은 그래프 구조를 보관하고 관리하는 객체가 바로 `Kafka Streams` 의 `Topology` 이다.  

여기서 `Sub Topology` 란 `Source Node` 에서 부터 추척을 시작해 `Sink Node` 혹은 종단까지 
하나의 그래프로 관계가 연결되는 것을 하나의 `Sub Topology` 라고 한다. 
위 예에서는 `Source Node` 는 2개 이지만 
최종적으로 하나의 `Sink Node` 로 연결되기 때문에 하나의 서브 토폴로지라고 할 수 있다. 
만약 `Source Node` 가 하나로 중간에 이어지지 않고 독립적으로 구성이 되는 구조라면 2개의 서브 토폴로지가 생기게 된다.  

`Source Node` 가 특정 토픽으로 부터 데이터를 수신하면 
`Processor Node` 에서 각 역할에 맞는 처리 및 가공을 수행하거나 다른 데이터 스토어에 저장할 것이다. 
그리고 그 데이터들을 다른 토픽으로 전달이 필요하다면 `Sink Node` 를 통해 다시 처리를 이어 갈 수 있다. 
각 노드에 대한 자세한 설명은 아래와 같다.  

#### Source Node
`Kafka Topic` 에서 데이터를 받아오는 역할을 수행하는 노드를 `Source Node` 라고 한다. 
`Kafka Streams` 에는 3가지 방식의 데이터 수신 패턴이 있는데(`KStream`, `KTable`, `GlobalKTable`), 
더 자세한 내용은 `DSL` 을 다루는 내용에서 확인 할 수 있다.  

#### Processor Node
`Source Node` 에서 받은 데이터를 처리하는 노드를 의미한다. 
실질적인 비지니스 로직이 포함되는 노드가 바로 해당 노드이다. 
`Processor Node` 는 처리 과정을 분할하는 것처럼 여러개를 만들어 연결해서 구현 할 수도 있다. 
위 `샘플 코드` 에서 `peek()` 과 `mapValues()` 가 `Processor Node` 에 해당한다.  

#### Sink Node
데이터를 다른 `Kafka Topic` 에 전송 역할을 수행하는 노드를 의미한다. 
위 `샘플 코드` 에서 `to()` 가 해당 노드에 해당한다.  

### Task
`Kafka Streams` 는 여러 머신에서 분산 처리를 할 수 있도록 제공한다. 
`Kafka Streams` 은 서브 토폴로지가 `polling` 을 수행하는 소스 토픽의 파티션 수에 맞춰 작업(`Task`)의 
단위로 클라이언트에 처리를 할당한다.  

예를 들어 아래와 같은 `A` 토폴로지가 있다고 가정해보자. 

- `A` 토폴로지는 아래 2개의 서브 토폴로지로 구성된다.
  - `A-1` : 토픽 `B-1`(파티션 수 8) 참조(1_0 ~ 1_7 태스크 생성)
  - `A-2` : 토픽 `B-2`(파티션 수 10) 참조(2_0 ~ 2_9 태스트 생성)

위와 같이 `A` 토폴로지 구성을 위해서 총 18개의 태스크가 생성된다. 
18개 작업 처리는 `Consumer Group` 에 속한 클라이언트에게 할당 된다. 
할당 전략에는 몇가지 패턴이 있는데 해당 패턴에 대한 설명은 추후에 더 자세히 설명하도록 한다.


---  
## Reference
[Kafka Streams vs. Kafka Consumer](https://www.baeldung.com/java-kafka-streams-vs-kafka-consumer)  
[Introducing Kafka Streams: Stream Processing Made Simple](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/)  
[Streams Concepts](https://docs.confluent.io/platform/current/streams/concepts.html#stream)  
[Introduction Kafka Streams API](https://docs.confluent.io/platform/current/streams/introduction.html#introduction-kstreams-api)  
[Streams Architecture](https://docs.confluent.io/platform/current/streams/architecture.html#streams-architecture)  
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#creating-source-streams-from-ak)  
[Kafka Streams Processor API](https://docs.confluent.io/platform/current/streams/developer-guide/processor-api.html#kstreams-processor-api)  

