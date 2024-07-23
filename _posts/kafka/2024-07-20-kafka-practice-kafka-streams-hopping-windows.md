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

## Hopping Windows
[Kafka Streams Windowing]()
에서 `Hopping Windows` 가 무엇이고 어떻게 윈도우를 구성하는지에 대해서는 알아보았다. 
이번 포스팅에서는 실제 `KafkaStreams` 를 사용해서 `Hopping Windows` 를 구성하고 실제로 윈도우가 어떻게 구성되는지 보다 상세히 살펴볼 것이다.  

먼저 `Hopping Windows` 에 기본적인 특성은 아래와 같다. 
- 고정된 윈도우 크기(`windowSize`)를 갖는다. 
- 윈도우의 진행 간격(`windowAdvance`)은 `windowSize` 보다 같거나 작다.   
- 그러므로 윈도우간 겹침이 존재하고, 하나의 이벤트는 여러 윈도우에 포함될 수 있다. 


```
my-event -> HoppingWindows process -> hopping-result
```

예제로 구성하는 `Topology` 는 위와 같다. 
이벤트는 `my-event` 라는 `Inbound Topic` 으로 인입되고, 
`Processor` 에서 `Hopping Windows` 를 사용해서 이벤트를 집계 시킨뒤 그 결과를 `hopping-result` 라는 `OutBound Topic` 로 전송한다.  

이후 예시코드의 전체내용은 [여기]()
에서 확인 할 수 있다.  

이전 포스팅인 [TumblingWindows]()
에서 테스트 코드의 기본적인 설정과 공통 내용들이 포함돼 있다. 
그러므로 포스팅에서 설정되지 않은 내용들은 `TumblingWindows` 포스팅에서 확인할 수 있다.   


### Topology
`process` 에 정의된 `HoppingWindows` 를 바탕으로 처리하는 내용은 아래와 같다.  

```java
public void processMyEvent(StreamsBuilder streamsBuilder) {
    Serde<String> stringSerde = new Serdes.StringSerde();
    Serde<MyEvent> myEventSerde = new MyEventSerde();
    Serde<MyEventAgg> myEventAggSerde = new MyEventAggSerde();

    streamsBuilder
            .stream("my-event", Consumed.with(stringSerde, myEventSerde))
            .peek((k, v) -> log.info("hopping input {} : {}", k, v))
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMillis(this.windowDuration)).advanceBy(Duration.ofMillis(this.windowAdvance)))
            .aggregate(() -> new MyEventAgg(),
                    ProcessorUtil::aggregateMyEvent,
                    Materialized.<String, MyEventAgg, WindowStore<Bytes, byte[]>>as("hopping-window-store")
                            .withKeySerde(stringSerde)
                            .withValueSerde(myEventAggSerde)
            )
            .suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
            .toStream()
            .map((k, v) -> KeyValue.pair(k.key(), v))
            .peek((k, v) -> log.info("hopping output {} : {}", k, v))
            .to("hopping-result", Produced.with(stringSerde, myEventAggSerde));
}
```  

이벤트 데이터가 토픽을 오고가는 과정을 표현하면 아래와 같다.  

```
my-event -> MyEvent -> process -> MyEventAgg -> hopping-result
```  

- `windowedBy()` : `Hopping Windows` 를 구성할 수 있는 `windowSize`(윈도우 크기) 와 `windowAdvance`(윈도우 진행 간격)을 설정한다. 
  - 윈도우 진행 간격이란 윈도우가 생성되는 주기를 의미한다. 


### Aggregate
여러 `MyEvent` 를 받아 `MyEventAgg` 로 집계하는 동작은 아래와 같다. 

- `firstSeq` : 집계에 사용한 이벤트 중 최소 시퀀스 값
- `lastSeq` : 집계에 사용한 이벤트 중 최대 시퀀스 값
- `count` : 집계에 사용한 이벤트의 수
- `str` : 집계에 사용한 문자열을 연결한 값

[TumblingWindows]()
의 내용과 동일하다.  

### Hopping Windows Test
`KafkaStreams` 의 테스를 위해선 우선 사전 작업이 필요한데, 
자세한 내용은 [TumblingWindows Test Setup]()
에서 확인 할 수 있다. 

전체 테스트 코드는 [여기]()
에서 확인 할 수 있고, 
테스트에서 윈도우 크기는 `10`, 윈도우 진행 간격은 `5` 로 설정해서 수행한다. 


#### Single Key
단일키를 사용하는 총 4개의 이벤트가 아래 코드와 같이 발생될 때 윈도우의 구성과 
각 윈도우에 포함되는 이벤트를 살펴보면 아래와 같다.   

```java
@Test
public void singleKey_eachWindow_twoEvents() {
    this.myEventInput.pipeInput("key1", Util.createMyEvent(1L, "a"), 0L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(2L, "b"), 7L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(3L, "c"), 10L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(4L, "d"), 15L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(5L, "z"), 30L);

    List<KeyValue<String, MyEventAgg>> list = this.hoppingResultOutput.readKeyValuesToList();

    assertThat(list, hasSize(4));

    assertThat(list.get(0).value.getFirstSeq(), is(1L));
    assertThat(list.get(0).value.getLastSeq(), is(2L));
    assertThat(list.get(0).value.getStr(), is("ab"));

    assertThat(list.get(1).value.getFirstSeq(), is(2L));
    assertThat(list.get(1).value.getLastSeq(), is(3L));
    assertThat(list.get(1).value.getStr(), is("bc"));

    assertThat(list.get(2).value.getFirstSeq(), is(3L));
    assertThat(list.get(2).value.getLastSeq(), is(4L));
    assertThat(list.get(2).value.getStr(), is("cd"));

    assertThat(list.get(3).value.getFirstSeq(), is(4L));
    assertThat(list.get(3).value.getLastSeq(), is(4L));
    assertThat(list.get(3).value.getStr(), is("d"));
}
```

위 테스트 코드에서 발생하는 이벤트와 이를 통해 생성되는 윈도우를 도식화 하면 아래와 같다.  

.. 그림 .. 

윈도우 범위|이벤트
---|---
w1(t0 ~ t10]|a(t0), b(t7)
w2(t5 ~ t15]|b(t7), c(t10)
w3(t10 ~ t20]|c(t10), d(t15)
w4(t15 ~ t25]|d(t15)

#### Multiple Key
이번에는 1개 이상의 키를 가지는 이벤트가 생성되는 상황을 살펴본다. 

```java
@Test
public void multipleKey_eachWindow_oneEvents_duplicated2() {
    this.myEventInput.pipeInput("key1", Util.createMyEvent(1L, "a"), 7L);
    this.myEventInput.pipeInput("key2", Util.createMyEvent(2L, "b"), 10L);
    this.myEventInput.pipeInput("key2", Util.createMyEvent(3L, "c"), 16L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(4L, "d"), 19L);
    this.myEventInput.pipeInput("key1", Util.createMyEvent(5L, "z"), 30L);

    List<KeyValue<String, MyEventAgg>> list = this.hoppingResultOutput.readKeyValuesToList();

    assertThat(list, hasSize(7));

    assertThat(list.get(0).value.getFirstSeq(), is(1L));
    assertThat(list.get(0).value.getLastSeq(), is(1L));
    assertThat(list.get(0).value.getStr(), is("a"));

    assertThat(list.get(1).value.getFirstSeq(), is(1L));
    assertThat(list.get(1).value.getLastSeq(), is(1L));
    assertThat(list.get(1).value.getStr(), is("a"));

    assertThat(list.get(2).value.getFirstSeq(), is(2L));
    assertThat(list.get(2).value.getLastSeq(), is(2L));
    assertThat(list.get(2).value.getStr(), is("b"));

    assertThat(list.get(3).value.getFirstSeq(), is(4L));
    assertThat(list.get(3).value.getLastSeq(), is(4L));
    assertThat(list.get(3).value.getStr(), is("d"));

    assertThat(list.get(4).value.getFirstSeq(), is(2L));
    assertThat(list.get(4).value.getLastSeq(), is(3L));
    assertThat(list.get(4).value.getStr(), is("bc"));

    assertThat(list.get(5).value.getFirstSeq(), is(4L));
    assertThat(list.get(5).value.getLastSeq(), is(4L));
    assertThat(list.get(5).value.getStr(), is("d"));

    assertThat(list.get(6).value.getFirstSeq(), is(3L));
    assertThat(list.get(6).value.getLastSeq(), is(3L));
    assertThat(list.get(6).value.getStr(), is("c"));

}
```

위 테스트 코드에서 발생하는 이벤트와 이를 통해 생성되는 윈도우를 도식화 하면 아래와 같다.

.. 그림 ..

키| 윈도우 범위        |이벤트
---|---------------|---
key1| w1(t0 ~ t10]  |a(t7)
key1| w2(t5 ~ t15]  |a(t7)         
key2| w3(t5 ~ t15]  | b(t10) 
key1| w4(t10 ~ t20] | d(t19)           
key2| w5(t10 ~ t20] | b(t10), c(t16)   
key1| w6(t15 ~ t25] | d(t19)           
key2| w7(t15 ~ t25] | c(t16)       

결과적으로 집계연산을 수행하기 전 `groupByKey()` 를 사용하고 있으므로 키가 다르다면 별도의 윈도우에 포함되는 것을 확인 할 수 있다. 





---  
## Reference
[Kafka Streams Windowing - Hopping Windows](https://www.lydtechconsulting.com/blog-kafka-streams-windows-hopping.html)  
[Apache Kafka Beyond the Basics: Windowing](https://www.confluent.io/ko-kr/blog/windowing-in-kafka-streams/)  
[Suppressed Event Aggregator](https://developer.confluent.io/patterns/stream-processing/suppressed-event-aggregator/)  



