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

## Kafka Streams Windowing
`Window` 란 정해진 시간 범위 안에 있는 데이터를 의미한다. 
윈도우에 대한 시간 범위, 유형 등과 같은 특성은 필요에 따라 정의할 수 있다. 
그리고 윈도우 처리에 대상이 되는 데이터는 시간 범위에 해당하는 데이터라는 점을 기억해야 한다. 
아래는 이러한 시간과 이벤트 그리고 윈도우의 관계를 도식화 해서 보여주는 그림이다.  


.. 그림 .. 

`t0 ~ t60` 까지 시간이 있을 때, `A-E` 이벤트가 발생했다. 
여기서 윈도우범위가 `t10 ~ t40` 이라면 해당 윈도우에 포함되는 이벤트는 `B, C, D, E` 가 된다.  
즉 윈도우 기반 처리를 한다고 했을 때 해당 윈도우에서 처리하는 데이터는 4개인 것이다.  

그리고 윈도우 유형을 어떤 것을 선택하냐에 따라 
하나의 이벤트가 단 한개의 윈도우에만 포함될 수 있고, 그 이상의 윈도우에 포함 될 수도 있다.  

`Kafka Streams` 는 아래와 같은 윈도우 유형을 제공한다. 
각 유형은 아래에서 자세한 설명을 이어가겠지만 추가적으로 [여기](https://www.confluent.io/ko-kr/blog/windowing-in-kafka-streams/)
에서도 확인 할 수 있다.  

윈도우 범위는 시작시간은 포함하고 종료시간은 포함하지 않는다. 
즉 만약 윈도우 범위가 `w1[t10 ~ t20)` 이라면 이는 `t10 <= time range < t20` 인 것이다.  

이름| 동작            |설명
---|---------------|---
Hoping| Time-based    |- 고정된 윈도우 크기<br>- 윈도우간 겹침
Tumbling| Time-based    |- 고정된 윈도우 크기<br>- 윈도우간 겹치지 않음
Sliding| Time-based    |- 고정된 윈도우 크기<br>- 이벤트 시점에 따라 윈도우가 겹치거나 안 겹침<br>
Session| Session-based |- 동적인 윈도우 크기<br>- 윈도우간 겹치지 않음<br>- 이벤트 기반 윈도우

