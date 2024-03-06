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

## Consumer Auto Offset Rest
`Kafka Consumer` 에서 `auto.offset.reset` 설정은 
`Consumer` 가 `Topic` 의 `Partition` 를 구독 시작하는 시점인 초기 오프셋이 없는 경우에 대한 동작방식을 정의한다. 
해당 설정을 통해 `Consumer Group` 을 구성하는 `Consumer` 들이 구독하는 `Topic` 의 `Partition` 을 시작 부분부터 읽을지, 
끝 부분부터 읽어야 할지 알 수 있다.  

### How Consumer consuming messages
모든 `Kafka Consumer` 는 `Consumer Group` 에 속하고 
어떤 `Consumer Group` 에 속하는지는 `Consumer` 의 설정 중 `group.id` 를 통해 결정된다. 
즉 모든 `Consumer` 는 `Consumer Group` 에 속하고 `Consumer Group` 에는 최소 하나 이상의 `Consumer` 가 포함된다. 
그리고 `Consumer Group` 을 구성하는 `Consumer` 가 메시지를 소비하기 위해서는 `Kafka Broker` 의 `Topic Partition` 할당을 받아야 한다. 
`Consumer Group` 을 기준으로 각 `Partition` 은 하나의 `Consumer` 만 할당되지만 `Consumer` 는 하나 이상의(혹은 모든) `Parition` 을 할당 받을 수 있다. 

`Consumer Group` 과 그 멤버인 `Consumer` 들이 생성되고 각 `Consumer` 들이 메시지를 소비하기 위해 `Partition` 을 할당 받게 된다. 
그리고 `Consumer` 가 메시지 폴링 시작전 `Partition` 의 어느 위치에서 부터 폴링을 할지 초기 시작 지점을 결정해야 한다. 
`Consumer` 에게 특정 위치의 `offset` 부터 시작해라는 설정이 없는 한 `Partition` 의 시작 부분부터 메시지를 읽는 것과 
`Consumer` 가 `Partition` 을 구독한 시점부터 메시지를 읽는 2가지 옵션이 있다.  
