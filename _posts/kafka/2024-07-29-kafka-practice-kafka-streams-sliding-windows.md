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

## Sliding Windows
[Kafka Streams Windowing]()
에서 `Sliding Windows` 가 무엇이고 어떻게 윈도우를 구성하는지에 대해서는 알아보았다. 
이번 포스팅에서는 실제 `KafkaStreams` 를 사용해서 `Sliding Windows` 를 구성하고 실제로 윈도우가 어떻게 구성되는지 보다 상세히 살펴볼 것이다.  

먼저 `Sliding Windows` 에 기본적인 특성은 아래와 같다. 
- 고정된 윈도우 크기(`windowSize`)를 갖는다. 
- 윈도우는 이벤트의 발생에 따라 윈도우의 구성원이 달라질때 마다 생성된다. 
- 그러므로 이벤트에 따라 윈도우간 겹침이 존재하고, 하나의 이벤트는 여러 윈도우에 포함될 수 있다. 


```
my-event -> SlidingWindows process -> sliding-result
```

예제로 구성하는 `Topology` 는 위와 같다. 
이벤트는 `my-event` 라는 `Inbound Topic` 으로 인입되고, 
`Processor` 에서 `Sliding Windows` 를 사용해서 이벤트를 집계 시킨뒤 그 결과를 `sliding-result` 라는 `OutBound Topic` 로 전송한다.  

이후 예시코드의 전체내용은 [여기]()
에서 확인 할 수 있다.  

이전 포스팅인 [TumblingWindows]()
에서 테스트 코드의 기본적인 설정과 공통 내용들이 포함돼 있다. 
그러므로 포스팅에서 설정되지 않은 내용들은 `TumblingWindows` 포스팅에서 확인할 수 있다.   


### Topology
`process` 에 정의된 `SlidingWindows` 를 바탕으로 처리하는 내용은 아래와 같다.  

```java
public void processMyEvent(StreamsBuilder streamsBuilder) {
    Serde<String> stringSerde = new Serdes.StringSerde();
    Serde<MyEvent> myEventSerde = new MyEventSerde();
    Serde<MyEventAgg> myEventAggSerde = new MyEventAggSerde();

    streamsBuilder.stream("my-event", Consumed.with(stringSerde, myEventSerde))
            .peek((k, v) -> log.info("sliding input {} : {}", k, v))
            .groupByKey()
            .windowedBy(SlidingWindows.ofTimeDifferenceAndGrace(Duration.ofMillis(this.windowDuration), Duration.ofMillis(this.windowGrace)))
            .aggregate(() -> new MyEventAgg(),
                    ProcessorUtil::aggregateMyEvent,
                    Materialized.<String, MyEventAgg, WindowStore<Bytes, byte[]>>as("sliding-window-store")
                            .withKeySerde(stringSerde)
                            .withValueSerde(myEventAggSerde)
            )
            .suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
            .toStream()
            .map((k, v) -> KeyValue.pair(k.key(), v))
            .peek((k, v) -> log.info("sliding output {} : {}", k, v))
            .to("sliding-result", Produced.with(stringSerde, myEventAggSerde));
}
```

이벤트 데이터가 토픽을 오고가는 과정을 표현하면 아래와 같다.  

```
my-event -> MyEvent -> process -> MyEventAgg -> sliding-result
```  

- `windowedBy()` : `Sliding Windows` 를 구성할 수 있는 `windowSize`(윈도우 크기) 와 `windowGrace`(윈도우 업데이트 유효시간)을 설정한다. 

여기서 `SlidingWindows` 를 설정한 방법인 `SlidingWindows.ofTimeDifferenceAndGrace()` 은
첫번째 인자인 `windowSize` 가 윈도우의 크기이자 무활동 시간을 의미한다. 
이는 `windowSize=10s` 일 경우 이전 이벤트 이후에 `10s` 이내에 발생하는 모든 이벤트는 동일한 윈도우에 포함될 수 있다는 의미이다. 
그러므로 위 방식은 고정된 슬라이딩 간격이 존재하지 않고, 
각 윈도우는 이벤트의 발생과 관련해 동적으로 생성된다. 
이벤트가 불규칙하게 발생하고 윈도우의 크기를 이벤트 간의 시간 차이에 기반해서 조정하고 싶을 경우 유용한 방법이다.  



### Aggregate
여러 `MyEvent` 를 받아 `MyEventAgg` 로 집계하는 동작은 아래와 같다. 

- `firstSeq` : 집계에 사용한 이벤트 중 최소 시퀀스 값
- `lastSeq` : 집계에 사용한 이벤트 중 최대 시퀀스 값
- `count` : 집계에 사용한 이벤트의 수
- `str` : 집계에 사용한 문자열을 연결한 값

[TumblingWindows]()
의 내용과 동일하다.  

### Sliding Windows Test
`KafkaStreams` 의 테스를 위해선 우선 사전 작업이 필요한데, 
자세한 내용은 [TumblingWindows Test Setup]()
에서 확인 할 수 있다. 

전체 테스트 코드는 [여기]()
에서 확인 할 수 있고, 
테스트에서 윈도우 크기는 `30`, 윈도우 업데이트 유효시간은 `1` 로 설정해 진행한다.  

