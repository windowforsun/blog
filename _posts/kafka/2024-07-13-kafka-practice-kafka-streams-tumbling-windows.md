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
