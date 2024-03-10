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

### How to config
우리는 위에서 설명한 `Conumer` 가 구독한 이후 메시지 폴링을 `Partition` 의 어느 곳부터 할지에 대해서는 
`auto.offset.reset` 에 아래와 같은 설정값을 지정해서 가능하다.  

value|desc
---|---
earliest|`offset` 을 가장 앞선 `offset` 인 `Partition` 의 시작 부분부터 사용하도록 재설정한다. 
latest|`offset` 을 가장 최신 `offset` 인 `Partition` 의 끝 부분부터 사용하도록 재설정한다. (기본값)
none|`Consumer Group` 에 대한 `offset` 이 없다면 예외가 발생한다. 

`Consumer Group` 의 `offset` 이 이미 주어진 있는 상황에서 위 설정 값들은 사용되지 않는다. 
위 설정 값들은 `Consumer Group` 중 특정 `Consumer` 가 멈추고 재시작 됐을 때, 
해당 `Consumer` 가 어느 부분부터 메시지를 소비해야할지 결정할 때 사용된다.  

### Earliest

.. 그림 ..

`auto.offset.resest: earliest` 로 설정한 경우 새롭게 시작된 `Consumer` 는 
자신에게 할당된 `Partition` 에 있는 모든 메시지를 처음부터 차례대로 소비하게 된다. 
위 그림을 보면 `Consumer` 가  `Partition` 의 시작부분에 있는 `message 1` 부터 소비하는 것을 알 수 있다.  

만약 `Partition` 에 수백만 개의 메시지가 있다면 전체 시스템에 큰 부하를 초래할 수 있으므로 `Partition` 의 볼륨을 잘 이해하고 해당 값을 설정해야 한다. 
`Partition` 의 데이터는 보존 용량/기간에 따라 오래된 메시지도 보관이 돼있을 수 있어 시스템을 특정 시점으로 되돌리거나 하는 상황에서 유용하게 사용될 수 있다. 
하지만 `retention.ms : -1` 철럼 설정된 경우에는 시스템 시작 시점부터 생성된 모든 메시지가 새로운 `Consumer` 가 구독을 시작 할때마다 소비 될 수 있음을 의미한다.  


### Latest

.. 그림 ..

`auto.offset.reset: latest` 로 설정 됐다면 새로운 `Consumer` 가 `Partition` 을 구독 한 이후 메시지부터 소비한다. 
위 그름을 보면 `offset` 이 3인 시점 부터 새로운 `Consumer` 가 `Partition` 구독을 시작 했으므로 `offset` 이 1과 2의 메시지는 무시하고, 
`offset 3` 부터 메시지를 소비하게 된다.  

즉 `latest` 는 기존 메시지는 소비하지 않고 스킵하는 설정인데 이는 애플리케이션의 요구사항을 잘 고려한 후 설정해야 한다.  
