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

### RangeAssignor
`RangeAssignor` 는 토픽마다 `Partition` 순서대로 `Consumer` 를 할당하는 전략이다. 
토픽 파티션들을 `Consumer Group` 의 컨슈머들에게 연속적인 범위로 할당하는 방식이다. 
주요한 목적은 `Consumer` 들이 파티션을 가능한 공평하게 분배하는 것이다.  

연속적인 범위 할당은 각 `Consumer` 가 하나 이상의 파티션을 연속적인 범위로 할당 받는 것을 의미한다. 
여기에서 연속적인 범위는 파티션 번호 순서를 의미한다.  

결과적으로 토픽의 파티션을 `Consumer` 수에 나눠 각 `Consumer` 에게 연속적인 파티션을 할당하게 된다. 
10개의 파티션과 3개의 `Consumer` 가 있다면, `Consumer A` 는 `Partition 0~3`, 
`Consumer B` 는 `Partition 4~6` 을 `Consumer C` 는 `Partition 7~9` 를 할당 받게 된다. 

파티션의 수보다 `Consumer` 의 수가 더 많은 경우 파티션을 할당 받지 못한 `Consumer` 가 존재 할 수 있음을 알아야 한다. 
그리고 각 파티션의 데이터가 균등하지 않는 경우 특정 `Consumer` 에 부하가 몰릴 수 있고, 
`Consumer Group` 의 멤버가 변경될 때마다 새로운 `Consumer` 수에 맞는 파티션 순차적 분배를 위해 `Rebalancing` 이 발생하게 된다.

#### 테스트
테스트를 위해서 `strategy-topic-1`, `strategy-topic-2` 2개의 토픽을 각 파티션 5개쌕 구성하고, 
`range-strategy-client-1 ~ 3` 이라는 이름으로 3개의 `Consumer` 를 생성해서 테스트를 진행한다.   

```bash
$ kafka-topics.sh --bootstrap-server localhost:9092 --create --partitions 5 --topic strategy-topic-1
Created topic strategy-topic-1.
$ kafka-topics.sh --bootstrap-server localhost:9092 --create --partitions 5 --topic strategy-topic-2
Created topic strategy-topic-2.
```  

그리고 `range-strategy-group` 이름으로 `Consumer Group` 을 생성해서 `strategy-topic-1, strategy-topic-2` 를 구독한다. 
`Client Id` 는 구분을 위해 `range-strategy-client-1` 로 한다.

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group range-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RangeAssignor \
--consumer-property client.id=range-consumer-client-1
```  

그리고 `range-strategy-group` 의 상세 정보를 조회해서 토픽의 파티션과 `Consumer` 의 분배현황을 살펴보면 아래와 같다.  

```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group range-strategy-group --describe

GROUP                TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                  HOST            CLIENT-ID
range-strategy-group strategy-topic-1 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 2          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 3          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 4          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 2          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 3          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 4          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
```  

위의 결과처럼 2개 토픽의 모든 10개의 파티션이 `range-strategy-client-1` 에 할당된 것을 확인 할 수 있다. 
이 상태에서 `range-strategy-client-2` 를 동일한 조건으로 추가하고나서 할당 현황을 다시 살펴본다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group range-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RangeAssignor \
--consumer-property client.id=range-consumer-client-2

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group range-strategy-group --describe

GROUP                TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                  HOST            CLIENT-ID
range-strategy-group strategy-topic-1 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 2          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 2          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 3          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-1 4          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-2 3          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-2 4          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
```  

`range-strategy-client-1` 에는 `strategy-topic-1,2 토픽의 파티션 0~2` 가 할당되고, 
`range-strategy-client-2` 에는 `strategy-topic-2,2 토픽의 파티션 3~4` 가 할당 된 것을 확인 할 수 있다.  

마지막으로 `range-strategy-client-3` 를 동일한 조건으로 추가한 후 할당 현황을 살펴본다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group range-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RangeAssignor \
--consumer-property client.id=range-consumer-client-3

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group range-strategy-group --describe

GROUP                TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                  HOST            CLIENT-ID
range-strategy-group strategy-topic-1 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 0          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-2 1          0               0               0               range-consumer-client-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     range-consumer-client-1
range-strategy-group strategy-topic-1 2          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-1 3          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-2 2          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-2 3          0               0               0               range-consumer-client-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     range-consumer-client-2
range-strategy-group strategy-topic-1 4          0               0               0               range-consumer-client-3-d61c357b-eeb8-4600-9815-ee767ca6fe8a /172.23.0.3     range-consumer-client-3
range-strategy-group strategy-topic-2 4          0               0               0               range-consumer-client-3-d61c357b-eeb8-4600-9815-ee767ca6fe8a /172.23.0.3     range-consumer-client-3
```  

최종적으로 할당된 `Consumer` 와 파티션의 현황을 정리하면 아래와 같은데, 
파티션의 연속적인 범위를 기반으로 `Consumer` 와 할당 된 것을 확인 할 수 있다.  

| Consumer                | Topic             |Partition
|-------------------------|-------------------|---
| range-consumer-client-1 | strategy-topic-1  |0
|                         |                   |1
|                         | strategy-topic-2  |0
|                         |                   |1
| range-consumer-client-2 | strategy-topic-1  |2
|                         |                   |3
|                         | strategy-topic-2  |2
|                         |                   |3
| range-consumer-client-3 | strategy-topic-1  |4
|                         | strategy-topic-2  |4

