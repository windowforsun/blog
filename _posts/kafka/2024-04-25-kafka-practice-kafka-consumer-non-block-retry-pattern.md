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

## Kafka Consumer Non-Blocking Retry
`Kafka` 의 `Topic Partition` 에서 에서 메시지를 소비하고 재시도를 수행하고자 한다면, 
`Blocking Retry` 와 `Non-Blocking Retry` 중 하나의 방식을 택해야 한다. 
`Blocking Retry` 는 재시도가 완료 되어야 `Topic Partition` 의 다음 메시지를 소비해서 이벤트 처리를 재개할 수 있다. 
이와 비교해서 `Non-Blocking Retry` 는 `Topic Partition` 의 메시지는 계속해서 소비하고, 
재시도가 필요한 메시지는 별도의 비동기 방식을 사용해 구현한다. 
`Non-Blocking Retry` 는 기존 `Topic Parition` 메시지 소비를 중단 없이 할 수 있지만, 
기존 메시지 순서는 보자될 수 없을 기억해야 한다. 

[Non-Blocking Retry Spring Topics]()
에서도 `Non-Blocking Retry` 에 대한 내용과 간단한 구현 방법에 대해 알아보았다. 
하지만 위 예제는 `Spring Kafka` 를 사용하는 환경에서만 간단한 `Annotation` 정으를 통해서 적용 가능하다. 
이번 포스팅에서는 `Spring` 이나 `Kafka` 가 아닌 다른 메시지 미들웨어를 사용하더라도 도입할 수 있는 
`Non-Blocking Retry Pattern` 에 대해 알아볼 것이다.  

예제의 내용은 [Kafka Consumer Retry]() 
동일하므로 관련 내용을 참고 할 수 있다.  

### Non-Blocking Retry Pattern