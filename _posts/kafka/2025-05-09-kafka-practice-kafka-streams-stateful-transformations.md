--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Stateful Transformations"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 Stateful Transformations 의 종류와 사용법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Stateful Transformations
    - Aggregation
    - Joining
    - Windowing
    - Custom Processors
    - State Store
    - Record cache
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

이후 소개하는 예제의 전체 코드 내용과 결과에 대한 테스트 코드는 [여기](https://github.com/windowforsun?tab=repositories)
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


#### reduce
`reduce()` 는 `Rolling Aggregation` 동작으로 그룹화된 키에 따라 레코드의 값을 결합하는 연산이다. 
현재 레코드 값은 마지막으로 결합된 값과 결합되어 새로운 값이 반환된다. 
결과의 반환값 유형은 `aggregate()` 와 달리 변경이 불가해 입력 유형과 결과 유형이 같아야 한다. 
`KGroupedStream` 과 `KGroupedTable` 에서 사용 가능하다. 

`KGroupedStream` 을 사용할 때는 `reducer(adder)` 를 제공해야 한다. 
그리고 `KGroupedTable` 을 사용할 때는 더하기 연산인 `reducer(adder)` 와 빼기 연산 `reducer(subtractor)` 를 제공해야 한다.  

`KGroupedStream` 에서 `reduce()` 를 수행할 때 주요 특징은 아래와 같다.  

- `null` 키가 있는 레코드는 무시된다. 
- 처음 수신한 레코드의 키는 해당 레코드의 값이 초기 집계값으로 사용한다. 
- `null` 이 아닌 값을 가진 레코드를 수신하면 `reducer(adder)` 가 호출된다. 

`KGroupedTable` 에서 `reduce()` 를 수행할 때 주요 특징은 아래와 같다.  

- `null` 키가 있는 레코드는 무시된다. 
- 처음 수신한 레코드의 키는 해당 레코드의 값이 초기 집계값으로 사용된다. 해당 동작은 `tombstone record` 수신 후에도 동일하게 발생한다. 
- 키에 대해 `null` 이 아닌 값을 가진 레코드를 수신하면 `reducer(adder)` 만 호출된다. 
- 키에 대해 다음 `null` 이 아닌 값을 수신하면, 테이블에 저장된 이전 값으로 `reducer(subtractor)` 가 호출되고, 새로운 값으로 `reducer(adder)` 가 호출된다. 
- 키에 대해 `null` 인 레코드를 받으면 `tombstone record` 이므로 `reducer(subtractor)` 만 호출된다. 빼기 연산에서 `null` 을 반환하면 해당 키는 `KTable` 에서 제거되고 이후 다시 동일한 키가 수신되면 초기화 과정을 거치게된다.  


```
KGroupedStream -> KTable
KGroupedTable -> KTable
```  

예제는 `KGroupedStream` 을 사용해 수신된 레코드의 값을 키가 동일한 것들과 결합하는 스트림이다.  

```java
public void reduceKGroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic",
        Consumed.with(Serdes.String(), Serdes.String()));

    KGroupedStream<String, String> kGroupedStream = inputStream.groupBy((key, value) -> value);

    KTable<String, String> kTable = kGroupedStream.reduce((aggValue, newValue) -> aggValue + newValue);

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  

다음 예제는 `KGroupedStream` 을 사용해 수신된 레코드의 값을 키가 동일한 것들과 결합하는 스트림이다.

```java
public void reduceKGroupedTable(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputTopic.toTable();

    KGroupedTable<String, String> kGroupedTable = inputTable.groupBy((key, value) -> KeyValue.pair(value, value));

    KTable<String, String> kTable = kGroupedTable.reduce((aggValue, newValue) -> aggValue + newValue, (aggValue, newValue) -> aggValue.replaceFirst(newValue, ""));

    kTable.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  


#### reduce(windowed)
`reduce(windowed)` 는 `Windowed Aggregation` 동작으로 그룹화된 키에 따라 각 윈도우별로 레코드의 값을 결합한다. 
대부분 특징은 앞선 `reduce()` 외 비슷하고, `null` 인 키 또는 값을 가진 레코드가 수신되면 무시된다는 특징이 있다. 
`TimeWindowedKStream` 과 `SessionWindowedKStream` 에서 사용할 수 있다.  

- `reduce()` 와 비슷하지만 결합 연산이 윈도우 별로 적용된다. 
- `null` 키인 레코드는 무시된다. 
- 특정 윈도우에서 처음 수신한 키는 해당 레코드의 값이 초기 집계 값으로 사용된다. 
- 특정 윈도우에서 `null` 값이 아닌 레코드를 받으면 `reducer(adder)` 가 호출된다.  

```
KGroupedStream -> TimeWindowedKStream -> KTable
KGroupedStream -> SessionWindowedKStream -> KTable
```  

예제는 `TimeWindowedKStream` 을 사용해 수신된 레코드의 값을 키가 동일한 것들과 윈도우별로 결합하는 스트림이다.

```java
public void reduceTimeWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    TimeWindowedKStream<String, String> timeWindowedKStream = kGroupedStream.windowedBy(TimeWindows.ofSizeWithNoGrace(
        Duration.ofMillis(10)));

    KTable<Windowed<String>, String> kTable = timeWindowedKStream.reduce((aggValue, newValue) -> aggValue + newValue);

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  

다음 예제는 `SessionWindowedKStream` 을 사용해 수신된 레코드의 값을 키가 동일한 것들과 윈도우별로 결합하는 스트림이다.

```java
public void reduceSessionWindowedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic.groupBy((key, value) -> value);

    SessionWindowedKStream<String, String> sessionWindowedKStream = kGroupedStream.windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMillis(10)));

    KTable<Windowed<String>, String> kTable = sessionWindowedKStream.reduce((aggValue, newValue) -> aggValue + newValue);

    kTable
        .toStream()
        .map((key, value) -> KeyValue.pair(key.key(), value))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  

#### Impact of record caches
`Kafka Streams DSL` 에서 `record cache` 는 상태 저장소에서 사용되는 일시적인 메모리 공간을 의미한다. 
이는 성능은 향상시킬 수 있지만 테이블 변경사항을 바로 스트림을 통해 전달 받을 수 없는 특징이 있어 이부분을 집고 넘어가고자 한다. 
하지만 그렇다고 해서 기본으로 활성화돼 있는 `record cache` 를 비활성화하는 것은 위험할 수 있다. 
테스트 목적이 아닌 이상 `record cache` 는 활성화를 권장한다. 

`KGroupedStream -> KTable` 로 변환하는 경우에서 `record cache` 가 어떤 영향을 미치는지 알아본다. 
테스트를 위해 입력으로 들어오는 동일한 키의 레코드를 카운트하는 스트림을 사용한다.  

```java
public void cacheImpactKGroupedStream(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.<String, String>stream("input-topic");

    KGroupedStream<String, String> kGroupedStream = inputTopic
        .groupByKey(Grouped.with(Serdes.String(), Serdes.String()));

    KTable<String, String> ktable = kGroupedStream.aggregate(
        () -> "0",
        (key, value, agg) -> String.valueOf(Integer.parseInt(agg) + Integer.parseInt(value)),
        Materialized.<String, String, KeyValueStore<Bytes, byte[]>>as("kGroupedStream-cache")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.String())
    );

    ktable.toStream()
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  

위 스트림 처리가 수행되는 과정을 표현하면 아래와 같다.  

| Timestamp | Input record<br>(KStream) | Grouping<br>(KStream) | Initializer<br>(KGroupedStream) | Adder<br>(KGroupedStream) | State<br>(KTable)                  |
|-----------|---------------------------|-----------------------|---------------------------------|---------------------------|------------------------------------|
| 1         | (hello, 1)                | (hello, 1)            | 0 (for hello)                   | (hello, 0 + 1)            | (hello, 1)                         |
| 2         | (kafka, 1)                | (kafka, 1)            | 0 (for kafka)                   | (kafka, 0 + 1)            | (hello, 1) (kafka, 1)              |
| 3         | (streams, 1)              | (streams, 1)          | 0 (for streams)                 | (streams, 0 + 1)          | (hello, 1) (kafka, 1) (streams, 1) |
| 4         | (kafka, 1)                | (kafka, 1)            |                                 | (kafka, 1 + 1)            | (hello, 1) (kafka, 2) (streams, 1) |
| 5         | (kafka, 1)                | (kafka, 1)            |                                 | (kafka, 2 + 1)            | (hello, 1) (kafka, 3) (streams, 1) |
| 6         | (streams, 1)              | (streams, 1)          |                                 | (streams, 1 + 1)          | (hello, 1) (kafka, 3) (streams, 2) |

아래 테스트 코드와 같이 강제로 캐시를 비활성화 한 경우(`statestore.cache.max.bytes=0, commit.interval.ms=0`)는 업데이트 되는 모든 결과가 결과 스트림으로 전달되는 것을 확인 할 수 있다.  

```java
@Test
public void no_cacheImpactKGroupedStream() throws InterruptedException {
    AggregatingTransforms aggregatingTransforms = new AggregatingTransforms();

    aggregatingTransforms.cacheImpactKGroupedStream(this.streamsBuilder);

    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test22");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-state");
    props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
    props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 0);

    Topology topology = this.streamsBuilder.build();
    KafkaStreams kafkaStreams1 = new KafkaStreams(topology, props);
    kafkaStreams1.start();

    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams1.state() == KafkaStreams.State.RUNNING);


    this.kafkatemplate.send("input-topic",0, 100L, "hello", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 200L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 300L, "stream", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 400L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 500L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 600L, "stream", "1");
    Thread.sleep(3000);

    assertThat(TestConsumerConfig.LISTEN_LIST, hasSize(6));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).key(), is("hello"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).key(), is("kafka"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).key(), is("stream"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).key(), is("kafka"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).value(), is("2"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).key(), is("kafka"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).key(), is("stream"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).value(), is("2"));
}
```  

하지만 `record cache` 를 활성화하면 동일한 키가 연속으로 사용되는 4, 5번의 타임스템프의 결과가 압축되는 것을 볼 수 있다.  

```java
@Test
public void cacheImpactKGroupedStream() throws InterruptedException {
    AggregatingTransforms aggregatingTransforms = new AggregatingTransforms();

    aggregatingTransforms.cacheImpactKGroupedStream(this.streamsBuilder);

    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test22");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-state");
    props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 100);
    props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100);

    Topology topology = this.streamsBuilder.build();
    KafkaStreams kafkaStreams1 = new KafkaStreams(topology, props);
    kafkaStreams1.start();

    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams1.state() == KafkaStreams.State.RUNNING);

    this.kafkatemplate.send("input-topic",0, 100L, "hello", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 200L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 300L, "stream", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 400L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 500L, "kafka", "1");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 600L, "stream", "1");
    Thread.sleep(3000);

    assertThat(TestConsumerConfig.LISTEN_LIST, hasSize(5));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).key(), is("hello"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).key(), is("stream"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).key(), is("kafka"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).value(), is("1"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).key(), is("kafka"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).key(), is("stream"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).value(), is("2"));
}
```  

다음은 `KGroupedTable -> KTable` 로 변환하는 경우에서 `record cache` 가 어떤 영향을 미치는지 알아본다.
테스트를 위해 입력으로 들어오는 레코드 키의 문자열 길이를 값을 기준으로 그룹화해 카운트하는 스트림을 사용한다.  

```java
public void cacheImpactKGroupedTable(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputTopic = streamsBuilder.<String, String>stream("input-topic");
    KTable<String, String> inputTable = inputTopic.<String, String>toTable();

    KGroupedTable<String, Integer> kGroupedTable = inputTable
        .groupBy((s, s2) -> KeyValue.pair(s2, s.length()), Grouped.with(Serdes.String(), Serdes.Integer()));

    KTable<String, Integer> kTable = kGroupedTable.aggregate(
        () -> 0,
        (key, newValue, aggValue) -> aggValue + newValue,
        (key, newValue, aggValue) -> aggValue - newValue,
        Materialized.<String, Integer, KeyValueStore<Bytes, byte[]>>as("kGroupedTable-cache")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Integer())
    );

    kTable.toStream().mapValues((s, integer) -> String.valueOf(integer))
        .to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}
```  

위 스트림 처리가 수행되는 과정을 표현하면 아래와 같다.

| Timestamp | Input record<br>(KTable) | Interpreted as<br>(KTable) | Grouping<br>(KTable) | Initializer<br>(KGroupedTable) | Adder<br>(KGroupedTable)  | Subtractor<br>(KGroupedTable)  | State<br>(KTable) |
|-----------|--------------------------|----------------|----------|--------------------------------|---------------------------|-------------|-------------------|
| 1         | (alice, E)               | INSERT alice   | (E, 5)   | 0 (for E)                      | (E, 0 + 5)                |             | (E, 5)            |
| 2         | (bob, A)                 | INSERT bob     | (A, 3)   | 0 (for A)                      | (A, 0 + 3)                |             | (A, 3) (E, 5)     |
| 3         | (charlie, A)             | INSERT charlie | (A, 7)   |                                | (A, 3 + 7)                |             | (A, 10) (E, 5)    |
| 4         | (alice, A)               | UPDATE alice   | (A, 5)   |                                | (A, 10 + 5)               | (E, 5 - 5)  | (A, 15) (E, 0)    |
| 5         | (charlie, null)          | DELETE charlie | (null, 7)|                                |                           | (A, 15 - 7) | (A, 8) (E, 0)     |
| 6         | (null, E)                | ignored        |          |                                |                           |             | (A, 8) (E, 0)     |
| 7         | (bob, E)                 | UPDATE bob     | (E, 3)   |                                | (E, 0 + 3)                | (A, 8 - 3)  | (A, 5) (E, 3)     |


아래 테스트 코드와 같이 강제로 캐시를 비활성화 한 경우(`statestore.cache.max.bytes=0, commit.interval.ms=0`)는 업데이트 되는 모든 `KTable` 변경이 결과 스트림으로 전달되는 것을 확인 할 수 있다.

```java
@Test
public void no_cacheImpactKGroupedTable() throws InterruptedException {
    AggregatingTransforms aggregatingTransforms = new AggregatingTransforms();

    aggregatingTransforms.cacheImpactKGroupedTable(this.streamsBuilder);

    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test22");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-state");
    props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
    props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 0);

    Topology topology = this.streamsBuilder.build();
    KafkaStreams kafkaStreams1 = new KafkaStreams(topology, props);
    kafkaStreams1.start();

    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams1.state() == KafkaStreams.State.RUNNING);

    this.kafkatemplate.send("input-topic",0, 100L, "alice", "E");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 200L, "bob", "A");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 300L, "charlie", "A");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 400L, "alice", "A");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 500L, "charlie", null);
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 600L, null, "E");
    Thread.sleep(50);
    this.kafkatemplate.send("input-topic", 0, 700L, "bob", "E");
    Thread.sleep(3000);

    TestConsumerConfig.LISTEN_LIST.forEach(System.out::println);

    assertThat(TestConsumerConfig.LISTEN_LIST, hasSize(8));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).value(), is("5"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).value(), is("10"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).value(), is("0"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).value(), is("15"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).value(), is("8"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(6).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(6).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(7).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(7).value(), is("5"));
}
```  

하지만 `record cache` 를 활성화하면 동일한 키가 연속으로 사용되는 4, 5번의 타임스템프의 결과가 압축되는 것을 볼 수 있다. 

```java
@Test
public void cacheImpactKGroupedTable() throws InterruptedException, ExecutionException {
    AggregatingTransforms aggregatingTransforms = new AggregatingTransforms();

    aggregatingTransforms.cacheImpactKGroupedTable(this.streamsBuilder);

    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test22");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-state");
    props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 200);
    props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100);

    Topology topology = this.streamsBuilder.build();
    KafkaStreams kafkaStreams1 = new KafkaStreams(topology, props);
    kafkaStreams1.start();

    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams1.state() == KafkaStreams.State.RUNNING);

    this.kafkatemplate.send("input-topic",0, 100L, "alice", "E").get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 200L, "bob", "A").get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 300L, "charlie", "A").get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 400L, "alice", "A").get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 500L, "charlie", null).get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 600L, null, "E").get();
    Thread.sleep(40);
    this.kafkatemplate.send("input-topic", 0, 700L, "bob", "E").get();
    Thread.sleep(3000);

    TestConsumerConfig.LISTEN_LIST.forEach(System.out::println);

    assertThat(TestConsumerConfig.LISTEN_LIST, hasSize(7));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(0).value(), is("5"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(1).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(2).value(), is("10"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(3).value(), is("0"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(4).value(), is("8"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).key(), is("E"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(5).value(), is("3"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(6).key(), is("A"));
    assertThat(TestConsumerConfig.LISTEN_LIST.get(6).value(), is("5"));
}
```  


### Joining
`Joining` 은 서로 다른 스트림 또는 테이블의 데이터를 결합하거나 하나의 출력 데이터를 만드는 연산이다. 
스트림간 또한 테이블간의 관계를 정의하고, 이를 통해 데이터의 상관관계를 파악하는데 사용될 수 있다. 
조인을 사용하면 원격 데이터베이스에 접근하는 대신, `Kafka Streams` 의 로컬 상태 저장소에서 저장된 테이블을 이용해 
매우 빠른 조인 연산을 수행할 수 있다. 즉 `CDC` 를 통해 데이터베이스의 내용을 `Kafka` 로 옮기는 등 해서 
스트림 처리과정에서 원격 호출을 최소화해 부하를 줄이고 성능은 높일 수 있다.  

조인을 수행할 때도 `Aggregating` 과 마찬가지로 `Windowed` 와 `Non-Windowed` 기반 조인이 있다. 
이는 조인하려는 두 소스가 어떤 유형이냐에 따라 달리질 수 있다. 
조인은 다양한 유형의 조인을 지원하는데 이를 표로 정리하면 아래와 같다.  

Join operands|Type|(Inner)JOIN|LEFT JOIN|OUTER JOIN
---|---|---|---|---
KStream-KStream|Windowed|O|O|O
KTable-KTable|Non-Windowed|O|O|O
KTable-KTable FK|Non-Windowed|O|O|X
KStream-KTable|Non-Windowed|O|O|X
KStream-GlobalKTable|Non-Windowed|O|O|X
KTable-GlobalKTable|-|X|X|X

`Kafka Streams` 간 `Joining` 을 수행할 떄, 
`Co-partitioning` 이라는 특징이 있다. 
`Co-partitioning` 은 두 개 이상의 스트림이나 테이블이 조인될 떄, 
동일한 키를 가진 레코드들이 동일한 파티션에 존재해야 한다는 요구사항을 의미한다. 
이는 `Kafka Streams` 가 분산 환경에서 데이터를 효율적으로 처리하고 조인 작업을 수행하기 위핸 핵심 매커니즘이다.  
`KTable-KTable FK` 조인과 `GlobalKTable` 과의 조인 외에 모든 조인 연산에서는 `Co-partitioning` 이 요구된다.  

`Kafka Streams` 에서 조인은 레코드의 키를 기반으로 수행되는데, 
`leftRecord.key == rightRecord.key` 와 같이 조인 조건을 설정하기 때문에 
스트림/테이블이 동일한 키로 파티셔닝돼야 한다. 
`Kafka Streams` 에서 조인 연산시 소스 스트림/테이블의 파티션 수가 같은지 확인하지만, 
파티셔닝 전략이 동일한지는 확인하지 않기 때문에 이는 개발자가 보장해야 한다. 

`Co-partitioning` 의 요구 사항은 아래와 같다. 

1. 조인에 사용되는 두 입력 토픽(좌측, 우측)은 동일한 파티션 수를 가져야 한다. 
2. 두 입력 토픽은 동일한 파티셔닝 전략을 시용해야 한다. 이를 통해 동일한 키를 가진 레코드가 동일한 파티션 번호로 전달되도록 해야 한다. 

`Kafka Client` 를 사용하는 `Kafka Producer API` 를 사용하는 애플리케이션은 동일한  `partitioner.class`(`ProducerConfig.PARTITIONER_CLASS_CONFIG`) 를 사용해야 하고, 
`Kafka Streams API` 를 사용하는 애플리케이션은 `KStream.to()` 를 사용할 때 동일한 `StreamPartitioner` 를 사용해야 한다. 
만약 별도로 설정을 해주지 않고 모든 애플리케이션이 기본 파티셔너를 사용한다면 파티셔닝 전략에 대해서는 고려할 필요 없다.  

`Co-partitioning` 이 돼있지 않다면 이는 수동으로 데이터를 `repartitioning` 해야 한다. 
이는 일반적으로 병목현상을 방지하기 위해 아래와 같은 조건을 기준으로 하는게 좋다. 

- 파티션 수가 적은 토픽을 더 큰 파티션 수로 `repartitioning` 한다. (반대로 하는 것도 가능하다.)
- `KStream-KTable` 조인의 경우 `KStream` 의 소스 토픽을 `repartitioning` 하는 것이 좋다. (`KTable` 을 할 경우 `State Store` 가 다시 생성될 수 있다.)
- `KTable-KTable` 조인의 경우 각 `KTable` 의 크기를 고려해 더 작은 `KTable` 을 `repartitioning` 하는 것이 좋다.  

`Co-partitioning` 을 위해 수동으로 `repartitioning` 을 진행한다면 아래와 같은 절차를 따를 수 있다.  

1. 조인에서 사용하는 두 입력 토픽 중 토픽의 파티션 수가 더 적은 쪽을 확인한다. (`bin/kafka-topics --describe` 를 통해 확인 가능)
2. 애플리케이션 내에서 파티션이 적은 토픽을 사용하는 소스를 대상으로 `repartitioning` 을 수행한다. 수행할 때는 파티션이 더 큰 쪽의 수와 동일하게 맞춘다. 
  - `KStream` 인 경우 `KStream.repartition(Repartitioned.numberOfPartitions(..))` 을 사용한다. 
  - `KTable` 인 경우 `KTable.toStream.repartition(Repartitioned.numberOfPartitions(..).toTable())` 을 사용한다. 
3. 스트림 애플리케이션 내에서 두 입력간 조인을 수행한다. 

`Kafka Streams` 의 `Joining` 에 대한 상세한 내용은 [여기]({{site.baseurl}}{% link _posts/kafka/2024-12-02-kafka-practice-kafka-streams-join.md %})
에서 확인 할 수 있다.  


### Windowing
`Windowing` 은 시간 기반으로 데이터를 그룹화하여 처리하는 것을 의미한다. 
스트림 데이터는 지속적으로 흘러들어오기 때문에, 특정 시간 간격으로 데이터를 구분해 처리할 필요가 있다. 
윈도우 연산은 스트림의 무한한 데이터를 시간 창(`window`)으로 나누어 집계하거나 조인할 수 있다.  

`Joining` 작업을 하면 정의된 윈도우 경계 내에서 레코드를 저장하기 위해 `Windowing State Store` 를 사용 한다. 
그리고 해당 저장소를 사용해 `Aggregating` 동작에서 최신 윈도우에 대한 집계를 수행하게 된다.  

`Windowing State Store` 에 있는 오래된 레코드는 `window retention period` 이후에는 제거된다. 
`Kafka Streams` 는 최소 설정된 시간 동안 윈도우 데이터를 유지하는데, 해당 기본 값은 1일이다. 
이 설정은 `Materialized.withRetention()` 을 통해 변경 가능하다.  

관련 연산으로는 `Grouping` 이 있다. 
`Grouping` 을 수행하면 동일한 키를 가진 모든 레코드를 그룹화해 후속 작업을 위해 데이터가 적절하게 `partitioning` 되었는지 확인할 수 있다. 
이후 그룹화된 것을 바탕으로 `Windowing` 을 수행하면 각 키별 레코드를 윈도우 별로 `sub-group` 할 수 있다.  

아래는 `Streams DSL` 을 사용했을 때 구현할 수 있는 윈도우의 종류이다.  


Window name|Behavior|Desc
---|---|---
Tumbling Window|Time-based|고정 크기, 윈도우간 겹치지 않음, 윈도우간 공백 시간 없음
Hopping Window|Time-based|고정 크기, 윈도우간 겹침
Sliding time Window|Time-based|고정 크기, 레코드 타임스탬프를 기준으로 겹치거나 겹치지 않음
Session Window|Session-based|동적 크기, 윈도우간 겹치지 않음, 레코드 기반 윈도우


`Kafka Streams` 의 `Windowing` 에 대한 상세한 내용은 [여기]({{site.baseurl}}{% link _posts/kafka/2024-06-20-kafka-practice-kafka-streams-windowing.md %})
에서 확인 할 수 있다.



---  
## Reference
[Stateful transformations](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#stateful-transformations)  
[co-partition](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#streams-developer-guide-dsl-joins-co-partitioning)  
[record cache](https://docs.confluent.io/platform/current/streams/developer-guide/memory-mgmt.html#streams-developer-guide-memory-management-record-cache)  
[Fault tolerance](https://docs.confluent.io/platform/current/streams/architecture.html#streams-architecture-fault-tolerance)  



