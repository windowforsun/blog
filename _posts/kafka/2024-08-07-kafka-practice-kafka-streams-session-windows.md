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
