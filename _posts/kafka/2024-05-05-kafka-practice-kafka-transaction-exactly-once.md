--- 
layout: single
classes: wide
title: "[Kafka] Kafka Transaction Exactly Once"
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

## Kafka Transaction Exactly-Once
`Kafka` 는 `Transacitonal API` 을 통해 `exactly-once` 메시징을 제공한다. 
이러한 방법론을 실제 애플리케이션에 적용하기 전에 `exactly-oncd` 가 어떤 것을 의미하고, 
`Kafka Transaction` 이 어떻게 동작하는지, 적용전 고려사항에는 어떠한 것들이 있는지에 대해 알아보고자 한다. 
그리고 추가적으로 `Spring Boot` 기반 애플리케이션을 구현 할때 
`Kafka Transaction` 을 어떠한 구성을 통해 적용할 수 있는지에 대해서도 알아본다.  

`Kafka Transaction` 은 `Kafka` 의 `Exactly-Once` 를 구현하는 주요기능이다. 
하지만 이 기능의 애플리케이션에 대한 적합성은 요구사항, 구성에 따라 달리질 수 있다. 
`Kafka` 관점에서 메시지는 `Exactly-Once` 로 동작할 수 있지만, 
애플리케이션에서 수행하는 `DB`, `REST API` 호출 등에는 중복 처리가 발생 할 수 있다는 의미이다. 
또한 `Kafka Transaction` 을 통해 메시지를 발생하는 `Topic` 을 사용하는 `downstream` 
애플리케이션도 어떤 `isolation.level` 아냐에 따라 커밋 전까지 여러번 메시지를 작성 할 수 있기 때문에 
중복 메시지를 받을 수도 있다는 점도 인지가 필요하다.  

### Exactly-Once
`Kafka Broker` 에서 메시지를 소비하는 `Consumer` 는 `at-least-once` 특성으로 메시지를 소비하기 때문에 최소 한번 메시지 소비를 보장 받을 수 있다. 
일시적인 실패 시나리오에서 메시지는 재전송되어 손실이 발생하지 않기 때문이다. 
하지만 이는 최소 한번은 메시지를 소비할 수 있지만 재전송 과정에서 중복 메시지가 발생할 수 있는 상황이 있다. 
이러한 중복 메시지에 대한 처리는 [Idempotent Consumer]()
를 통해 처리가 필요하다. (하지만 이는 추가적인 오버헤드가 있다.)  

`Kafka` 에서 `exactly-once` 는 여러 단계의 처리 과정이 정확히 한번 수행된다는 것을 의미한다. 
이는 `Inbound Topic` 에서 메시지가 한번 소비되고, 가공 후 `Outbound Topic` 에 그 결과를 한번만 쓴다는 의미이다. 
즉 `Outound Topic` 에서 결과 메시지를 받아 이어서 처리하는 `downstream` 은 결과를 단 한번만 받는 것이 보장된다. 
메시지 처리 과정에서 재전송 등의 동작은 발생 할 수 있어 `Inbound Topic` 에서 메시지가 여러번 소비 될 수는 있지만, 
결과가 쓰여지는 `Outbound Topic` 에 정확히 한번 결과가 쓰여지고 중복 이벤트가 발생하는 결과를 방지 할 수 있다.  

.. 그림 ..

