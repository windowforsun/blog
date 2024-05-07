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
