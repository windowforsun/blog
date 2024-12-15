--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Join"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 KStream 혹은 KTable 간 조인 연산에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Join
    - Inner Join
    - Outer Join
    - Left Join
    - KStream
    - KTable
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

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-join-1.jpg)


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

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-2.drawio.png)


이후 각 예제에서 사용되는 예제 코드의 전체 내용은 [kafka-streams-join](https://github.com/windowforsun/kafka-streams-kstream-ktable-join-exam)
에서 확인 할 수 있다.  

### KStream-KStream Inner Join
`KStream` 간 조인을 하려면 `Window` 를 반드시 적용해야 하는데, 
이를 위해서는 유의미한 값을 도출할 수 있는 `Window` 의 시간 범위를 정의해야 한다. 
`KStream` 은 `Statess` 하므로 조인을 수행하기 위해서는 스트림상에서 상태를 유지할 기준을 마련해야 하기 때문이다. 
그렇지 않으면 무한한 스트림 전체를 스캔해 상태를 구성해야 한다. 
그리고 두 스트림 조인은 일반적으로 가까운 시간내에 있을 때 의미있는 결과를 도출하기 마련이기 때문이다. 
이렇게 `Window` 에 대한 자세한 내용은 [여기]({{site.baseurl}}{% link _posts/kafka/2024-06-20-kafka-practice-kafka-streams-windowing.md %})
에서 확인 할 수 있다. 
이렇게 `Window` 를 정의해서 조인을 수행하면 
두 스트림에서 `Window` 시간내에 포함되는 모든 레코드들을 기반으로 조인이 수행된다.  

아래는 10초 `Window` 를 사용해서 `View` 와 `Click` 스트림을 조인했을 때의 결과 예시이다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-3.drawio.png)

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

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-4.drawio.png)

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

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-5.drawio.png)

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


### KTable-KTable Inner Join
`KTable` 은 `changelog stream` 으로 `KStream` 의 `Stateless` 한 성격과 달리 `Stateful` 한 성격을 지닌다. 
그리고 `KTable` 은 이러한 `changelog stream` 기반 상태관리를 위해 `State Store` 를 주로 사용한다. 
가장 대표적인 예가 바로 `KStream` 의 레코드를 `State Store` 로 구체화해 키에 대한 최신 값을 포함하는 테이블을 만드는 것이다. 
그러므로 `KTable` 의 조인에는 `KStream` 에서 필수적으로 사용했던 `Join Window` 가 사용되지 않고, 
`KTable` 을 통해 계속해서 업데이트 되는 테이블을 기준으로 조인이 수행된다. 
조인 결과 `KTable` 은 소스 테이블이 업데이트 될 때마다 새로운 조인 결과를 출력하는 레코드이다. 
즉 소스 테이블처럼 구체화되지는 않는다.  

아래는 `View`, `Click` 스트림을 `KTable` 로 구체화 한 후 `Inner Join` 한 결과 예시이다. 

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-6.drawio.png)

- 두 소스 스트림에 존재하는 동일한 키에 대한 조인은 모두 조인 결과로 출력된다. 
- `Join Window` 를 사용하지 않기 때문에, `Join Window` 예시에서는 조인되지 않았던 `B` 키에 대한 조인도 결과로 출력된다. 
- `F` 키 이벤트의 경우 `View KTable` 이 `F1` 으로 설정되고, `F2` 로 업데이트 된 다음 조인 되기 때문에 1개의 조인 결과만 출력된다. 
- `G` 키 이벤트의 경우 `View KTable` 에 `G` 이벤트가 있는 상태에서, `Click Table` 에 `G1` 이 설정 될 때 한번, `G2` 로 업데이트 될 때 한번 조인 결과가 발생해 총 2번 출력된다. 

아래는 코드로 구현한 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KTable<String, String> viewTable = streamsBuilder.table(this.viewTopic, Materialized.as("view-store"));
    KTable<String, String> clickTable = streamsBuilder.table(this.clickTopic, Materialized.as("click-store"));

    KTable<String, String> joinTable = viewTable.join(clickTable,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        });

    joinTable.toStream().to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewTable_clickTable_join() {
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
	assertThat(recordList.get(2).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(3).timestamp(), is(9000L));
	assertThat(recordList.get(3).key(), is("G"));
	assertThat(recordList.get(3).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(4).timestamp(), is(9000L));
	assertThat(recordList.get(4).key(), is("G"));
	assertThat(recordList.get(4).value(), is("VIEW:G1, CLICK:G2"));

	assertThat(recordList.get(5).timestamp(), is(12000L));
	assertThat(recordList.get(5).key(), is("B"));
	assertThat(recordList.get(5).value(), is("VIEW:B1, CLICK:B1"));
}
```  

`KTable` 업데이트에 따른 조인은 삭제동작에 해당하는 [Tombstone](https://medium.com/@damienthomlutz/deleting-records-in-kafka-aka-tombstones-651114655a16)
레코드에도 수행된다는 것을 기억해야 한다. 
여기서 삭제 레코드란 `key=A, payload=null` 과 같은 레코드를 의미한다. 
이러한 레코드로 테이블이 업데이트 되면 `A` 키에 대한 조인 결과도 `null` 값으로 변경되어 전달된다. 

```java
@Test
public void viewTable_clickTable_join2() {
    Util.sendEvent(this.viewEventInput, this.clickEventInput);
    this.viewEventInput.pipeInput("A", null, 13000);
    this.viewEventInput.pipeInput("B", null, 14000);

    List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

    assertThat(recordList, hasSize(8));

    assertThat(recordList.get(0).timestamp(), is(1000L));
    assertThat(recordList.get(0).key(), is("A"));
    assertThat(recordList.get(0).value(), is("VIEW:A1, CLICK:A1"));

    assertThat(recordList.get(1).timestamp(), is(3000L));
    assertThat(recordList.get(1).key(), is("C"));
    assertThat(recordList.get(1).value(), is("VIEW:C1, CLICK:C1"));

    assertThat(recordList.get(2).timestamp(), is(7000L));
    assertThat(recordList.get(2).key(), is("F"));
    assertThat(recordList.get(2).value(), is("VIEW:F2, CLICK:F1"));

    assertThat(recordList.get(3).timestamp(), is(9000L));
    assertThat(recordList.get(3).key(), is("G"));
    assertThat(recordList.get(3).value(), is("VIEW:G1, CLICK:G1"));

    assertThat(recordList.get(4).timestamp(), is(9000L));
    assertThat(recordList.get(4).key(), is("G"));
    assertThat(recordList.get(4).value(), is("VIEW:G1, CLICK:G2"));

    assertThat(recordList.get(5).timestamp(), is(12000L));
    assertThat(recordList.get(5).key(), is("B"));
    assertThat(recordList.get(5).value(), is("VIEW:B1, CLICK:B1"));

    // delete operation
    assertThat(recordList.get(6).timestamp(), is(13000L));
    assertThat(recordList.get(6).key(), is("A"));
    assertThat(recordList.get(6).value(), nullValue());

    assertThat(recordList.get(7).timestamp(), is(14000L));
    assertThat(recordList.get(7).key(), is("B"));
    assertThat(recordList.get(7).value(), nullValue());
}
```  


### KTable-KTable Left Join
`KTable` 간 `Left Join` 은 `KStream` 에서 알아본 개념과 동일하게 `Left Table(Primary:View)` 의 업데이트가 발생 할 때마다 
조인 결과가 발생하는데, `Right Table(Secondary:Click)` 에 매칭되는 키가 있는 경우에는 두 테이블의 값이 조인되고, 없는 경우 `null` 로 설정 된다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-7.drawio.png)

- `Inner Join` 의 결과를 모두 포함한다. 
- `View` 에 이벤트가 먼저 업데이트 되는 레코드에 대해 `(A, null)` 과 같은 결과가 추가로 출력된다.  

아래는 코드로 구현한 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KTable<String, String> viewTable = streamsBuilder.table(this.viewTopic, Materialized.as("view-store"));
    KTable<String, String> clickTable = streamsBuilder.table(this.clickTopic, Materialized.as("click-store"));

    KTable<String, String> joinTable = viewTable.leftJoin(clickTable,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        });

    joinTable.toStream().to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewTable_clickTable_left_join() {
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
	assertThat(recordList.get(7).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(8).timestamp(), is(8000L));
	assertThat(recordList.get(8).key(), is("G"));
	assertThat(recordList.get(8).value(), is("VIEW:G1, null"));

	assertThat(recordList.get(9).timestamp(), is(9000L));
	assertThat(recordList.get(9).key(), is("G"));
	assertThat(recordList.get(9).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(10).timestamp(), is(9000L));
	assertThat(recordList.get(10).key(), is("G"));
	assertThat(recordList.get(10).value(), is("VIEW:G1, CLICK:G2"));

	assertThat(recordList.get(11).timestamp(), is(12000L));
	assertThat(recordList.get(11).key(), is("B"));
	assertThat(recordList.get(11).value(), is("VIEW:B1, CLICK:B1"));
}
```  


### KTable-KTable Outer Join
`KTable` 의 `Outer Join` 또한 `KStream` 에서 알아본 개념과 동일하다. 
`Left Table(Primary:View)`, `Right Table(Secondary:Click)` 에 업데이트 시점과 `Inner Join` 의 결과를 모두 포함한다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-8.drawio.png)

아래는 코드로 구현한 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KTable<String, String> viewTable = streamsBuilder.table(this.viewTopic, Materialized.as("view-store"));
    KTable<String, String> clickTable = streamsBuilder.table(this.clickTopic, Materialized.as("click-store"));

    KTable<String, String> joinTable = viewTable.outerJoin(clickTable,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        });

    joinTable.toStream().to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewTable_clickTable_outer_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	assertThat(recordList, hasSize(14));

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
	assertThat(recordList.get(9).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(10).timestamp(), is(8000L));
	assertThat(recordList.get(10).key(), is("G"));
	assertThat(recordList.get(10).value(), is("VIEW:G1, null"));

	assertThat(recordList.get(11).timestamp(), is(9000L));
	assertThat(recordList.get(11).key(), is("G"));
	assertThat(recordList.get(11).value(), is("VIEW:G1, CLICK:G1"));

	assertThat(recordList.get(12).timestamp(), is(9000L));
	assertThat(recordList.get(12).key(), is("G"));
	assertThat(recordList.get(12).value(), is("VIEW:G1, CLICK:G2"));

	assertThat(recordList.get(13).timestamp(), is(12000L));
	assertThat(recordList.get(13).key(), is("B"));
	assertThat(recordList.get(13).value(), is("VIEW:B1, CLICK:B1"));
}
```  

`KTable` 의 조인은 `KStream` 보다 좀 더 `SQL` 조인과 유사하다. 
차이점으로는 소스 `KTable` 이 업데이트 되면 조인 결과 `KTable` 도 함께 업데이트 된다는 점이다. 


### KStream-KTable Inner Join
`KStream-KTable` 조인을 사용하면 `KStream` 을 통해 무한하게 들어오는 이벤트를 테이블과 실시간으로 조인 할 수 있다. 
`KTable-KTable` 조인과 같이 `Join Window` 를 사용하지 않지만 조인 결과는 `KStream` 이다. 
먼저 알아본 `KStream-KStream`, `KTable-KTable` 과 같은 동일 타입의 조인이 대칭적인 성격이 있었다면, 
`KStream-KTable` 의 조인은 비대칭적인 성격을 띈다. 
여기서 대칭적이라는 것은 조인의 소스가 되는 `Left`, `Right` 중 어느 하나가 업데이트 되더라도 조인 수행의 트리거가 된다는 것으 의미한다. 
비대칭성격을 띄는 `KStream-KTable` 조인은 `Left(KStream)` 만이 조인 수행을 트리거 하고, 
`Right(KTable)` 은 구체화된 테이블만 업데이트한다. 
`Join Window` 를 사용하지 않기 때문에 `KStream` 은 상태가 없으므로 `KTable` 이 조인 수행을 트리거 할 수 없기 때문이다.  

이러한 `KStream-KTable` 은 일반적으로 `KStream` 의 이벤트 데이터를 좀 더 풍부하게 구성하는 용도로 사용 할 수 있다. 
이벤트 키별로 구체화된 `KTable` 을 바탕으로 `KStream` 에 새로운 이벤트가 들어왔을 때 `KTable` 에 있는 추가적인 데이터와 매핑해서 사용할 수 있기 때문이다. 
간단한 예로 키는 사용자 `ID` 이고 `KTable` 에는 사용자에 대한 데이터가 구체화 돼있다고 해보자. 
이때 `View` 이벤트 레코드에는 간략한 정보만 있더라도 `KTable` 과 매핑해서 구체적인 사용자 정보를 실시간으로 조회해서 활용할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-9.drawio.png)

아래는 코드 구현의 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KTable<String, String> clickTable = streamsBuilder.table(this.clickTopic, Materialized.as("click-store"));

    KStream<String, String> joinedStream = viewStream.join(clickTable,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
    });

    joinedStream.to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewStream_clickTable_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	System.out.println(recordList.size());
	recordList.forEach(System.out::println);

	assertThat(recordList, hasSize(1));

	assertThat(recordList.get(0).timestamp(), is(3000L));
	assertThat(recordList.get(0).key(), is("C"));
	assertThat(recordList.get(0).value(), is("VIEW:C1, CLICK:C1"));
}
```  

`KStream-KTable` 조인의 경우 비대칭성으로 소스의 실제 이벤트 전달 시점/순서 따라 조인 결과에 큰 영향을 미친다. 
`KStream-KStream`, `KTable-KTable` 는 실제 이벤트 전달 시점/순서 따라 큰 영향은 미치지 않는데, 
이는 대칭성으로 양쪽에서 조인 트리거가 가능하기 때문이다. 
하지만 `KStream-KTable` 조인의 경우 비대칭적이므로 `KTable` 레코드의 타임스탬프가 `KStream` 레코드보다 직지만, 
우연히 `KStream` 의 레코드가 먼저 처리되면 `KStream` 이 조인을 트리거하는 시점에는 `KTable` 레코드가 존재하지 않기 때문에 조인 결과는 손실된다. (`2.0.x` 이전 기준)  

`2.1.x` 이후 버전 부터는 `Kafka Streams` 에 대한 타임스탬프 동기화가 개선되어 타임스탬프를 기준으로 레코드를 처리할 때 더 나은 정확성을 보장한다. 
이는 `partition head` 레코드의 타임스탬프가 가장 작은 것을 선택하여 처리하는 방안으로, 
소스 스트림이 사용하는 파티션에 하나 이상의 메시지가 존재할 때만 가능하다. 
그래서 사용하는 파티션 중 하나라도 메시지가 존재하지 않는 경우 `max.task.idel.ms` 로 최대 대기시간을 설정 할 수 있다. 
설정한 대기시간 동안 도착하지 않으면, 수행 가능한 레코드를 기반으로 처리를 계속하게 된다.  


### KStream-KTable Left Join
`Left Join` 의 경우 `Inner Join` 을 결과를 포함해서, `KStream` 의 모든 이벤트에 대한 조인 결과를 포함한다. (`KTable` 과 매칭이 안될 경우 `null`)  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-10.drawio.png)


아래는 코드 구현의 예시이다. 

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    KTable<String, String> clickTable = streamsBuilder.table(this.clickTopic, Materialized.as("click-store"));

    KStream<String, String> joinedStream = viewStream.leftJoin(clickTable,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
    });

    joinedStream.to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewStream_clickStream_left_join() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	recordList.forEach(System.out::println);

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

### KStream-GlobalKTable Inner Join
`KStream-GlobalKTable` 조인의 경우 `KStream-KTable` 과 동작은 동일하지만, 
전혀 다른 결과를 출력할 수 있다. 
이는 `GlobalKTable` 이 `KTable` 과는 구체화된 테이블 구성 방삭이 다르기 때문이다. 
`GlobalKTable` 의 경우 구성이 시작되면 소스 토픽의 전체에 대한 구체화된 테이블을 우선 구성한 뒤 처리가 시작된다. 
그리고 업데이트가 발생 한다면 해당 테이블에 계속해서 반영한다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-11.drawio.png)

- 모든 `Click` 토픽에 있는 이벤트를 `GlobalKTable` 로 구성한다. 
- 그 이후 `KStream` 이벤트에 따라 조인이 수행된다. 
- 그러므로 `D` 이벤트를 제외한 모든 이벤트가 조인된다. 
- `G` 키의 경우 `G1` 에서 `G2` 로 업데이트 된 후 `KStream` 이벤트를 기준으로 조인 되기때문에 `G2` 에 대한 조인 결과만 출력된다.  

아래는 코드 구현의 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    GlobalKTable<String, String> clickGlobalTable = streamsBuilder.globalTable(this.clickTopic, Materialized.as("click-store"));

    KStream<String, String> joinedStream = viewStream.join(clickGlobalTable,
        (leftViewKey, rightClickKey) -> leftViewKey,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        });

    joinedStream.to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewStream_clickGlobalTable_join_presetGlobalTable() {
	Util.sendClickEvent(this.clickEventInput);
	Util.sendViewEvent(this.viewEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	assertThat(recordList, hasSize(6));

	assertThat(recordList.get(0).timestamp(), is(0L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, CLICK:A1"));

	assertThat(recordList.get(1).timestamp(), is(1000L));
	assertThat(recordList.get(1).key(), is("B"));
	assertThat(recordList.get(1).value(), is("VIEW:B1, CLICK:B1"));

	assertThat(recordList.get(2).timestamp(), is(3000L));
	assertThat(recordList.get(2).key(), is("C"));
	assertThat(recordList.get(2).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(3).timestamp(), is(6000L));
	assertThat(recordList.get(3).key(), is("F"));
	assertThat(recordList.get(3).value(), is("VIEW:F1, CLICK:F1"));

	assertThat(recordList.get(4).timestamp(), is(6000L));
	assertThat(recordList.get(4).key(), is("F"));
	assertThat(recordList.get(4).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(5).timestamp(), is(8000L));
	assertThat(recordList.get(5).key(), is("G"));
	assertThat(recordList.get(5).value(), is("VIEW:G1, CLICK:G2"));
}
```  

`KStream-GlobalKTable` 조인의 경우 시작 시점 부터 구체화된 테이블을 먼저 구축하고 처리가 시작되므로, 
`KStream-KTable` 과 같은 런타임 시점/순서에 따른 의존성은 가지지 않는다.  

추가적으로 앞선 코드에서 테스트 예시는 다른 테스트 코드와는 달리, 
`GlobalKTable` 을 선 구축하기 위해 `Click` 이벤트를 먼저 전체 발송 후 `View` 이벤트를 발생 했다. 
만약 기존 테스트 코드와 같이 `View` 와 `Click` 이벤트가 기존 순서대로 발생한다면 아래와 같이 `KStream-KTable` 결과와 동일하다.  

```java
@Test
public void viewStream_clickGlobalTable_join_not_presetGlobalTable() {
    Util.sendEvent(this.viewEventInput, this.clickEventInput);

    List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

    assertThat(recordList, hasSize(1));

    assertThat(recordList.get(0).timestamp(), is(3000L));
    assertThat(recordList.get(0).key(), is("C"));
    assertThat(recordList.get(0).value(), is("VIEW:C1, CLICK:C1"));
}
```  


### KStream-GlobalKTable Left Join
`Left Join` 은 기존 `Inner Join` 의 결과에서 `D` 키 이벤트에 대한 결과가 포함된다.  

![그림 1]({{site.baseurl}}/img/kafka/kkafka-streams-join-12.drawio.png)

아래는 코드 구현의 예시이다.  

```java
public void process(StreamsBuilder streamsBuilder) {
    KStream<String, String> viewStream = streamsBuilder.stream(this.viewTopic);
    GlobalKTable<String, String> clickGlobalTable = streamsBuilder.globalTable(this.clickTopic, Materialized.as("click-store"));

    KStream<String, String> joinedStream = viewStream.leftJoin(clickGlobalTable,
        (leftViewKey, rightClickKey) -> leftViewKey,
        (leftViewValue, rightClickValue) -> {
            String result = leftViewValue + ", " + rightClickValue;
            log.info(result);
            return result;
        });

    joinedStream.to(this.resultTopic, Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void viewStream_clickGlobalTable_join_not_presetGlobalTable() {
	Util.sendEvent(this.viewEventInput, this.clickEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	System.out.println(recordList.size());
	recordList.forEach(System.out::println);

	assertThat(recordList, hasSize(7));

	assertThat(recordList.get(0).timestamp(), is(0L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, null"));

	assertThat(recordList.get(1).timestamp(), is(1000L));
	assertThat(recordList.get(1).key(), is("B"));
	assertThat(recordList.get(1).value(), is("VIEW:B1, null"));

	assertThat(recordList.get(2).timestamp(), is(3000L));
	assertThat(recordList.get(2).key(), is("C"));
	assertThat(recordList.get(2).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(3).timestamp(), is(4000L));
	assertThat(recordList.get(3).key(), is("D"));
	assertThat(recordList.get(3).value(), is("VIEW:D1, null"));

	assertThat(recordList.get(4).timestamp(), is(6000L));
	assertThat(recordList.get(4).key(), is("F"));
	assertThat(recordList.get(4).value(), is("VIEW:F1, null"));

	assertThat(recordList.get(5).timestamp(), is(6000L));
	assertThat(recordList.get(5).key(), is("F"));
	assertThat(recordList.get(5).value(), is("VIEW:F2, null"));

	assertThat(recordList.get(6).timestamp(), is(8000L));
	assertThat(recordList.get(6).key(), is("G"));
	assertThat(recordList.get(6).value(), is("VIEW:G1, null"));
}

@Test
public void viewStream_clickGlobalTable_join_presetGlobalTable() {
	Util.sendClickEvent(this.clickEventInput);
	Util.sendViewEvent(this.viewEventInput);

	List<TestRecord<String, String>> recordList = this.resultOutput.readRecordsToList();

	System.out.println(recordList.size());
	recordList.forEach(System.out::println);

	assertThat(recordList, hasSize(7));

	assertThat(recordList.get(0).timestamp(), is(0L));
	assertThat(recordList.get(0).key(), is("A"));
	assertThat(recordList.get(0).value(), is("VIEW:A1, CLICK:A1"));

	assertThat(recordList.get(1).timestamp(), is(1000L));
	assertThat(recordList.get(1).key(), is("B"));
	assertThat(recordList.get(1).value(), is("VIEW:B1, CLICK:B1"));

	assertThat(recordList.get(2).timestamp(), is(3000L));
	assertThat(recordList.get(2).key(), is("C"));
	assertThat(recordList.get(2).value(), is("VIEW:C1, CLICK:C1"));

	assertThat(recordList.get(3).timestamp(), is(4000L));
	assertThat(recordList.get(3).key(), is("D"));
	assertThat(recordList.get(3).value(), is("VIEW:D1, null"));

	assertThat(recordList.get(4).timestamp(), is(6000L));
	assertThat(recordList.get(4).key(), is("F"));
	assertThat(recordList.get(4).value(), is("VIEW:F1, CLICK:F1"));

	assertThat(recordList.get(5).timestamp(), is(6000L));
	assertThat(recordList.get(5).key(), is("F"));
	assertThat(recordList.get(5).value(), is("VIEW:F2, CLICK:F1"));

	assertThat(recordList.get(6).timestamp(), is(8000L));
	assertThat(recordList.get(6).key(), is("G"));
	assertThat(recordList.get(6).value(), is("VIEW:G1, CLICK:G2"));
}
```  

### Partition & Parallelization
`Kafka` 는 기본적으로 `Consumer Group` 그룹이 하나의 `Topic` 을 담당하는 구조이다. 
그리고 `Consumer Group` 은 1개 이상의 `Consumer Instance` 로 구성되고, 
`Topic` 은 1개 이상의 `Partition` 으로 구성된다. 
`Consumer Instance` 는 하나 이상의 `Topic Partition` 을 할당 받아 `Topic` 으로 들어오는 메시지를 소비하는 구조가 되는 것이다.  

위와 같은 구조에서 `State Store` 를 사용하고, `Join` 을 사용하는 `Kafka Streams` 애플리케이션을 가정해 보자. 
이때 중요한 것은 `Join` 을 수행하려면 두 소스 `Topic` 이 `Co-Partitioned` 되어야 한다는 것이다. 
이는 두 `Topic` 이 동알한 수의 `Partition` 을 가진다는 의미로, 그렇지 않으면 `Join` 은 수행될 수 없다. (`GlobalKTable` 제외)
더 나아가 해당 두 소스 `Topic` 으로 메시지를 생산하는 `Produce` 가 사용하는 `Partitioner` 또한 동일해야 한다. 
앞서 사용한 예제에서 `View` 토픽의 `A` 키 이벤트는 0번 파티션으로 전달되고, `Click` 토픽의 `A` 키 이벤트는 1번 파티션으로 전달된다면, 
`Partition` 의 수가 동일하더라더 기대하는 `Join` 결과는 얻을 수 없게 된다.  

위와 같은 몇가지 제약사항에서 벗어날 수 있는 방법이 바로 `GlobalKTable` 의 사용이다. 
`GlobalKTable` 은 소스 토픽의 모든 파티션 데이터 전체를 복사본으로 보유하는 특성이 있기 대문에, 
`GlobalKTable` 과 `Join` 연산을 수행하는 다른 소스 토픽의 파티션 수/파티셔너가 일치할 필요 없다. 
하지만 이런 `GlobalKTable` 은 주로 정적인 데이터에만 사용해야하는데, 그 이유는 메모리/디스크 사용량이 크게 증가할 수 있다는 점에 있다. 
또한 `GlobalKTable` 의 특성으로 재처리와 같은 연산에 큰 어려움이 있다. 
그러므로 `GlobalKTable` 은 좋은 대안일 수는 있지만, 정적인 데이터에만 사용하고 동적인 데이터에는 `KTable` 을 사용해야 한다.  



---  
## Reference
[Crossing the Streams – Joins in Apache Kafka](https://www.confluent.io/blog/crossing-streams-joins-apache-kafka/)   
[KIP-99: Add Global Tables to Kafka Streams](https://cwiki.apache.org/confluence/display/KAFKA/KIP-99%3A+Add+Global+Tables+to+Kafka+Streams)   
[GlobalKTable](https://docs.confluent.io/platform/current/streams/concepts.html#globalktable)   
[KTable](https://docs.confluent.io/platform/current/streams/concepts.html#ktable)   
[Kafka Streams Joining](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#joining)   

