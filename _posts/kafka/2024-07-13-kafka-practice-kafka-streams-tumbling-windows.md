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

## Tumbling Windows
[Kafka Streams Windowing]()
에서 `Tumbling Windows` 가 무엇이고 어떻게 윈도우를 구성하는지에 대해서는 알아보았다. 
이번 포스팅에서는 실제 `KafkaStreams` 를 사용해서 `Tubmling Windows` 를 구성하고 실제로 윈도우가 어떻게 구성되는지 보다 상세히 살펴볼 것이다.  

먼저 `Tubmling Windows` 에 기본적인 특성은 아래와 같다. 
- 고정된 윈도우 크기(`windowSize`)를 갖는다. 
- 윈도우의 진행 간격(`advanceInterval`)과 윈도우 크기가 동일하다.  
- 그러므로 윈도우간 겹침이 존재하지 않는다.  


```
my-event -> TumblingWindows process -> tumbling-result
```

예제로 구성하는 `Topology` 는 위와 같다. 
이벤트는 `my-event` 라는 `Inbound Topic` 으로 인입되고, 
`Processor` 에서 `Tumbling Windows` 를 사용해서 이벤트를 집계 시킨뒤 그 결과를 `tumbling-result` 라는 `OutBound Topic` 로 전송한다.  

예제에서는 윈도우로 집계된 결과만을 보기위해 [suppress()](https://developer.confluent.io/patterns/stream-processing/suppressed-event-aggregator/) 
를 사용한다. 
`suppress()` 는 윈도우 구성에서 필수는 아니지만, 이를 사용하면 윈도우의 최종 집계 결과만 받아 볼 수 있다. 
실제 윈도우 동작에는 윈도우의 최종 결과가 아닌 중간 중간 윈도우가 업데이트 될때 마다 연속적인 집계를 처리할 수 있지만, 
테스트를 통해 알아보고자 하는 것이 최종 결과이기 때문에 사용하였다.  

이후 예시코드의 전체내용은 [여기]()
에서 확인 할 수 있다.  

### Topology
`Processor` 에 정의된 `Tumbling Windows` 를 바탕으로 처리하는 내용은 아래와 같다.  

```java
public void processMyEvent(StreamsBuilder streamsBuilder) {
    Serde<String> stringSerde = new Serdes.StringSerde();
    Serde<MyEvent> myEventSerde = new MyEventSerde();
    Serde<MyEventAgg> myEventAggSerde = new MyEventAggSerde();

    streamsBuilder.stream("my-event", Consumed.with(stringSerde, myEventSerde))
            .peek((k, v) -> log.info("tumbling input {} : {}", k, v))
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMillis(this.windowDuration), Duration.ofMillis(this.windowGrace)))
            .aggregate(() -> new MyEventAgg(),
                    ProcessorUtil::aggregateMyEvent,
                    Materialized.<String, MyEventAgg, WindowStore<Bytes, byte[]>>as("tumbling-window-store")
                            .withKeySerde(stringSerde)
                            .withValueSerde(myEventAggSerde)
            )
            .suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
            .toStream()
            .map((k, v) -> KeyValue.pair(k.key(), v))
            .peek((k, v) -> log.info("tumbling output {} : {}", k, v))
            .to("tumbling-result", Produced.with(stringSerde, myEventAggSerde));

}
```

이벤트 데이터가 토픽을 오고가는 과정을 표현하면 아래와 같다.  

```
my-event -> MyEvent -> process -> MyEventAgg -> tumbling-result
```  

- `stream()` : `Topology` 의 소스 토픽을 설정한다. 그리고 해당 토픽에서 인입되는 키와 값에 대한 직렬화/역직렬화에 필요한 설정도 포함한다. 
- `groupByKey()` : 이벤트 집계를 위해 우선 이벤트의 키별로 그룹화 한다. `groupBy` 와 다르게 키별 파티셔닝이 발생한다. 
- `windowedBy()` : `Tumbling Windows` 를 구성할 수 있는 `windowSize` 와 윈도우 업데이트 유효시간(`windowGrace`) 시간값을 설정한다.  
- `aggregate()` : 윈도우 크기 시간 동안 이벤트에 대한 처리를 한다. 
`reduce` 는 입력된 여러 값을 동일한 타입의 단일 값으로 결합하는 반면, 
`aggregate` 는 입력된 여러 값을 다른 타입의 단일 값으로 반환한다. 이러한 집계연산을 위해 아래와 같은 파라미터가 필요하다. 
  - 집계연산의 결과로 리턴할 값을 초기화한다. (`MyEventAgg`)
  - 실제 집계 연산을 수행하는 메서드(`aggregateMyEvent`)
  - 집계를 수행하는 동안 저장할 저장소와 저장시 사용할 직렬화/역직렬화, 윈도우 저장소로는 `RocksDB` 를 사용한다. (`Materialized`)
- `suppress()` : 억제를 통해 윈도우의 최종결과만 받는다. 사용하지 않을 윈도우 크기 동안 업데이트 될때 마다 결과를 받게 된다. 
- `toStream()` : `aggregate()`, `suppress()` 는 `KTable` 을 사용하기 때문에 이를 다시 `KStream` 으로 변환해 준다. 
- `map()` : 최종 이벤트에 대한 키와 값을 매칭해 `KeyValue` 타입으로 변환한다. 
- `to()` : 최종 이벤트를 전송할 토픽을 명시하고, 사용할 직렬화/역직렬화를 설정한다. 
- `peek()` : 스트림 처리에서 중간에 아이템을 확인해 볼 수 있는 메소드이다. 
