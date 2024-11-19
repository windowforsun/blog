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

## Spring Kafka Consumer Producer
[Spring Kafka](https://spring.io/projects/spring-kafka)
는 `Kafka Consumer` 와 `Kafka Producer API` 를 추상화한 구현체이므로 이를 사용해서 `Kafka Cluster` 
토픽에 메시지를 읽고 쓸 수 있다. 
사용법은 일반적인 `Spring` 프로젝트와 동일하게 비지니스 로직에는 큰 영향 없이 `Annotation` 을 
기반으로 설정과 빈 주입이 가능하다.  

`Spring Kafka` 를 사용해서 구현한 애플리케이션을 도식화하면 아래와 같이 `Inbound Topic` 에서 
메시지를 `Consume` 하고 처리 후 `Outbound Topic` 으로 메시지를 `Producd` 하는 형상이다.  

.. 그림 ..

실 서비스에서 안정적인 메시징을 위해서는 메시지 중복 처리, 트랜잭션, 메시지 순서 등 고려할 것들이 많다. 
하지만 이번 포스팅에서는 `Spring Kafka` 를 기반으로 `Kafka Broker` 와 메시지 소비/생산에 대한 기본적인 
부분에 대해서만 초점을 맞춘 내용만 다룬다.  
