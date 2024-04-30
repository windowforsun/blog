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
`Create Event` 와 `Update Event` 가 있을 떄, 
`Create Event` 로 아이템이 저장되고, `Update Event` 가 전달된다면 모든 이벤트는 정상 처리 될 수 있다. 
하지만 `Create Event` 전 `Update Event` 가 먼저 온다면 `Update Event` 는 `Create Event` 가 수행 완료 되는 시점까지 대기해야 한다. 
그리고 이벤트가 처리 될 수 있는 최대 시간과 이벤트의 재시도 주기의 시간값을 통해 재시도를 조절 한다.  

`Non-Blocking Retry` 를 위해서는 재시도용 `Topic` 을 사용한다. 
특정 이벤트의 재시도가 필요한 경우 해당 이벤트를 재시도용 토픽으로 보내고, 
이를 다시 소비해서 재시도 여부 판별 후 재시도 수행이 필요하다면 다시 원본 토픽으로 이벤트를 전송하게 된다. 
아래는 이러한 과정을 도식화한 내용이다.

.. 그림 ..

- 현재 시간과 `원본 이벤트 수신 시간` 차이가 `maxRetryDuration` 을 넘어가면 해당 이벤트는 버린다. 
- 위에서 버려지지 않은 이벤트는 현재 시간과 `재시도 수신 시간` 차이가 `retryInterval` 보다 크면 재시도를 수행한다. 
- 재시도 판별이 완료된 이벤트는 원본 토픽에 쓰여지고 다시 소비되고 처리된다. 

아래 그림은 `UpdateEvent` 가 수신 됐을 때 처리되는 전체 괴정을 보여준다.   


.. 그림 ..


소개한 `Non-Blocking Retry` 패턴은 원본 토픽에서 처음 수신된 시간을 사용해서 
`maxRetryDuration` 을 초과했는지 검사하는데 이를 위해
처음 수신 시점의 시간 값을 계속해서 재시도 토픽, 원본 토픽에 쓰여질 때마다 해당 값을 해더로 지니고 있는다. 
그리고 원본 토픽 이름도 해더에 추가해서 재시도가 필요한 시점에 다시 원본 토픽으로 이벤트를 전송하게 된다.  

이 패턴의 핵심은 이벤트가 재시도 자격을 충복하면 원본 토픽으로 전송해서, 다시 주어진 처리가 수행될 수 있도록 한다. 
그리고 처리 수행 도중 재시도가 필요하다고 판단되면 다시 재시도 토픽으로 전달돼 재시도 여부 판별 부터 
`maxRetryDuration` 로 만료 될 떄까지 수행되는 것이다. 
