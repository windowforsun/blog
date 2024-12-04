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


### Join Demo
각 소스와 각 조인 연산에 대한 결과 차이를 보이기 위해 아래와 같은 예제 이벤트를 사용한다. 
이는 웹 사이트에서 사용자가 방문하고, 페이지의 특정 영역을 클릭하는 이벤트를 `Kafka Topic` 으로 전송하여 이를 조인하는 예제이다. 
사용자가 페이지에 접근 하는 이벤트를 담는 `View Topic` 과 페이지 접근 후 특정 영역을 클릭하는 `Click Topic` 을 사용한다. 
그리고 두 `Topic` 에 기록되는 메시지의 키는 모두 사용자의 아이디가 동일하게 사용된다.  

예제 이벤트 스트림을 통해 아래 7가지 시나리오를 살펴볼 수 있도록 구성했다. 

- A : 클릭 이벤트가 방문 이벤트 1초 후 도착 
- B : 클릭 이벤트가 방문 이벤트 11초 후 도착 
- C : 방문 이벤트가 클릭 이벤트 1초 후 도착
- D : 방문 이벤트는 있지만 클릭 이벤트는 없음
- E : 클릭 이벤트는 있지만 방문 이벤트는 없음
- F : 2번의 연속 방문 이벤트 1초 후 클릭 이벤트가 있음
- G : 방문 이벤트 1초 후 2개의 클릭 이벤트가 있음

이를 타임시리즈 기반으로 도식화 하면 아래와 같다. 
시간단위는 초를 의미하고 키는 색상으로 구분했다. 

.. 그림 .. 

이후 각 예제에서 사용되는 예제 코드의 전체 내용은 [kafka-streams-join]()
에서 확인 할 수 있다.  

### KStream-KStream Inner Join
`KStream` 간 조인을 하려면 `Window` 를 반드시 적용해야 하는데, 
이를 위해서는 유의미한 값을 도출할 수 있는 `Window` 의 시간 범위를 정의해야 한다. 
`KStream` 은 `Statess` 하므로 조인을 수행하기 위해서는 스트림상에서 상태를 유지할 기준을 마련해야 하기 때문이다. 
그렇지 않으면 무한한 스트림 전체를 스캔해 상태를 구성해야 한다. 
그리고 두 스트림 조인은 일반적으로 가까운 시간내에 있을 때 의미있는 결과를 도출하기 마련이기 때문이다. 
이렇게 `Window` 에 대한 자세한 내용은 [여기]()
에서 확인 할 수 있다. 
이렇게 `Window` 를 정의해서 조인을 수행하면 
두 스트림에서 `Window` 시간내에 포함되는 모든 레코드들을 기반으로 조인이 수행된다.  

아래는 10초 `Window` 를 사용해서 `View` 와 `Click` 스트림을 조인했을 때의 결과 예시이다.  

.. 그림 ..

- `A` 와 `C` 키 이벤트는 발생하고 10초 이내, 다른 스트림에서 `A`, `C` 이벤트가 발생했기 때문에 조인 결과가 출력된다. 
- `B` 키 이벤트가 발생하고 10초 이내 다른 스트림에서 `B` 이벤트가 발생하지 않았으므로 조인 결과는 존재하지 않는다. 
- `View` 에서만 `D` 키 이벤트가 있고, `Click` 에서만 `E` 키 이벤트가 있기 때문에 조인 결과는 존재하지 않는다. 
- `F` 키에 대한 이벤트가 2번 발생하고, 10초 이내 다른 스트림에서 `F` 이벤트가 발생했기 때문에 2개의 조인 결과가 출력된다. 
- `G` 키에 대한 이벤트가 빌생하고, 10초 이내 다른 스트림에서 `G` 이벤트가 2번 발생했기 때문에 2개의 조인 결과가 출력된다. 

이를 코드로 작성하면 아래와 같다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KStream<String, String> clickStream = streamsBuilder.stream(this.clickTopic);
    JoinWindows joinWindows = JoinWindows.of(Duration.ofMillis(this.windowDuration));

    KStream<String, String> joinedStream = viewStream.join(clickStream,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        },
        joinWindows,
        StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
    );

    joinedStream.to(this.resultTopic);
}

@Test
public void viewStream_clickStream_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	assertThat(recordList, hasSize(6));

	assertThat(recordList.get(0).timestamp(), is(1000L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, CLICK:A1"));

	assertThat(recordList.get(1).timestamp(), is(3000L));
	assertThat(recordList.get(1).key(), is("C"));
	assertThat(recordList.get(1).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(2).timestamp(), is(7000L));
	assertThat(recordList.get(2).key(), is("F"));
	assertThat(recordList.get(2).value(), is("VIEW:F1, CLICK:F1"));

	assertThat(recordList.get(3).timestamp(), is(7000L));
	assertThat(recordList.get(3).key(), is("F"));
	assertThat(recordList.get(3).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(4).timestamp(), is(9000L));
	assertThat(recordList.get(4).key(), is("G"));
	assertThat(recordList.get(4).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(5).timestamp(), is(9000L));
	assertThat(recordList.get(5).key(), is("G"));
	assertThat(recordList.get(5).value(), is("VIEW:G1, CLICK:G2"));
}
```  

일반적인 두 스트림의 `Join Window` 는 대칭적인 성격을 띈다. 
이는 다른 스트림과 조인이 될 레코드가 과거, 미래에 모두 존재할 수 있고 모든 경우에도 동일 `Window` 에만 있다면 조인이 발생한다. 
간단한 예로 `View` 이벤트가 발생한 이후 발생한 `Click` 이벤트에 대한 조인만 하고 싶은 경우가 있을 것이다. 
현재 위 예제는 이러한 경우는 대응되지 않았지만 필요한 경우 추가로 구현해서 `(C, C)` 조인 결과를 제거할 수 있는데 그 예시는 아래와 같다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KStream<String, String> clickStream = streamsBuilder.stream(this.clickTopic);
    JoinWindows joinWindows = JoinWindows.of(Duration.ofMillis(this.windowDuration));

    KStream<String, String> joinedStream = clickStream.leftJoin(viewStream,
        (leftClickValue, rightViewValue) -> {
            String result = rightViewValue + ", " + leftClickValue;
            log.info(result);
            return result;
        },
        joinWindows,
        StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
    );

    streamsBuilder.addStateStore(Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("buffer-store"),
        Serdes.String(),
        new ValueAndTimestampSerde<>(Serdes.String()))
    );

    KStream<String, String> filteredStream = joinedStream.transform(() -> new Transformer<String, String, KeyValue<String, String>>() {
        private KeyValueStore<String, ValueAndTimestamp<String>> bufferStore;

        @Override
        public void init(ProcessorContext processorContext) {
            this.bufferStore = (KeyValueStore<String, ValueAndTimestamp<String>>) processorContext.getStateStore("buffer-store");
        }

        @Override
        public KeyValue<String, String> transform(String key, String value) {
            if (value.contains("null")) {
                this.bufferStore.put(key, ValueAndTimestamp.make(value, System.currentTimeMillis()));

                return null;
            } else {
                ValueAndTimestamp<String> bufferedValue = this.bufferStore.get(key);

                if (bufferedValue != null) {
                    bufferStore.delete(key);

                    return null;
                } else {
                    return new KeyValue<>(key, value);
                }

            }
        }

        @Override
        public void close() {

        }
    }, "buffer-store");

    filteredStream.to(this.resultTopic);
}
```  

조인 결과는 토픽 이벤트의 타임스탬프를 기준 순서대로 처리된다는 것을 기억해야 한다.  


### KStream-KStream Left Join
`Left Join` 은 `Left Stream(Primary:View)` 와 `Right Stream(Secondary:Click)` 에 이벤트가 도착할 때마다 조인 연산을 수행하는데 출력 결과에는 약간 차이가 있다. 
우선 `Left Stream` 에 이벤트가 도착하면 다른 스트림에 이전에 이벤트가 도착하면 해당 레코드와 조인하고 없다면 `null` 로 조인하여 무조건 결과가 출력된다. 
그리고 `Right Stream` 에 이벤트가 도착하는 경우에는 `Left Stream` 에 이전에 도착한 이벤트 중 조인 가능한 레코드가 있는 경우에만 결과가 출력된다. 
아래는 실제로 예제 스트림을 `Left Join` 했을 때의 결과이다.  

.. 그림 ..

- `Left Join` 의 결과는 `Inner Join` 의 모든 결과를 포함한다. 
- `View` 에만 존재하는 `D` 키 이벤트에 대한 조인 결과가 출력된다. 
- `View` 에 먼저 도착한 경우인 `A`, `F1`, `F2`, `G` 의 타임스탬프에도 조인 결과가 포함된다. 

아래는 코드로 구현한 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KStream<String, String> clickStream = streamsBuilder.stream(this.clickTopic);
    JoinWindows joinWindows = JoinWindows.of(Duration.ofMillis(this.windowDuration));

    KStream<String, String> joinedStream = viewStream.leftJoin(clickStream,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        },
        joinWindows,
        StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
    );

    joinedStream.to(this.resultTopic);
}

@Test
public void viewStream_clickStream_left_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	assertThat(recordList, hasSize(12));

	assertThat(recordList.get(0).timestamp(), is(0L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, null"));

	assertThat(recordList.get(1).timestamp(), is(1000L));
	assertThat(recordList.get(1).key(), is("B"));
	assertThat(recordList.get(1).value(), is("VIEW:B1, null"));

	assertThat(recordList.get(2).timestamp(), is(1000L));
	assertThat(recordList.get(2).key(), is("A"));
	assertThat(recordList.get(2).value(), is("VIEW:A1, CLICK:A1"));

	assertThat(recordList.get(3).timestamp(), is(3000L));
	assertThat(recordList.get(3).key(), is("C"));
	assertThat(recordList.get(3).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(4).timestamp(), is(4000L));
	assertThat(recordList.get(4).key(), is("D"));
	assertThat(recordList.get(4).value(), is("VIEW:D1, null"));

	assertThat(recordList.get(5).timestamp(), is(6000L));
	assertThat(recordList.get(5).key(), is("F"));
	assertThat(recordList.get(5).value(), is("VIEW:F1, null"));

	assertThat(recordList.get(6).timestamp(), is(6000L));
	assertThat(recordList.get(6).key(), is("F"));
	assertThat(recordList.get(6).value(), is("VIEW:F2, null"));

	assertThat(recordList.get(7).timestamp(), is(7000L));
	assertThat(recordList.get(7).key(), is("F"));
	assertThat(recordList.get(7).value(), is("VIEW:F1, CLICK:F1"));

	assertThat(recordList.get(8).timestamp(), is(7000L));
	assertThat(recordList.get(8).key(), is("F"));
	assertThat(recordList.get(8).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(9).timestamp(), is(8000L));
	assertThat(recordList.get(9).key(), is("G"));
	assertThat(recordList.get(9).value(), is("VIEW:G1, null"));

	assertThat(recordList.get(10).timestamp(), is(9000L));
	assertThat(recordList.get(10).key(), is("G"));
	assertThat(recordList.get(10).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(11).timestamp(), is(9000L));
	assertThat(recordList.get(11).key(), is("G"));
	assertThat(recordList.get(11).value(), is("VIEW:G1, CLICK:G2"));
}
```  

다시 한번 정리하면 `Left Join` 은 `Inner Join` 과 `Left Stream` 에 도착한 모든 레코드를 조인결과로 포함한다고 할 수 있다.  


### KStream-KStream Outer Join
`Ourter Join` 은 `Left Stream(Primary:View)`, `Right Stream(Secondary:Click)` 중 어느 스트림에서든 이벤트가 발생 할 때 조인 결과가 발생한다. 
아래는 `Outer Join` 의 결과이다. 

.. 그림 ..

- `Outer Join` 은 `Left Join` 의 모든 결과를 포함한다.
- `Click` 에만 존재하는 `E` 키 이벤트에 대한 조인 결과가 출력된다. 
- `Click` 에 먼저 도착한 경우인 `C` 키 이벤트의 타임스탬프에도 조인 결과가 포함된다. 
- `Click` 에서 `Join Window` 에 포함되지 못한 `B` 키 이벤트에 대한 타임스탬프에도 조인 결과가 포함된다.  

아래는 코드로 구현한 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KStream<String, String> clickStream = streamsBuilder.stream(this.clickTopic);
    JoinWindows joinWindows = JoinWindows.of(Duration.ofMillis(this.windowDuration));

    KStream<String, String> joinedStream = viewStream.outerJoin(clickStream,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        },
        joinWindows,
        StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
    );

    joinedStream.to(this.resultTopic);
}

@Test
public void viewStream_clickStream_outer_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	assertThat(recordList, hasSize(15));

	assertThat(recordList.get(0).timestamp(), is(0L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, null"));

	assertThat(recordList.get(1).timestamp(), is(1000L));
	assertThat(recordList.get(1).key(), is("B"));
	assertThat(recordList.get(1).value(), is("VIEW:B1, null"));

	assertThat(recordList.get(2).timestamp(), is(1000L));
	assertThat(recordList.get(2).key(), is("A"));
	assertThat(recordList.get(2).value(), is("VIEW:A1, CLICK:A1"));

	assertThat(recordList.get(3).timestamp(), is(2000L));
	assertThat(recordList.get(3).key(), is("C"));
	assertThat(recordList.get(3).value(), is("null, CLICK:C1"));

	assertThat(recordList.get(4).timestamp(), is(3000L));
	assertThat(recordList.get(4).key(), is("C"));
	assertThat(recordList.get(4).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(5).timestamp(), is(4000L));
	assertThat(recordList.get(5).key(), is("D"));
	assertThat(recordList.get(5).value(), is("VIEW:D1, null"));

	assertThat(recordList.get(6).timestamp(), is(5000L));
	assertThat(recordList.get(6).key(), is("E"));
	assertThat(recordList.get(6).value(), is("null, CLICK:E1"));

	assertThat(recordList.get(7).timestamp(), is(6000L));
	assertThat(recordList.get(7).key(), is("F"));
	assertThat(recordList.get(7).value(), is("VIEW:F1, null"));

	assertThat(recordList.get(8).timestamp(), is(6000L));
	assertThat(recordList.get(8).key(), is("F"));
	assertThat(recordList.get(8).value(), is("VIEW:F2, null"));

	assertThat(recordList.get(9).timestamp(), is(7000L));
	assertThat(recordList.get(9).key(), is("F"));
	assertThat(recordList.get(9).value(), is("VIEW:F1, CLICK:F1"));

	assertThat(recordList.get(10).timestamp(), is(7000L));
	assertThat(recordList.get(10).key(), is("F"));
	assertThat(recordList.get(10).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(11).timestamp(), is(8000L));
	assertThat(recordList.get(11).key(), is("G"));
	assertThat(recordList.get(11).value(), is("VIEW:G1, null"));

	assertThat(recordList.get(12).timestamp(), is(9000L));
	assertThat(recordList.get(12).key(), is("G"));
	assertThat(recordList.get(12).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(13).timestamp(), is(9000L));
	assertThat(recordList.get(13).key(), is("G"));
	assertThat(recordList.get(13).value(), is("VIEW:G1, CLICK:G2"));

	assertThat(recordList.get(14).timestamp(), is(12000L));
	assertThat(recordList.get(14).key(), is("B"));
	assertThat(recordList.get(14).value(), is("null, CLICK:B1"));
}
```  

정리하면 `Outer Join` 은 `Inner Join` 의 결과와 `Left Stream` 와 `Right Stream` 에 도착한 모든 레코드를 조인결과로 포함한다고 할 수 있다.  

