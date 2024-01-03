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
    - Rebalancing
    - Partition
    - Consumer Group
    - Consumer
    - Topic
toc: true
use_math: true
---  

## Partition Assignment Strategy
`Partition Assignment Strategy` 는 `Consumer Group` 를 사용 할떄
클라이언트가 `Consumer Instance` 간 토픽의 `Parition` 을 분산하는데 
사용할 할당 전략을 의미한다.  

`Parition Assignment Strategy` 는 `Consumer` 의 설정 프로퍼티 중 
[partition.assignment.strategy](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#partition-assignment-strategy)
를 사용해서 설정 할 수 있다.  

총 5가지 전략이 있는데 간략하게 설명하면 아래와 같다.  

Strategy|Desc
---|---
org.apache.kafka.clients.consumer.RangeAssignor(default)|기본값으로 `Topic` 별로 `Partition` 을 할당한다. 
org.apache.kafka.clients.consumer.RoundRobinAssignor|`Round-Robin` 방식을 사용해서 `Consumer` 에게 `Partition` 을 할당한다. 
org.apache.kafka.clients.consumer.StickyAssignor|기존 `Partition` 할당 상태를 최대한 많이 유지하면서 최대로 균형 잡힌 할당을 보장한다. 
org.apache.kafka.clients.consumer.CooperativeStickyAssignor|동일한 `StickyAssignor` 동일한 논리를 바탕으로 하지만, 보다 협력적으로 `Rebalancing` 이 허용된다. 
org.apache.kafka.clients.consumer.ConsumerPartitionAssignor|인터페이스 구현을 통해 사용자가 직접 구현한 전략을 사용 할 수 있다. 


이러한 할당 전략이 필요한 이유는 아래와 같다. 

- `Load Balancing`(부하 분산) : 컨슈머 그룹에 있는 모든 컨슈머가 균등하게 작업을 분담해 부하가 분산 될 수 있도록 한다. 
- `High Availabiltity`(고가용성) : 특정 컨슈머가 가용 불가 상태에 빠지더라도 다른 컨슈머가 해당 파티션의 메시지를 이어서 처리 할 수 있다. 
- `Scalability`(확장성) : 큰슈머 그룹에 컨슈머를 추가하거나 제거할 떄 파티선 데이터를 담당하는 컨슈머를 효율적으로 재분배 할 수 있다. 

일반적인 상황에서는 기본 전략인 `RangeAssignor` 를 사용해도 무방하지만 별도의 전략을 사용해야하는 상황이 있다. 

- 부하 균등화 : 컨슈머 간 작업 부하를 균등하게 분산하고자 할때 `RoundRobinAssignor` 과 같은 전략을 고민해 볼 수 있다. 
- 처리 순서 : 처리 순서를 유지해야 하는 경우, `RangeAssignor` 과 같은 전략을 사용해 연속적으로 파티션 범위를 할당할 수 있다. 
- 리밸런싱 : 적절한 전략 사용을 통해 리밸런싱을 최소화하면서 컨슈머 그룹의 안정성까지 유지 할 수 있다. 



