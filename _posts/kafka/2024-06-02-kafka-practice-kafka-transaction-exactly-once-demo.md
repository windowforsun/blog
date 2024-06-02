--- 
layout: single
classes: wide
title: "[Kafka] Kafka Transaction Exactly Once Demo"
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
    - Transaction
    - Kafka Transaction
    - Exactly-Once
    - Isolation Level
    - Consumer
    - Producer
toc: true
use_math: true
---

## Kafka Transaction Demo
[]()
에서 `Kafka Transaction` 과 `Exactly-Once` 에 대해 개념적으로 알아 보았다.
이번 포스팅에서는 이를 검증해볼 수 있는 애플리케이션을 통해 구현 방식을 바탕으로 좀 더 알아보고자한다. 

### Demo
데모 애플리케이션은 `Kafka Transaction` 을 통해 `Exactly-Once` 를 테스트해볼 수 있다. 
`Inbound Topic` 으로 수신된 메시지는 구현한 `Consumer` 로 소비되고, 
해당 메시지는 애플리케이션 처리를 수행한 후 결과를 `Kafka Transaction` 을 사용하는 방식과 사용하지 않는 방식으로 각 `Outbound Topic` 에 보내진다. 
그리고 최종적으로 `Outbound Topic` 을 구독하는 `Consumer` 가 이를 소비해 메시지가 어떤식으로 전달 됐는지 살펴본다.  

데모 애플리케이션의 전체 코드는 []()
에서 확인 할 수 있다.  

아래 그림은 데모의 구성 요소와 애플리케이션의 동작 과정을 보여준다.  

.. 그림 ..

1. `Inbound Topic` 으로 수신된 메시지는 처리 후 `Outbound Topic 1` 로 메시지를 전송한다. 
2. `Third party service` 의 `REST` 호출을 수행한다. 
3. `Outbound Topic 2` 로 메시지를 전송한다. 

데모는 위 과정 중 `REST` 호출이 이뤄지는 부분을 `Wiremock` 통해 성공/실패를 제어하는 방식으로 
성공과 실패 과정 그리고 `Kafka Transaction` 적용 여부 및 `READ_COMMITTED`, `READ_UNCOMMITTED` 소비 방식에 따른 결과를 살펴볼 것이다.  


