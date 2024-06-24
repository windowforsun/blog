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


### Tumbling Window
`Tumbling Window` 는 윈도우간 겹침 없이 고정된 윈도우 크기와 연속적인 순서를 갖으며, 
윈도우를 대표하는 특성을 지닌 윈도우이다. 

.. 그림 ..

시간 프레임에서 윈도우간 겹침이 없기 때문에, 이벤트는 1개의 윈도우에만 포함된다. 
위 그림은 `Window size` 가 `20t` 일 떄, 
5개의 이벤트가 발생 했을 때 생성되는 윈도우와 각 윈도우가 포함하는 이벤트를 정리하면 아래와 같다.  

- `w1[t0 ~ t20)` - A(t8), B(t12)
- `w3[t20 ~ t40)` - C(t20), D(t25), E(t34)

`Tumbing Window` 는 `t0` 부터 항상 시작하고 시작 포함하지만 종료시간은 포함하지 않는 다는 점을 기억해야 한다. 
`t20` 에 발생한 `C` 이벤트는 시간 범위가 20일 때 `w1` 에 포함되지 않고, 
다음 윈도우인 `w2` 에 포함된 것이 바로 그 이유다. 

