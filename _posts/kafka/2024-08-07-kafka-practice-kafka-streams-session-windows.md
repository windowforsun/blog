--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Session Window"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams Window 방식 중 Session Window 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Session Window
    - Window
    - Windowing
toc: true
use_math: true
---  

## Session Windows
[Kafka Streams Windowing]()
에서 `Session Windows` 가 무엇이고 어떻게 윈도우를 구성하는지에 대해서는 알아보았다. 
이번 포스팅에서는 실제 `KafkaStreams` 를 사용해서 `Session Windows` 를 구성하고 실제로 윈도우가 어떻게 구성되는지 보다 상세히 살펴볼 것이다.  

먼저 `Session Windows` 에 기본적인 특성은 아래와 같다. 
- 동적인 윈도우 크기(`windowSize`)를 갖는다.  
- 이벤트 기반으로 윈도우가 시작하고 종료한다.  
- 윈도우간 겹침이 존재하지 않는다. 


```
my-event -> SessionWinow process -> session-result
```

예제로 구성하는 `Topology` 는 위와 같다. 
이벤트는 `my-event` 라는 `Inbound Topic` 으로 인입되고, 
`Processor` 에서 `Session Windows` 를 사용해서 이벤트를 집계 시킨뒤 그 결과를 `session-result` 라는 `OutBound Topic` 로 전송한다.  

이후 예시코드의 전체내용은 [여기]()
에서 확인 할 수 있다.  

이전 포스팅인 [TumblingWindows]()
에서 테스트 코드의 기본적인 설정과 공통 내용들이 포함돼 있다. 
그러므로 포스팅에서 설정되지 않은 내용들은 `TumblingWindows` 포스팅에서 확인할 수 있다.   


### Topology
`process` 에 정의된 `SessionWindows` 를 바탕으로 처리하는 내용은 아래와 같다.  

```java
public void processMyEvent(StreamsBuilder streamsBuilder) {
    Serde<String> stringSerde = new Serdes.StringSerde();
    Serde<MyEvent> myEventSerde = new MyEventSerde();
    Serde<MyEventAgg> myEventAggSerde = new MyEventAggSerde();

    Merger<String, MyEventAgg> sessionMerger = (aggKey, aggOne, aggTwo) -> MyEventAgg.builder()
            .firstSeq(Long.min(aggOne.getFirstSeq(), aggTwo.getFirstSeq()))
            .lastSeq(Long.max(aggOne.getLastSeq(), aggTwo.getLastSeq()))
            .count(aggOne.getCount() + aggTwo.getCount())
            .str(aggOne.getStr().concat(aggTwo.getStr()))
            .build();

    streamsBuilder.stream("my-event", Consumed.with(stringSerde, myEventSerde))
            .peek((k, v) -> log.info("session input event {} : {}", k, v))
            .groupByKey()
            .windowedBy(SessionWindows.ofInactivityGapAndGrace(Duration.ofMillis(this.inactivityGap), Duration.ofMillis(this.windowGrade)))
            .aggregate(() -> new MyEventAgg(),
                    ProcessorUtil::aggregateMyEvent,
                    sessionMerger,
                    Materialized.<String, MyEventAgg, SessionStore<Bytes, byte[]>>as("session-window-store")
                            .withKeySerde(stringSerde)
                            .withValueSerde(myEventAggSerde)
            )
            .suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
            .toStream()
            .map((k, v) -> KeyValue.pair(k.key(), v))
            .peek((k, v) -> log.info("session output {} : {}", k, v))
            .to("session-result", Produced.with(stringSerde, myEventAggSerde));
}
```

이벤트 데이터가 토픽을 오고가는 과정을 표현하면 아래와 같다.  

```
my-event -> MyEvent -> process -> MyEventAgg -> session-result
```  

- `windowedBy()` : `Session Windows` 를 구성할 수 있는 `inactivityGap`(무활동 시간) 와 `windowGrace`(윈도우 업데이트 유효시간)을 설정한다. 

이전 시간 베이스 윈도우들은 `aggregate()` 에서 이벤트를 하나로 집계하는 집계 함수인 `aggregator` 만 필요했다. 
`SessionWindows` 의 경우 `inactivityGap` 에 따라 윈도우간 `Merge` 동작도 필수는 아니지만 정의할 수 있다. 
`sessionMerger` 의 구현체를 보면 `aggregator` 를 통해 집계된 두 집계 결과를 다시 하나의 결과로 머지하는 동작을 수행하고 있음을 확인 할 수 있다.  


### Aggregate
여러 `MyEvent` 를 받아 `MyEventAgg` 로 집계하는 동작은 아래와 같다. 

- `firstSeq` : 집계에 사용한 이벤트 중 최소 시퀀스 값
- `lastSeq` : 집계에 사용한 이벤트 중 최대 시퀀스 값
- `count` : 집계에 사용한 이벤트의 수
- `str` : 집계에 사용한 문자열을 연결한 값

[TumblingWindows]()
의 내용과 동일하다.  

### Session Windows Test
`KafkaStreams` 의 테스를 위해선 우선 사전 작업이 필요한데, 
자세한 내용은 [TumblingWindows Test Setup]()
에서 확인 할 수 있다. 

전체 테스트 코드는 [여기]()
에서 확인 할 수 있고, 
테스트에서 무활동 시간은 `10`, 윈도우 업데이트 유효시간은 `0` 로 설정해 진행한다.  


#### Single Key
단일키를 사용하는 총 5개의 이벤트가 아래 코드와 같이 발생될 때 윈도우의 구성과 
각 윈도우에 포함되는 이벤트를 살펴보면 아래와 같다.   

```java
@Test
public void singleKey() {
    this.myEventInput.pipeInput("key1", Util.createMyEvent(1L, "a"), 11L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(2L, "b"), 15L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(3L, "c"), 20L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(4L, "d"), 25L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(5L, "e"), 37L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(6L, "f"), 50L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(7L, "g"), 59L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(9999L, "z"), 9999L);

    List<KeyValue<String, MyEventAgg>> list = this.sessionOutput.readKeyValuesToList();

    assertThat(list, hasSize(3));

    assertThat(list.get(0).value.getFirstSeq(), is(1L));
    assertThat(list.get(0).value.getLastSeq(), is(4L));
    assertThat(list.get(0).value.getStr(), is("abcd"));

    assertThat(list.get(1).value.getFirstSeq(), is(5L));
    assertThat(list.get(1).value.getLastSeq(), is(5L));
    assertThat(list.get(1).value.getStr(), is("e"));

    assertThat(list.get(2).value.getFirstSeq(), is(6L));
    assertThat(list.get(2).value.getLastSeq(), is(7L));
    assertThat(list.get(2).value.getStr(), is("fg"));
}
```

위 테스트 코드에서 발생하는 이벤트와 이를 통해 생성되는 윈도우를 도식화 하면 아래와 같다.  

.. 그림 .. 

윈도우 범위|이벤트
---|---
w1(t11 ~ t35]|a(t11), b(t15), c(t20), d(t25)
w2(t37 ~ t37]|e(t37)
w3(t50 ~ t69]|f(t50), g(t69)



#### Multiple Key
이번에는 1개 이상의 키를 가지는 이벤트가 생성되는 상황을 살펴본다. 

```java
@Test
public void multipleKey() {
    this.myEventInput.pipeInput("key1", Util.createMyEvent(1L, "a"), 11L);
    this.myEventInput.pipeInput("key2", Util.createMyEvent(2L, "b"), 15L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(3L, "c"), 20L);
    this.myEventInput.pipeInput("key2", Util.createMyEvent(4L, "d"), 25L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(5L, "e"), 37L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(6L, "f"), 50L);
    this.myEventInput.pipeInput("key2", Util.createMyEvent(7L, "g"), 59L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(9999L, "z"), 9999L);

    List<KeyValue<String, MyEventAgg>> list = this.sessionOutput.readKeyValuesToList();

    assertThat(list, hasSize(5));

    assertThat(list.get(0).value.getFirstSeq(), is(1L));
    assertThat(list.get(0).value.getLastSeq(), is(3L));
    assertThat(list.get(0).value.getStr(), is("ac"));

    assertThat(list.get(1).value.getFirstSeq(), is(2L));
    assertThat(list.get(1).value.getLastSeq(), is(4L));
    assertThat(list.get(1).value.getStr(), is("bd"));

    assertThat(list.get(2).value.getFirstSeq(), is(5L));
    assertThat(list.get(2).value.getLastSeq(), is(5L));
    assertThat(list.get(2).value.getStr(), is("e"));

    assertThat(list.get(3).value.getFirstSeq(), is(6L));
    assertThat(list.get(3).value.getLastSeq(), is(6L));
    assertThat(list.get(3).value.getStr(), is("f"));

    assertThat(list.get(4).value.getFirstSeq(), is(7L));
    assertThat(list.get(4).value.getLastSeq(), is(7L));
    assertThat(list.get(4).value.getStr(), is("g"));
}
```

위 테스트 코드에서 발생하는 이벤트와 이를 통해 생성되는 윈도우를 도식화 하면 아래와 같다.

.. 그림 ..

키| 윈도우 범위        |이벤트
---|---------------|---
key1| w1(t11 ~ t30] | a(t11), c(t20) 
key2| w2(t15 ~ t35] | b(t15), d(t25) 
key1| w3(t37 ~ t47] | e(t37)        
key1| w4(t50 ~ t60] | f(t57)        
key2| w5(t59 ~ t69] | g(t59)        

결과적으로 집계연산을 수행하기 전 `groupByKey()` 를 사용하고 있으므로 키가 다르다면 별도의 윈도우에 포함되는 것을 확인 할 수 있다. 





---  
## Reference
[SessionWindows](https://kafka.apache.org/21/javadoc/index.html?org/apache/kafka/streams/kstream/SessionWindows.html)  
[Apache Kafka Beyond the Basics: Windowing](https://www.confluent.io/ko-kr/blog/windowing-in-kafka-streams/)  
[Suppressed Event Aggregator](https://developer.confluent.io/patterns/stream-processing/suppressed-event-aggregator/)  



