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


## Kafka Replication
`Kafka` 는 분산 메시징 시스템으로 고가용성과 내구성을 제공한다. 
이러한 `Kafka` 의 특성은 `Broker Cluster` 간의 `Replication` 이 있기 때문에 보장될 수 있다. 
특정 `Broker` 가 실패하더라도 사용 중인 `Topic Partition` 의 데이터는 유실되지 않고, 
복제된 `Broker` 의 `Topic Partition` 의 데이터로 지속적인 서비스가 가능하다. 
`Replication` 동작은 말그대로 복제이기 때문에 약간의 지연시간이 증가할 수 있다. 
하지만 복제 수준 설정이 가능하기 때문에 서비스 니즈에 맞는 설정해주어야 한다. 
이러한 설정은 `Producer` 가 `acks` 를 수신할 최소 `Replication` 개수인 `Min In-Sync Replica(Min ISR)` 이라고 한다.  

