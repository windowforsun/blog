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
