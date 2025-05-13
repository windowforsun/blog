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

### Aggregating
`Aggregating` 은 레코드의 키를 기준으로 그룹화된 데이터에 대한 연산을 수행하여 결과를 출적하는 과정이다. 
`Kafka Streams` 에서는 `groupByKey()` ,`groupBy()` 를 통해 데이터를 `KGroupedStreams` 혹은 
`KGroupedTable` 로 변환한 후 집계 연산을 적용할 수 있다. 
그리고 이러한 집계 연산은 경우에 따라 `Windowed`(윈도우 기반), `Non-Windowed`(비윈도우 기반)에 대해 수행될 수 있다.  

레코드를 키를 기준으로 집계해 결과를 만들어내는 연산인 만큼 `초기화 함수` 와 `집계 함수` 를 별도로 구현해야 한다. 
여기서 `fault-tolerance` 를 지원하고 예기치 않는 결과를 방지하기 위해서는 초기화 함수와 집계 함수는 상태를 가지지 않아야 한다. 
이는 초기화 함수와 집계 함수가 외부 상태값에 대한 의존성 없이 연산의 반환 값으로 전달되야 한다는 것을 의미한다. 
예를 들어 이들이 외부에 정의된 클래스 멤버 변수를 사용해 반환 값이 결정된다면 장애 발생시 데이터가 손실될 수 있다.  


#### aggregate
`aggregate()` 는 `Rolling Aggregation` 을 의미한다. 
이는 그룹화된 키에 따라(윈도우 X) 레코드의 값을 집계하는 방식이다. 
`Aggregate` 는 `Reduce` 의 좀 더 일반화된 형태로, 입력 값과 다른 타입의 결과를 가질 수 있다. 
`aggregate()` 는 `KGroupedStream`, `CogroupedKStream`, `KGroupedTable` 을 통해 사용할 수 있다.  

먼저 `KGroupedStream` 은 그룹화된 스트림을 집계할 떄 사용 가능한데 초기 값을 제공해야하고, 
`aggregator` 라는 집계 함수를 정의해야 한다. 
그리고 `CogroupedKStream` 은 다수의 그룹화된 스크림을 집계 할때 사용 가능하고, 
초기 값만 제공하면 된다. 이는 `CogroupedKStream` 을 구성하는 시점에 이미 `cogroup()` 을 통해 `aggregator` 가 제공됐기 때문이다. 
마지막으로 `KGroupedTable` 은 그룹화된 테이블을 집계할 떄 사용 가능하고, 
초기 값과 집계의 더하는 연산 `aggregator` 와 집계의 빼는 연산 `aggregator(adder)` 와 같이 2개의 `aggregator(subtractor)` 함수를 정의해야 한다.  

그리고 `KGroupedStream` 과 `CogroupedKStream` 에서 `aggregate()` 의 주요 특징은 아래와 같다. 

- `null` 키가 있는 레코드는 무시된다. 
- 키가 처음 수신되면 정의한 초기값이 호출 된다. 
- `null` 이 아닌 값의 레코드가 수신되면 `aggregator` 가 호출된다. 

`KGroupedTable` 에서 `aggregator()` 의 주요 특징은 아래와 같다. 

- `null` 키가 있는 레코드는 무시된다. 
- 키가 처음 수신되면 정의한 초기값이 호출 된다. `KGroupedTable` 는 `tombstone record` 처리를 위해 초기값이 여러번 호출 될 수 있다.  
- 처음 `null` 이 아닌 값이 수신되면, `adder` 만 호출된다. 
- 이후 `null` 이 아닌 값이 수신되면, 먼저 기존 값에 `subtractor` 가 호출되고 그 다음 새로 수신된 값에 `adder` 가 호출된다. 
- `null` 값을 가진 레코드(`tombstone record`)를 수신하면 `subtractor` 만 호출된다. `subtractor` 가 `null` 을 반환하면 해당 키는 `KTable` 에서 제거되고, 다음에 해당 키가 다시 수신되면 초기값을 다시 호출한다. 

```
KGroupedStream -> KTable
CogroupedKStream -> KTable
KGroupedTable -> KTable
```  

예제는 `KGroupedStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 카운트하는 스트림이다. 


```java
public void aggregatingKGroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic",
        Consumed.with(Serdes.String(), Serdes.String()));

    KGroupedStream<String, String> kGroupedStream = inputStream.groupBy((key, value) -> value);

    KTable<String, Long> kTable = kGroupedStream
        .aggregate(
            () -> 0L,
            (key, value, agg) -> agg + 1L,
            Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("aggregating-kgroupedstream-store")
                .withKeySerde(Serdes.String())
                .withValueSerde(Serdes.Long())
        );

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

다음은 예제는 `CogroupedKStream` 을 사용해 여러 토픽에서 수신된 레코드의 값과 동일한 레코드의 수를 카운트하는 스트림이다.

```java
public void aggregatingCoGroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> input1Topic = streamsBuilder.stream("input1-topic");
    KStream<String, String> input2Topic = streamsBuilder.stream("input2-topic");

    KGroupedStream<String, String> input1GroupedStream = input1Topic.groupBy((key, value) -> value,
        Grouped.with("group1", Serdes.String(), Serdes.String()));
    KGroupedStream<String, String> input2GroupedStream = input2Topic.groupBy((key, value) -> value,
        Grouped.with("group2", Serdes.String(), Serdes.String()));

    CogroupedKStream<String, Long> cogroupedKStream = input1GroupedStream.<Long>cogroup((key, value, agg) -> agg + 1L)
        .cogroup(input2GroupedStream, (key, value, agg) -> agg + 1L);

    KTable<String, Long> ktable = cogroupedKStream.aggregate(
        () -> 0L,
        Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("aggregating-cogroupstream-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    ktable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  


다음은 예제는 `KGroupedTable` 을 사용해 여러 토픽에서 수신된 레코드의 값과 동일한 레코드의 수를 카운트하는 스트림이다. 

```java
public void aggregatingKGroupedTable(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputTopic.toTable();

    KGroupedTable<String, String> kGroupedTable = inputTable.groupBy((key, value) -> KeyValue.pair(value, value));

    KTable<String, Long> kTable = kGroupedTable.aggregate(
        () -> 0L,
        (key, value, agg) -> agg + 1L,
        (key, value, agg) -> agg - 1L,
        Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("aggregating-kgroupedtable-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  


#### aggregate(windowed)
`aggregate(windowed)` 는 `Windowed Aggregation` 으로 윈도우 기반 집계 연산을 의미한다. 
각 윈도우별 그룹화된 키를 기준으로 레코드들의 값을 집계하는 방식이다. 
이는 `reduce(windowed)` 의 일반화된 형태로 입력 값과 다른 집계 결과 값을 가질 수 있다.  
`aggregate(windowed)` 는 `TimeWindowedKStream` 과 `SessionWindowedKStream`, `TimeWindowedCogroupedKStream`, `SessionWindowedCogroupedKStream` 을 통해 사용할 수 있다.  

여기서 `TimeWindowedKStream` 과 `SessionWindowedKStream` 은 시간/세션 기반 윈도우가 적용된 스트림을 의미하고, 
`TimeWindowedCogroupedKStream`, `SessionWindowedCogroupedKStream` 은 여러 스트림을 하나로 묶은 후, 시간/세션 기반 윈도우를 적용한 스트림을 의미한다.  

`aggregate(windowed)` 또한 초기값과 `aggregator(adder)` 를 제공해야 한다. 
추가로 윈도우에 대한 정의 또한 필요해 총 3개의 정의가 필요하다.  
`CogorupKStream` 기반인 경우 `aggregator(adder)` 는 이미 `CogroupKStream` 생성시점에 정의가 됐기 때문에, 
초기값과 윈도우에 대한 정의만 제공하면 된다. 
그리고 모든 `Session` 기반 윈도우는 `session merger` 를 추가로 제공해야한다. 
이는 세션은 가변적인 시간의 크기이므로 두 세션을 병합하는 정의가 필요하기 때문이다.  

주요 동작은 아래와 같다.  

- 먼저 살펴본 `rolling aggregation` 과 대부분 유사하지만 윈도우 별로 집계가 적용된다. 
- `null` 키를 가진 입력 레코드는 무시된다. 
- 특정 윈도우 + 키에 대해 처음 레코드가 도착하면 초기화 함수가 호출되고, 그 후부터는 `adder` 가 호출된다. 
- 세션 윈도우의 경우 두 세션이 병합될 때 정의한 `session merger` 가 호출된다.  

```
KGroupedStream -> TimeWindowedKStream -> KTable
KGroupedStream -> SessionWindowedKStream -> KTable
CogroupedKStream -> TimeWindowedCogroupedKStream -> KTable
CogroupedKStream -> SessionWindowedCogroupedKStream -> KTable
```  


예제는 `TimeWindowedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별 카운트하는 스트림이다.

```java
public void aggregatingTimeWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    TimeWindowedKStream<String, String> timeWindowedKStream = kGroupedStream.windowedBy(TimeWindows.ofSizeWithNoGrace(
        Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = timeWindowedKStream.aggregate(
        () -> 0L,
        (key, value, agg) -> agg + 1L,
        Materialized.<String, Long, WindowStore<Bytes, byte[]>>as("aggregating-timewindowedstream-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  


다음 예제는 `SessionWindowedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별 카운트하는 스트림이다.

```java
public void aggregatingSessionWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    SessionWindowedKStream<String, String> sessionWindowedKStream = kGroupedStream.windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = sessionWindowedKStream.aggregate(
        () -> 0L,
        (key, value, agg) -> agg + 1L,
        (key, aggOne, aggTwo) -> aggOne + aggTwo,
        Materialized.<String, Long, SessionStore<Bytes, byte[]>>as("aggregating-sessionwindowedstream-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

다음 예제는 `TimeWindowedCogroupedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별 카운트하는 스트림이다.

```java
public void aggregatingTimeWindowedCogroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> input1Topic = streamsBuilder.stream("input1-topic");
    KStream<String, String> input2Topic = streamsBuilder.stream("input2-topic");

    KGroupedStream<String, String> input1GroupedStream = input1Topic.groupBy((key, value) -> value,
        Grouped.with("group1", Serdes.String(), Serdes.String()));
    KGroupedStream<String, String> input2GroupedStream = input2Topic.groupBy((key, value) -> value,
        Grouped.with("group2", Serdes.String(), Serdes.String()));

    CogroupedKStream<String, Long> cogroupedKStream = input1GroupedStream.<Long>cogroup((key, value, agg) -> agg + 1L)
        .cogroup(input2GroupedStream, (key, value, agg) -> agg + 1L);

    TimeWindowedCogroupedKStream<String, Long> timeWindowedCogroupedKStream = cogroupedKStream.windowedBy(TimeWindows.ofSizeWithNoGrace(
        Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = timeWindowedCogroupedKStream.aggregate(
        () -> 0L,
        Materialized.<String, Long, WindowStore<Bytes, byte[]>>as("aggregating-timewindowedcogroupedstream-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

다음 예제는 `SessionWindowedCogroupedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별 카운트하는 스트림이다.

```java
public void aggregatingSessionWindowedCogroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> input1Topic = streamsBuilder.stream("input1-topic");
    KStream<String, String> input2Topic = streamsBuilder.stream("input2-topic");

    KGroupedStream<String, String> input1GroupedStream = input1Topic.groupBy((key, value) -> value,
        Grouped.with("group1", Serdes.String(), Serdes.String()));
    KGroupedStream<String, String> input2GroupedStream = input2Topic.groupBy((key, value) -> value,
        Grouped.with("group2", Serdes.String(), Serdes.String()));

    CogroupedKStream<String, Long> cogroupedKStream = input1GroupedStream.<Long>cogroup((key, value, agg) -> agg + 1L)
        .cogroup(input2GroupedStream, (key, value, agg) -> agg + 1L);

    SessionWindowedCogroupedKStream<String, Long> sessionWindowedCogroupedKStream = cogroupedKStream.windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = sessionWindowedCogroupedKStream.aggregate(
        () -> 0L,
        (key, aggOne, aggTwo) -> aggOne + aggTwo,
        Materialized.<String, Long, SessionStore<Bytes, byte[]>>as("aggregating-sessionwindowedstream-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

#### count
`count()` 는 `Rolling Aggregation` 으로 `aggregation()` 과 비슷한 특성을 지니며 그룹화된 키에 따라 레코드의 수를 세는 연산이다. 
`KGroupedStream` 과 `KGroupedTable` 에서 사용 가능한데 각 주요 특징은 아래와 같다.  

`KGroupedStream` 의 경우 `null` 키/값이 존재하는 경우 레코드는 무시된다. 
그리고 `KGroupedTable` 은 `null` 키인 레코드는 무시되고, 
값이 `null` 인 경우 무시되진 않지만 해당 키에 대한 삭제로 해석해 키를 삭제한다.  

```
KGroupedStream -> KTable
KGroupedTable -> KTable
```  

예제는 `KGroupedStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 카운트하는 스트림이다.

```java
public void countKGroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic",
        Consumed.with(Serdes.String(), Serdes.String()));

    KGroupedStream<String, String> kGroupedStream = inputStream.groupBy((key, value) -> value);

    KTable<String, Long> kTable = kGroupedStream
        .count();

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

다음 예제는 `KGroupedTable` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 카운트하는 스트림이다.

```java
public void countKGroupedKTable(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputTopic.toTable();

    KGroupedTable<String, String> kGroupedTable = inputTable.groupBy((key, value) -> KeyValue.pair(value, value));

    KTable<String, Long> kTable = kGroupedTable
        .count();

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

#### count(windowed)
`count(windowed)` 는 `Windowed Aggregation` 으로 `aggregation(windowed)` 와 비슷한 특성을 지닌다. 
그룹화된 키에 따라 윈도우 별로 레코드 수를 카운트하는 연산이다. 
이는 `TimeWindowedKStream` 과 `SessionWindowedKStream` 에서 사용 가능하다.  

주요 특징으로는 `null` 키 또는 값을 갖는 레코드는 무시된다.  

```
KGroupedStream -> TimeWindowedKStream -> KTable
KGroupedStream -> SessionWindowedKStream -> KTable
```  

예제는 `TimeWindowedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별로 카운트하는 스트림이다.

```java
public void countTimeWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    TimeWindowedKStream<String, String> timeWindowedKStream = kGroupedStream.windowedBy(TimeWindows.ofSizeWithNoGrace(
        Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = timeWindowedKStream.
        count();

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

예제는 `SessionWindowedKStream` 을 사용해 수신된 레코드의 값과 동일한 레코드의 수를 윈도우별로 카운트하는 스트림이다.

```java
public void countSessionWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    SessionWindowedKStream<String, String> sessionWindowedKStream = kGroupedStream.windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMillis(10)));

    KTable<Windowed<String>, Long> kTable = sessionWindowedKStream
        .count();

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}
```  

