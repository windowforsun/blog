--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams State Store Type"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 사용할 수 있는 State Store 의 종류와 특징에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - State Store
    - In-Memory
    - Persistent
    - KeyValueStore
    - VersionedStateStore
    - SessionStore
    - WindowStore
    - TimestampKeyValueStore
    - TimestampWindowStore
    - RocksDB
    - Change-Log Topic
toc: true
use_math: true
---  

## Kafka Streams State Store Type
`Kafka Streams` 에서 `State Store` 은 스트리밍 애플리케이션이 상태를 유지할 수 있도록 하는 중요한 구성 요소이다. 
`State Store` 은 데이터 처리 중에 필요한 상태(e.g. 집계, 윈도우, 조인, ..)를 저장하고, 
이를 통해 스트림 처리를 바탕으로 필요한 비지니스를 구현할 수 있다. 
`Kafka Streams` 에서는 크게 `In-Memory State Store` 와 `Persistent State Store` 라는 두 가지 유형의 상태 저장소가 있다.  

이후 설명하는 `State Store` 의 설명은 각 유형별 특징과 차이점에 초점을 맞춘 내용이다. 
`State Store` 에 대한 전반적인 내용은 [여기](https://github.com/windowforsun/kafka-streams-state-store-type)
에서 확인 가능하다.  


### In-Memory State Store
`In-Memory State Store` 은 애플리케이션의 메모리에 상태 데이터를 저장한다. 
메모리에 저장된 데이터는 디스크에 기록되지 않기 때문에 애플리케이션이 재시작되거나 장애기 발생하면 해당 데이터는 사라져 `State Persistent` 를 제공하지 않는다. 
이렇게 데이터가 메모리에 저장되는 만큼 읽고 쓰기가 매우 빠른 접근 속도를 제공한다. 
이는 지연을 최소화하고 빠른 실시간 처리가 필요한 애플리케이션에서 유리하다. 
그리고 모메리 용량에 따라 최대로 저장할 수 있는 데이터의 양이 제한된다. 
큰 데이터를 다루거나, 상태 크기가 커지는 경우에는 적합하지 않을 수 있다.

### Persistent State Store
`Persistent State Store` 는 상태 데이터를 디스크에 저장한다. 
`RocksDB` 와 같은 `key-value` 저장소를 사용해 데이터를 관리하며, 
애플리케이션 재시작이 되더라도 상태가 유지될 수 있는 `State Persistent` 를 제공한다. 
디스크에 상태를 저장하기 때문에 애플리케이션 종료나 장애상황에서도 데이터가 유지될 수 있기 때문에 복구 관점에서 매우 유리하다. 
하지만 읽고 쓰기의 동작이 디스크 I/O에 크게 의존하기 때문에 `In-Memory State Store` 보다는 성능적으로 불리할 수 있다. 
그렇지만 `RocksDB` 는 고성능 데이터저장소이므로 이런 성능 저하를 최소화 할 수 있다.   


> 여기서 주의해야할 점은 `In-Memory State Store` 와 `Persistent State Store` 의 가장 큰 차이점은 `State Persistent` 의 제공 여부이다. 
즉 `Fault-Tolerance` 보장 관점에서는 두 저장소 유형 모두 이를 제공한다는 의미이다. 
`Kafka Streams State Store` 는 `change-log topic` 을 바탕으로 `State Store` 의 `Fault-Tolerance` 를 제공한다. 
`State Store` 의 변경상태를 `change-log topic` 에 기록하고 이러한 싱태변경 기록을 바탕으로 애플리케이션이 재시작되거나 
장애가 발생했을 때 애플리케이션에서 상태를 복구할 수 있도록 한다. 
이는 `In-Memory State Store`, `Persistent State Store` 모두 재시작, 장애 상황에서도 현 상태를 복구할 수 있는 매커니즘은 존재한다는 의미이다. 
하지만 `Persistent State Store` 는 해당하는 상태파일이 저장소에 있다면 `change-log topic` 을 바탕으로 복구를 진행하지 않고, 
`In-Memory State Stre` 는 매번 `change-log topic` 을 바탕으로 상태 복구가 진행될 수 있기 때문에 저장소 크기에 따라 복구 성능에는 차이가 있을 수 있다. 
이에 대한 자세한 내용은 이후에 다시 다루도록 한다.  


### State Store Type
`Kafka Streams` 을 사용해서 `Topology` 를 구성할 때 사용할 수 있는 `State Store` 에는 어떤 유형이 있는지 알아본다. 
이와 관련된 전체 소스 코드는 [여기](https://github.com/windowforsun/kafka-streams-state-store-type)
에서 확인 할 수 있다.  

사용 가능한 `State Store` 종류별 특징을 확인하기 위해 예제 스트림은 투표결과를 카운트하는 비지니스를 구현한다. 
이를 통해 동일한 투표 스트림 데이터가 들어올 때 각 `State Store` 가 어떤 결과를 도출하는 지 확인하는 과정으로 각 상태 저장소의 특징과 차이를 알아본다.  
아래는 예제에 대한 메시지와 `State Store` 기반 처리 과정을 도식화한 것이다. 


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-1.drawio.png)


위 메시지 스트림에서 주의해야할 부분은 `voter5` 의 투표이다. 
`a` 로 투표한 메시지가 먼저 도착한 후, `b` 로 변경된 투표가 도착한 것을 볼 수 있다. 
하지만 먼저 도착한 `a` 의 시간값이 더 최신이고, 
`b` 로 변경된 투표는 과거이므로 `voter5` 는 실제로는 `b` 로 투표를 한 후 `a` 로 변경했지만 시스템의 문제로 메시지 순서가 변경된 것이다. 
각 `State Store` 마다 이런 상황에서 어떠한 결과를 보이는지도 함께 살펴보고자 한다. 
추가로 이러한 순서가 바뀐 경우 순서를 보장하도록 처리할 수 있는 방안이 `VersionedStateStore` 인데 해당 포스팅에서는 간단한 개념만 다루고, 
자세한 내용은 [여기]() 에서 확인 할 수 있다.  

#### KeyValueStore
간단한 `key-value` 저장소이다. 
각 키에 대한 단일 값을 저장할 수 있고, 
이 값은 동일한 키로 업데이트될 수 있다. 
가장 단순한 형태의 상태 저장소로, 이후 설명하는 저장소들과 같이 세션이나 시간과 무관하게 `key-value` 쌍을 저장한다. 
그리고 `In-Memory`, `Persistent` 타입의 `State Store` 를 모두 제공한다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-2.drawio.png)


예제 코드 테스트 결과를 보면 알 수 있듯이, 
`input-topic` 으로 들어오는 투표자들의 모든 투표를 통합해서 각 후보마다 카운트하는 결과를 도출하게 된다. 
순서가 뒤바뀐 `voter5` 에 대한 투표는 시간 기준으로 먼저 투표한 결과만 반영된 결과를 보이기 때문에 실제로는 이런 상황에서 옳바른 결과를 도출하지 못하는 것을 확인할 수 있다.  

아래는 `inMemoryKeyValueStore` 을 사용한 예제이다. 


```java
public void inMemoryKeyValueStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.<String, String>stream("input-topic");

    // count
    inputTopicStream
        .toTable(
            Materialized.<String, String>as(Stores.inMemoryKeyValueStore("in-memory-key-value-store"))
                .withKeySerde(Serdes.String())
                .withValueSerde(Serdes.String()))
        .groupBy((voter, item) -> KeyValue.pair(item, item))
        .count(Materialized.as(Stores.inMemoryKeyValueStore("in-memory-key-value-store-count")))
        .toStream()
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void inMemoryKeyValueStore() {
	this.inMemoryStateStore.inMemoryKeyValueStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	KeyValueStore<String, Long> outputStore = this.topologyTestDriver.getKeyValueStore("in-memory-key-value-store-count");

	assertThat(outputStore.get("a").longValue(), is(2L));
	assertThat(outputStore.get("b").longValue(), is(2L));
	assertThat(outputStore.get("c").longValue(), is(2L));

	List<KeyValue<String, Long>> outputStream = this.outputTopic.readKeyValuesToList();

	assertThat(outputStream, hasSize(8));

	assertThat(outputStream.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputStream.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputStream.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputStream.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputStream.get(5), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(6), is(KeyValue.pair("b", 2L)));
	assertThat(outputStream.get(7), is(KeyValue.pair("c", 2L)));
}
```  

그리고 아래는 `persistentKeyValueStore` 의 사용 예시이다.  

```java
public void persistentKeyValueStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .toTable(
                Materialized.<String, String>as(Stores.persistentKeyValueStore("persistent-key-value-store"))
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.String()))
        .groupBy((voter, item) -> KeyValue.pair(item, item))
        .count(Materialized.as(Stores.persistentKeyValueStore("persistent-key-value-store-count")))
        .toStream()
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void persistentKeyValueStore() {
	this.persistentStateStore.persistentKeyValueStore(this.streamsBuilder);
	this.startStream();
	
	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);


	KeyValueStore<String, Long> outputStore = this.topologyTestDriver.getKeyValueStore("persistent-key-value-store-count");
	outputStore.all().forEachRemaining(System.out::println);

	assertThat(outputStore.get("a").longValue(), is(2L));
	assertThat(outputStore.get("b").longValue(), is(2L));
	assertThat(outputStore.get("c").longValue(), is(2L));

	List<KeyValue<String, Long>> outputStream = this.outputTopic.readKeyValuesToList();
	outputStream.forEach(System.out::println);

	assertThat(outputStream, hasSize(8));

	assertThat(outputStream.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputStream.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputStream.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputStream.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputStream.get(5), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(6), is(KeyValue.pair("b", 2L)));
	assertThat(outputStream.get(7), is(KeyValue.pair("c", 2L)));
}
```  

#### SessionStore
`Session` 기반의 저장소로 특정 키에 대한 연속적인 이벤트 그룹을 저장한다. 
`SessionStore` 는 각 키에 대해 `Session` 을 관리하며, 세션이 끝난 후 특정 기간 동안 저장된 유지한다. 
`SessionStore` 는 [여기]() 
에서 좀 더 자세한 내용을 확인 할 수 있다. 
즉 `SessionStore` 를 사용하면 키 별 이벤트 발생을 기준으로 그룹화를 시작하고 정해진 `inactivityGap` 시간 동안 해당 `Session` 이 유지되며 들어오는 이벤트를 하나의 그룹으로 구성한다. 
그리고 그룹화 된 데이터는 `retentionPeriod` 동안 저장소에 유지되는 방식이다.  
`SessionStore` 도 `In-Memory`, `Persistent` 저장소에서 모두 사용할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-3.drawio.png)


위 결과는 `inactivityGap=10ms` 이고 `retentionPeriod=100ms` 로 설정된 상태에서 결과이다. 
`voter3` 과 `voter6` 이 투표한 `c` 결과를 보면 `t=3` 에 한번 `t=30` 에 한번씩 메시지가 들어오는 상황에서 
`inactivityGap=10ms` 을 지난 이벤트이므로 다른 그룹으로 구성된 것을 확인 할 수 있다.  

아래는 `inMemorySessionStore` 을 사용한 예제이다. 

```java
public void inMemorySessionStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .groupBy((voter, item) -> item)
        .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMillis(10)))
        .count(Materialized.<String, Long>as(
                Stores.inMemorySessionStore("in-memory-session-store-count", Duration.ofMillis(100)))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .map((stringWindowed, aLong) -> KeyValue.pair(stringWindowed.key(), aLong))
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void inMemorySessionStore() {
	this.inMemoryStateStore.inMemorySessionStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	SessionStore<String, Long> outputStore = this.topologyTestDriver.getSessionStore("in-memory-session-store-count");

	assertThat(outputStore.fetchSession("a", 1l, 10L), is(3L));
	assertThat(outputStore.fetchSession("b", 2L, 8L), is(2L));
	assertThat(outputStore.fetchSession("c", 3L, 3L), is(1L));
	assertThat(outputStore.fetchSession("c", 30L, 30L), is(1L));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(10));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(nullValue()));
	assertThat(outputStream.get(3).timestamp(), is(1L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(2L));
	assertThat(outputStream.get(4).timestamp(), is(5L));

	assertThat(outputStream.get(5).key(), is("a"));
	assertThat(outputStream.get(5).value(), is(nullValue()));
	assertThat(outputStream.get(5).timestamp(), is(5L));

	assertThat(outputStream.get(6).key(), is("a"));
	assertThat(outputStream.get(6).value(), is(3L));
	assertThat(outputStream.get(6).timestamp(), is(10L));

	assertThat(outputStream.get(7).key(), is("b"));
	assertThat(outputStream.get(7).value(), is(nullValue()));
	assertThat(outputStream.get(7).timestamp(), is(2L));

	assertThat(outputStream.get(8).key(), is("b"));
	assertThat(outputStream.get(8).value(), is(2L));
	assertThat(outputStream.get(8).timestamp(), is(8L));

	assertThat(outputStream.get(9).key(), is("c"));
	assertThat(outputStream.get(9).value(), is(1L));
	assertThat(outputStream.get(9).timestamp(), is(30L));
}
```  

그리고 아래는 `persistentSessionStore` 를 사용한 예제이다.  

```java
public void persistentSessionStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .groupBy((voter, item) -> item)
        .windowedBy(SessionWindows.ofInactivityGapAndGrace(Duration.ofMillis(10), Duration.ofMillis(10)))
        .count(Materialized.<String, Long>as(
                Stores.persistentSessionStore("persistent-session-store-count", Duration.ofMillis(100)))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .map((stringWindowed, aLong) -> KeyValue.pair(stringWindowed.key(), aLong))
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void persistentSessionStore() {
	this.persistentStateStore.persistentSessionStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);


	SessionStore<String, Long> outputStore = this.topologyTestDriver.getSessionStore("persistent-session-store-count");

	assertThat(outputStore.fetchSession("a", 1l, 10L), is(3L));
	assertThat(outputStore.fetchSession("b", 2L, 8L), is(2L));
	assertThat(outputStore.fetchSession("c", 3L, 3L), is(1L));
	assertThat(outputStore.fetchSession("c", 30L, 30L), is(1L));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(10));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(nullValue()));
	assertThat(outputStream.get(3).timestamp(), is(1L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(2L));
	assertThat(outputStream.get(4).timestamp(), is(5L));

	assertThat(outputStream.get(5).key(), is("a"));
	assertThat(outputStream.get(5).value(), is(nullValue()));
	assertThat(outputStream.get(5).timestamp(), is(5L));

	assertThat(outputStream.get(6).key(), is("a"));
	assertThat(outputStream.get(6).value(), is(3L));
	assertThat(outputStream.get(6).timestamp(), is(10L));

	assertThat(outputStream.get(7).key(), is("b"));
	assertThat(outputStream.get(7).value(), is(nullValue()));
	assertThat(outputStream.get(7).timestamp(), is(2L));

	assertThat(outputStream.get(8).key(), is("b"));
	assertThat(outputStream.get(8).value(), is(2L));
	assertThat(outputStream.get(8).timestamp(), is(8L));

	assertThat(outputStream.get(9).key(), is("c"));
	assertThat(outputStream.get(9).value(), is(1L));
	assertThat(outputStream.get(9).timestamp(), is(30L));
}
```  

#### WindowStore
`Window` 기반의 저장소이다. 
특정 키에 대해 시간 기반 윈도우를 설정하고, 해당 기간 동안 발생한 모든 데이터르 그룹화해 저장한다. 
각 윈도우에 대해 데이터를 관리하며, 윈도우가 종료되면 데이터를 정리할 수 있다. 
`WindowStore` 도 [여기]()
에서 더 다양하고 자세한 종류에 대한 설명을 확인할 수 있다.  
즉 `WindowStore` 는 `windowSize` 라는 시간 범위안에 발생한 키별 이벤트를 그룹화하고, 
필요에 따라 그룹화한 데이터 유지기한인 `retentionPeriod` 또는 윈도우 업데이트 유효시간인 `graceTime` 를 정해 구성할 수 있다. 
`WindowStore` 도 `In-Memory`, `Persistent` 저장소에서 모두 사용할 수 있다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-4.drawio.png)


위 결과는 `windowSize=10ms`, `graceTime=10ms`, `retentionPeriod=100ms` 로 설정했을 떄의 결과이다. 
`WindowStore` 의 시간 범위는 `[0~10)`(0이상 10미만)이므로 `a` 에 투표한 `voter1` 과 `voter4` 는 동일한 윈도우에 속했지만, 
`voter5` 의 투포는 동일한 윈도우에 포함되지 않을 것을 확인 할 수 있다.  

아래는 `inMemoryWindowStore` 의 사용 예시이다.  

```java
public void inMemoryWindowStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .groupBy((voter, item) -> item)
        .windowedBy(TimeWindows.of(Duration.ofMillis(10)))
        .count(Materialized.<String, Long>as(
                Stores.inMemoryWindowStore("in-memory-window-store-count", Duration.ofMillis(100),
                    Duration.ofMillis(10), false))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .map((stringWindowed, aLong) -> KeyValue.pair(stringWindowed.key(), aLong))
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void inMemoryWindowStore() {
	this.inMemoryStateStore.inMemoryWindowStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	WindowStore<String, Long> outputStore = this.topologyTestDriver.getWindowStore("in-memory-window-store-count");

	assertThat(outputStore.fetch("a", 0L, 10L).next().value, is(2L));
	assertThat(outputStore.fetch("a", 10L, 20L).next().value, is(1L));
	assertThat(outputStore.fetch("b", 0L, 10L).next().value, is(2L));
	assertThat(outputStore.fetch("c", 0L, 10L).next().value, is(1L));
	assertThat(outputStore.fetch("c", 30L, 40L).next().value, is(1L));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(7));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(2L));
	assertThat(outputStream.get(3).timestamp(), is(5L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(1L));
	assertThat(outputStream.get(4).timestamp(), is(10L));

	assertThat(outputStream.get(5).key(), is("b"));
	assertThat(outputStream.get(5).value(), is(2L));
	assertThat(outputStream.get(5).timestamp(), is(8L));

	assertThat(outputStream.get(6).key(), is("c"));
	assertThat(outputStream.get(6).value(), is(1L));
	assertThat(outputStream.get(6).timestamp(), is(30L));
}
```  

그리고 아래는 `persistentWindowStore` 의 사용 예시이다.  

```java
public void persistentWindowStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .groupBy((voter, item) -> item)
        .windowedBy(TimeWindows.of(Duration.ofMillis(10)))
        .count(Materialized.<String, Long>as(
                Stores.persistentWindowStore("persistent-window-store-count", Duration.ofMillis(100),
                    Duration.ofMillis(10), false))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .map((stringWindowed, aLong) -> KeyValue.pair(stringWindowed.key(), aLong))
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}


@Test
public void persistentWindowStore() {
	this.persistentStateStore.persistentWindowStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	WindowStore<String, Long> outputStore = this.topologyTestDriver.getWindowStore("persistent-window-store-count");

	assertThat(outputStore.fetch("a", 0L, 10L).next().value, is(2L));
	assertThat(outputStore.fetch("a", 10L, 20L).next().value, is(1L));
	assertThat(outputStore.fetch("b", 0L, 10L).next().value, is(2L));
	assertThat(outputStore.fetch("c", 0L, 10L).next().value, is(1L));
	assertThat(outputStore.fetch("c", 30L, 40L).next().value, is(1L));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(7));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(2L));
	assertThat(outputStream.get(3).timestamp(), is(5L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(1L));
	assertThat(outputStream.get(4).timestamp(), is(10L));

	assertThat(outputStream.get(5).key(), is("b"));
	assertThat(outputStream.get(5).value(), is(2L));
	assertThat(outputStream.get(5).timestamp(), is(8L));

	assertThat(outputStream.get(6).key(), is("c"));
	assertThat(outputStream.get(6).value(), is(1L));
	assertThat(outputStream.get(6).timestamp(), is(30L));
}
```  


#### TimestampKeyValueStore
`Timestamp` 가 포함된 `KeyValueStore` 이다.
`KeyValueStore` 의 각 `key-value` 쌍에 대해 `Timestamp` 를 관리한다.
각 키의 현재 값이 마지막으로 업데이트된 `Timestamp` 를 확인할 수 있다.
`TimestampKeyValueStore` 는 `Persistent` 저장소에서만 사용 가능하다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-6.drawio.png)


위 그림을 보면 `TimestampKeyValueStore` 에 실제로 저장된 값에는 각 키가 가장 최근에 업데이트된 타임스탬프도 함께 구성되는 것을 확인할 수 있다.
처리나 동작은 기존 `KeyValueStore` 와 동일하다.

아래는 `persistentTimestampKeyValueStore` 의 사용 예시이다.

```java
public void persistentTimestampKeyValueStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .toTable(
            Materialized.<String, String>as(Stores.persistentTimestampedKeyValueStore("persistent-timestamp-key-value-store"))
                .withKeySerde(Serdes.String())
                .withValueSerde(Serdes.String()))
        .groupBy((voter, item) -> KeyValue.pair(item, item))
        .count(Materialized.<String, Long>as(
                Stores.persistentTimestampedKeyValueStore("persistent-timestamp-key-value-store-count"))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void persistentTimestampKeyValueStore() {
	this.persistentStateStore.persistentTimestampKeyValueStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	KeyValueStore<String, ValueAndTimestamp<Long>> outputStore = this.topologyTestDriver.getTimestampedKeyValueStore("persistent-timestamp-key-value-store-count");

	assertThat(outputStore.get("a"), is(ValueAndTimestamp.make(2L, 10L)));
	assertThat(outputStore.get("b"), is(ValueAndTimestamp.make(2L, 8L)));
	assertThat(outputStore.get("c"), is(ValueAndTimestamp.make(2L, 30L)));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(8));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(2L));
	assertThat(outputStream.get(3).timestamp(), is(5L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(3L));
	assertThat(outputStream.get(4).timestamp(), is(10L));

	assertThat(outputStream.get(5).key(), is("a"));
	assertThat(outputStream.get(5).value(), is(2L));
	assertThat(outputStream.get(5).timestamp(), is(10L));

	assertThat(outputStream.get(6).key(), is("b"));
	assertThat(outputStream.get(6).value(), is(2L));
	assertThat(outputStream.get(6).timestamp(), is(8L));

	assertThat(outputStream.get(7).key(), is("c"));
	assertThat(outputStream.get(7).value(), is(2L));
	assertThat(outputStream.get(7).timestamp(), is(30L));
}
```  



#### TimestampWindowStore
`Timestamp` 가 포함된 `WindowStore` 이다.
기본적으로 `WindowStore` 가 관리하는 시간 값은 현재 윈도우의 시작과 끝나는 시간값이다.
여기에 대해 `TimestampWindowStore` 는 해당 윈도우가 마지막에 업데이트된 `Timestamp` 도 관리하기 때문에 확인도 가능하다.  
`TimestampWindowStore` 는 `Persistent` 저장소에서만 사용 가능하다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-7.drawio.png)


기본적인 동적과 처리 결과는 `WindowStore` 와 동일하다.
차이점은 `TimestampKeyValueStore` 와 동일하게 각 윈도우 별로 가장 최근에 업데이트된 타임스템프가 함께 저장된다는 점이다.


아래는 `persistentTimestampWindowStore` 의 사용 예시이다.

```java
public void persistentTimestampWindowStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .groupBy((voter, item) -> item)
        .windowedBy(TimeWindows.of(Duration.ofMillis(10)))
        .count(Materialized.<String, Long>as(
                Stores.persistentTimestampedWindowStore("persistent-timestamp-window-store-count", Duration.ofMillis(100),
                    Duration.ofMillis(10), false))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .map((stringWindowed, aLong) -> KeyValue.pair(stringWindowed.key(), aLong))
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void persistentTimestampWindowStore() {
	this.persistentStateStore.persistentTimestampWindowStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);

	WindowStore<String, ValueAndTimestamp<Long>> outputStore = this.topologyTestDriver.getTimestampedWindowStore("persistent-timestamp-window-store-count");

	assertThat(outputStore.fetch("a", 0L, 10L).next().value, is(ValueAndTimestamp.make(2L, 5L)));
	assertThat(outputStore.fetch("a", 10L, 20L).next().value, is(ValueAndTimestamp.make(1L, 10L)));
	assertThat(outputStore.fetch("b", 0L, 10L).next().value, is(ValueAndTimestamp.make(2L, 8L)));
	assertThat(outputStore.fetch("c", 0L, 10L).next().value, is(ValueAndTimestamp.make(1L, 3L)));
	assertThat(outputStore.fetch("c", 30L, 40L).next().value, is(ValueAndTimestamp.make(1L, 30L)));

	List<TestRecord<String, Long>> outputStream = this.outputTopic.readRecordsToList();

	assertThat(outputStream, hasSize(7));

	assertThat(outputStream.get(0).key(), is("a"));
	assertThat(outputStream.get(0).value(), is(1L));
	assertThat(outputStream.get(0).timestamp(), is(1L));

	assertThat(outputStream.get(1).key(), is("b"));
	assertThat(outputStream.get(1).value(), is(1L));
	assertThat(outputStream.get(1).timestamp(), is(2L));

	assertThat(outputStream.get(2).key(), is("c"));
	assertThat(outputStream.get(2).value(), is(1L));
	assertThat(outputStream.get(2).timestamp(), is(3L));

	assertThat(outputStream.get(3).key(), is("a"));
	assertThat(outputStream.get(3).value(), is(2L));
	assertThat(outputStream.get(3).timestamp(), is(5L));

	assertThat(outputStream.get(4).key(), is("a"));
	assertThat(outputStream.get(4).value(), is(1L));
	assertThat(outputStream.get(4).timestamp(), is(10L));

	assertThat(outputStream.get(5).key(), is("b"));
	assertThat(outputStream.get(5).value(), is(2L));
	assertThat(outputStream.get(5).timestamp(), is(8L));

	assertThat(outputStream.get(6).key(), is("c"));
	assertThat(outputStream.get(6).value(), is(1L));
	assertThat(outputStream.get(6).timestamp(), is(30L));
}
```  

#### VersionedKeyValueStore
버전관리가 가능한 `KeyValueStore` 이다.
동일한 키에 대해 여러 버전의 값을 저장할 수 있다.
각 버전은 특정 타임스탬프와 함께 관리되고,
과거 데이터를 조회하거나 특정 시점의 데이터를 복원할 수 있다.
`VersionedKeyValueStore` 관련 상세한 내용은 [여기]()
에서 확인 가능하다.
그리고 `VersionedKeyValueStore` 는 `Persistent` 저장소에만 사용 가능하다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-type-5.drawio.png)


위 그림을 보면 대부분 처리는 `KeyValueStore` 와 유사하다.
하지만 `voter5` 에 대한 투표 결과를 보면 `KeyValueStore` 와 다른 것을 볼 수 있다.
이는 `VersionedKeyValueStore` 의 경우 집계 연산인 `count()` 를 수행할 때,
`t=10` 에 `a` 에 투표한 결과만 반영된다.
순서가 바껴 늦게 도착한 `t=8` 에 `b` 에 투표한 메시지는 시간 상으로는 더 과거이므로,
`voter5` 의 최신 투표는 `a` 이기 때문에 `b` 에 투표한 것은 무시되는 것이다.

아래는 `persistentVersionedKeyValueStore` 의 사용 예시이다.

```java
public void persistentVersionedKeyValueStore(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopicStream = streamsBuilder.stream("input-topic");

    // count
    inputTopicStream
        .toTable(
            Materialized.<String, String>as(Stores.persistentVersionedKeyValueStore("persistent-versioned-key-value-store", Duration.ofMillis(10)))
                .withKeySerde(Serdes.String())
                .withValueSerde(Serdes.String()))
        .groupBy((voter, item) -> KeyValue.pair(item, item))
        .count(Materialized.<String, Long>as(
                Stores.persistentVersionedKeyValueStore("persistent-versioned-key-value-store-count", Duration.ofMillis(10)))
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long()))
        .toStream()
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void persistentVersionedKeyValueStore() throws InterruptedException {
	this.persistentStateStore.persistentVersionedKeyValueStore(this.streamsBuilder);
	this.startStream();

	this.inputTopic.pipeInput("voter1", "a", 1L);
	this.inputTopic.pipeInput("voter2", "b", 2L);
	this.inputTopic.pipeInput("voter3", "c", 3L);

	this.inputTopic.pipeInput("voter4", "a", 5L);

	this.inputTopic.pipeInput("voter5", "a", 10L);
	this.inputTopic.pipeInput("voter5", "b", 8L);
	this.inputTopic.pipeInput("voter6", "c", 30L);


	VersionedKeyValueStore<String, Long> outputStore = this.topologyTestDriver.getVersionedKeyValueStore("persistent-versioned-key-value-store-count");

	assertThat(outputStore.get("a"), is(new VersionedRecord<>(3L, 10L)));
	assertThat(outputStore.get("b"), is(new VersionedRecord<>(1L, 2L)));
	assertThat(outputStore.get("c"), is(new VersionedRecord<>(2L, 30L)));

	List<KeyValue<String, Long>> outputStream = this.outputTopic.readKeyValuesToList();

	assertThat(outputStream, hasSize(6));

	assertThat(outputStream.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputStream.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputStream.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputStream.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputStream.get(5), is(KeyValue.pair("c", 2L)));
}
```  




---  
## Reference
[Kafka Streams State Stores](https://kafka.apache.org/38/javadoc/org/apache/kafka/streams/state/Stores.html)  


