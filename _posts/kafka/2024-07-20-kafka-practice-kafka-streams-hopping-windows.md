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

