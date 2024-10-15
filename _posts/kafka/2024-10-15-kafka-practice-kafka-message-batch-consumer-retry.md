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

## Kafka Batch Consume Retry
`Kafka Consumer` 는 `Kafka Broker` 로 부터 `Batch` 성으로 다수의 메시지를 한번에 받을 수 있다. 
이런 `Batch` 성 작업을 수행할 때 중요하게 고려돼야 하는 것이 바로 실패에 대한 시나리오이다. 
`Kafka Library` 가 이런 상황에서 어떠한 식으로 동작하는지에 대해 알아보고자 한다.  

`Kafka Consumer` 는 하나 이상의 `Topic` 혹은 하나 이상의 `Partition` 을 할당 받게 된다. 
이는 `Kafka Consumer` 는 주로 단일 `Topic` 을 구독해 처리하지만, 다수의 `Topic` 을 구독해 처리할 수 있다는 의미이고 
그 수는 제한되지 않는다. 
각 시나리오 설명을 위해 `Topic` 에서 `Batch` 로 메시지를 소비할 때 3개씩 받아 처리한다고 가정한다. 
그리고 명시적인 언급이 없다면 이후 시나리오에서 [enable.auto.commit](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#enable-auto-commit)
은 `true` 로 활성화 돼있다고 가정한다. 
`enable.auto.commit` 이 `false` 로 비성화 돼있다면 `Consumer` 의 `offset` 관리는 코드에서 직접 수행해야 한다.  

[max.poll.records](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-records)
와
[max.poll.interval.ms](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-interval-ms)
에서 보면 알 수 있듯이 `Kafka Consumer` 는 기본적으로 `batch` 방식, 
즉 `poll` 을 수행 했을 때 단일 메시지를 소비하는 것이 아니라 `N` 개의 메시지 소비한다는 점을 기억해야 한다.  

그리고 관련해서 혼동이 있을 수 있는데, 포스팅에서 다루는 내용은 `Kafka Consumer` 의 `max.poll.records` 와 연관된 `batch` 방식과 관련있다. 
`Spring Kafka` 를 사용해서 `Kafka Consumer` 관련 빈을 구성 할 때, `ConcurrentKafkaListenerContainerFactory` 에서 설정하는 `setBatchListener(true)` 와는 시나리오 별 차이가 있다는 점도 기억해야 한다.  

