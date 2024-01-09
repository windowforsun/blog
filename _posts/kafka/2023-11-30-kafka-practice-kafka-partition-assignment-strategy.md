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


### RoundRobinAssignor
`RoundRobinAssignor` 는 이름 그대로 `Round-Robin` 방식으로 `Parition` 을 할당하는 전략이다. 
`Consumer Group` 내 `Consumer` 들에게 토픽의 파티션을 균등하고 순차적으로 할당하는 방식으로, 
주요한 목적은 작업 부하를 `Consumer Group` 의 모든 `Consumer` 에게 균등하게 분산시키는 것이다.  

균등분배 방식으로 토픽의 파티션을 모든 `Consumer` 에게 순차적으로 할당한다. 
각 `Consumer` 가 동일한 수의 파티션을 가질 수 있도록 하는데 목적이 있다.  

그리고 순차적 할당방식을 통해 `Consumer A` 가 `Partition 1` 을 할당 받았다면, 
`Consumer B` 는 `Partition 2` 를 `Consumer C` 는 `Partition 3` 를 할당 받는다. 
이러한 방식으로 모든 컨슈머에 파티션이 할당 될떄 까지 반복한다.  

이러한 `RoundRobin` 방식은 부하를 각 `Consumer` 에게 균등하게 분산하고, 
간단한 방식이라는 점이 시스템 관리에 용이함을 가져 올 수 있다.  

하지만 모든 파티션의 데이터가 균등하지 않는 경우에는 부하가 균등하지 않고 특정 `Consumer` 에게 부하가 집중 될 수 있다. 
또한 `Consumer Group` 의 멤버가 변경 될 때마다 `Rebalancing` 이 발생 하게 된다. 
이는 변경된 `Consumer Group` 에 맞게 다시 균등 분배를 하기 위함 인데, 
이로 인한 시스템에 부하가 발생 할 수 있기에 주의해야 한다.  


#### 테스트
`RangeAssignor` 에서 생성한 `strategy-topic-1` 과 `strategy-topic-2` 를 동일하게 사용한다. 
`RoundRobin` 방식을 사용했을 때는 어떤 식으로 할당되는지 확인하기 위해, 
`Consumer Group` 은 `roundrobin-strategy-group`, `Client Id` 는 `roundrobin-strategy-client-1` 로 
두 토픽을 구독하도록 한다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group roundrobin-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor \
--consumer-property client.id=roundrobin-consumer-client-1

```  

그리고 `roundrobin-strategy-group` 의 상세 정보를 조회해서 구독 토픽의 파티션과 `Consumer` 의 할당 현황을 살펴보면 아래와 같다.  

```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group roundrobin-strategy-group --describe

GROUP                     TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                       HOST            CLIENT-ID
roundrobin-strategy-group strategy-topic-1 0          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 1          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 2          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 3          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 4          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 0          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 1          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 2          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 3          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 4          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
```  

단일 클라이언트인 경우 2개의 토픽 모든 파티션이 `roundrobin-strategy-client-1` 에 할당된 것을 확인 할 수 있다. 
동일한 조건으로 `roundrobin-strategy-client-2` 를 추가로 구독 시키고 다시 할당 현황을 살펴보면 아래와 같다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group roundrobin-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor \
--consumer-property client.id=roundrobin-consumer-client-2

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group roundrobin-strategy-group --describe

GROUP                     TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                       HOST            CLIENT-ID
roundrobin-strategy-group strategy-topic-1 0          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 1          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-1 2          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 3          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-1 4          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 0          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-2 1          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 2          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-2 3          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 4          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
```  

각 토픽당 `Consumer` 를 순차적으로 할당하기 때문에, 
`strategy-topic-1` 은 `roundrobin-consumer-client-1` 이 먼저 할당 받고, 
`strategy-topic-2` 은 `roundrobin-consumer-client-2` 이 먼저 할당 받았기 때문에 아래와 같이 정리해 볼 수 있다.  


| Topic            | Partition |Consumer|Rebalancing
|------------------|-----------|---|---
| strategy-topic-1 | 0         |roundrobin-consumer-client-1|X
|                  | 1         |roundrobin-consumer-client-2|O
|                  | 2         |roundrobin-consumer-client-1|X
|                  | 3         |roundrobin-consumer-client-2|O
|                  | 4         |roundrobin-consumer-client-1|X
| strategy-topic-2 | 0         |roundrobin-consumer-client-2|O
|                  | 1         |roundrobin-consumer-client-1|X
|                  | 2         |roundrobin-consumer-client-2|O
|                  | 3         |roundrobin-consumer-client-1|X
|                  | 4         |roundrobin-consumer-client-2|O

마지막으로 `roundrobin-strategy-client-3` 을 동일한 조건으로 추가한 후 현황을 보면 아래와 같다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group roundrobin-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor \
--consumer-property client.id=roundrobin-consumer-client-3

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group roundrobin-strategy-group --describe

GROUP                     TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                       HOST            CLIENT-ID
roundrobin-strategy-group strategy-topic-1 0          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 3          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 1          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-2 4          0               0               0               roundrobin-consumer-client-1-8f2ccc9e-6ad4-4615-bdea-4090d40df99b /172.23.0.3     roundrobin-consumer-client-1
roundrobin-strategy-group strategy-topic-1 1          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-2 2          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-1 4          0               0               0               roundrobin-consumer-client-2-34c2c497-1b86-4fbb-9a0c-d982566ebafd /172.23.0.3     roundrobin-consumer-client-2
roundrobin-strategy-group strategy-topic-1 2          0               0               0               roundrobin-consumer-client-3-3df29831-6d8c-4844-a762-23a3a892062c /172.23.0.3     roundrobin-consumer-client-3
roundrobin-strategy-group strategy-topic-2 0          0               0               0               roundrobin-consumer-client-3-3df29831-6d8c-4844-a762-23a3a892062c /172.23.0.3     roundrobin-consumer-client-3
roundrobin-strategy-group strategy-topic-2 3          0               0               0               roundrobin-consumer-client-3-3df29831-6d8c-4844-a762-23a3a892062c /172.23.0.3     roundrobin-consumer-client-3
```  

`strategy-topic-1` 은 `roundrobin-strategy-client-1` 에서 3번 순으로 할당 받고,
`strategy-topic-2` 는 `roundrobin-strategy-client-3` 에서 1번 순으로 할당 할당 받았다는 것을 인지하면 아래와 할당 결과를 토픽별 순서대로 정리하면 아래와 같다.  

| Topic            | Partition |Consumer|Rebalcning
|------------------|-----------|---|---
| strategy-topic-1 | 0         |roundrobin-consumer-client-1|X
|                  | 1         |roundrobin-consumer-client-2|X
|                  | 2         |roundrobin-consumer-client-3|O
|                  | 3         |roundrobin-consumer-client-1|O
|                  | 4         |roundrobin-consumer-client-2|O
| strategy-topic-2 | 0         |roundrobin-consumer-client-3|O
|                  | 1         |roundrobin-consumer-client-1|X
|                  | 2         |roundrobin-consumer-client-2|X
|                  | 3         |roundrobin-consumer-client-3|O
|                  | 4         |roundrobin-consumer-client-1|O

결과적으로 보면 각 `Consumer`를 기준으로 본다면 동일한 파티션 수를 할당 받았지만, 
`Consumer` 가 추가될 때마다 매번 절반정도의 파티션에서 `Reblancing` 이 발생한 것을 확인 할 수 있다.    


### StickyAssignor
`StickyAssignor` 는 `Consumer Group` 의 `Rebalancing` 중에 현재 파티션 할당을 최대한 유지하는 전략이다. 
`Rebalancing` 을 최소화하기 때문에 `Consumer` 가 `Rebalancing` 이후 새로운 파티션을 할당 받고 초기화 하는 작업을 줄여, 
전체적인 효율을 향상시킬 수 있다.  

기존 파티션 할당을 유지하며 부하 균형을 맞추는 방식으로 이뤄지는데, 
이로 인해 `Consumer` 는 `Rebalancing` 이후에도 동일한 파티션을 계속 처리할 가능성이 높아지게 된다. 
초기 할당은 `RoundRobin` 과 비슷하지만 재할당의 경우 전체를 재할당 하는 것이 아니라 기존 할당 유지와 부하 분산을 모두 고려해서 동작이 수행된다.  


#### 테스트
토픽은 앞서 테스트에 사용한 토픽과 동일한 것을 사용한다. 
`Sticky` 방식을 사용했을 때는 어떤 식으로 할당되는지 확인하기 위해,
`Consumer Group` 은 `sticky-strategy-group`, `Client Id` 는 `sticky-strategy-client-1` 로
두 토픽을 구독하도록 한다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group sticky-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor \
--consumer-property client.id=sticky-consumer-client-1

```  

그리고 `sticky-strategy-group` 의 상세 정보를 조회해서 구독 토픽의 파티션과 `Consumer` 의 할당 현황을 살펴보면 아래와 같다.


```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group sticky-strategy-group --describe

GROUP                 TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                   HOST            CLIENT-ID
sticky-strategy-group strategy-topic-1 0          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-1 1          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-1 2          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-1 3          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-1 4          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 0          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 1          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 2          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 3          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 4          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
```  

단일 클라이언트인 경우 2개의 토픽 모든 파티션이 `sticky-strategy-client-1` 에 할당된 것을 확인 할 수 있다.
동일한 조건으로 `sticky-strategy-client-2` 를 추가로 구독 시키고 다시 할당 현황을 살펴보면 아래와 같다.

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group sticky-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor \
--consumer-property client.id=sticky-consumer-client-2

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group sticky-strategy-group --describe

GROUP                 TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                   HOST            CLIENT-ID
sticky-strategy-group strategy-topic-1 0          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 1          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 2          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 3          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 4          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-2 0          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 1          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 2          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 3          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 4          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
```  

`RoundRobin` 과 달리 토픽 `strategy-topic-1` 은 `sticky-consumer-client-2` 에 할당되고, 
`strategy-topic-2` 는 `sticky-consumer-client-1` 에 핼당 된 것을 확인 할 수 있다. 
`Rebalancing` 은 `strategy-topic-1` 의 토픽에 대해서만 발생한 것을 확인 할 수 있다.  

| Topic            | Partition | Consumer                 |Rebalcning
|------------------|-----------|--------------------------|---
| strategy-topic-1 | 0         | sticky-consumer-client-2 |O
|                  | 1         | sticky-consumer-client-2 |O
|                  | 2         | sticky-consumer-client-2 |O
|                  | 3         | sticky-consumer-client-2 |O
|                  | 4         | sticky-consumer-client-2 |O
| strategy-topic-2 | 0         | sticky-consumer-client-1 |X
|                  | 1         | sticky-consumer-client-1 |X
|                  | 2         | sticky-consumer-client-1 |X
|                  | 3         | sticky-consumer-client-1 |X
|                  | 4         | sticky-consumer-client-1 |X



마지막으로 `sticky-strategy-client-3` 을 동일한 조건으로 추가한 후 현황을 보면 아래와 같다.  

```bash
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--group sticky-strategy-group  \
--whitelist 'strategy-topic-1|strategy-topic-2' \
--consumer-property partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor \
--consumer-property client.id=sticky-consumer-client-3

$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group sticky-strategy-group --describe

GROUP                 TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                   HOST            CLIENT-ID
sticky-strategy-group strategy-topic-1 0          0               0               0               sticky-consumer-client-3-496f069f-baf0-402d-ba70-02b50017c1b8 /172.23.0.3     sticky-consumer-client-3
sticky-strategy-group strategy-topic-1 1          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 2          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 3          0               0               0               sticky-consumer-client-2-a343d9d3-a719-44a7-8529-9afb06a39458 /172.23.0.3     sticky-consumer-client-2
sticky-strategy-group strategy-topic-1 4          0               0               0               sticky-consumer-client-3-496f069f-baf0-402d-ba70-02b50017c1b8 /172.23.0.3     sticky-consumer-client-3
sticky-strategy-group strategy-topic-2 0          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 1          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 2          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 3          0               0               0               sticky-consumer-client-1-6f007c57-9952-40de-a1c2-388f611935e8 /172.23.0.3     sticky-consumer-client-1
sticky-strategy-group strategy-topic-2 4          0               0               0               sticky-consumer-client-3-496f069f-baf0-402d-ba70-02b50017c1b8 /172.23.0.3     sticky-consumer-client-3
```  

| Topic            | Partition | Consumer                 |Rebalcning
|------------------|-----------|--------------------------|---
| strategy-topic-1 | 0         | sticky-consumer-client-3 |O
|                  | 1         | sticky-consumer-client-2 |X
|                  | 2         | sticky-consumer-client-2 |X
|                  | 3         | sticky-consumer-client-2 |X
|                  | 4         | sticky-consumer-client-3 |O
| strategy-topic-2 | 0         | sticky-consumer-client-1 |X
|                  | 1         | sticky-consumer-client-1 |X
|                  | 2         | sticky-consumer-client-1 |X
|                  | 3         | sticky-consumer-client-1 |X
|                  | 4         | sticky-consumer-client-3 |O

마지막 `Consumer` 를 추가했을 때로 비교하면 앞선 `Roundrobin` 에 비해 `Rebalancing` 이 발생하는 파티션이 절반정도 줄어든 것을 확인 할 수 있다. 
하지만 `sticky-consumer-client-1` 은 `strategy-topic-1` 을 할당 받지 못했고, 
`sticky-consumer-client-2` 는 `strategy-topic-2` 를 할당 받지 못한 것처럼 의도와 달리 `Consumer` 하나가 2개의 토픽을 모두 구독하지 않는 증상이 발생 할 수 있으므로 주의가 필요하다.  



### CooperativeStickyAssignor
`CoorativeStickyAssignor` 는 `StickyAssignor` 의 개선 버전으로 `Rebalancing` 과정을 `협력적` 으로 만들어 `Consumer Group` 내의 파티션 재할당을 점진적으로 수행한다. 
`StickyAssignor` 처럼 기존 파티션 할당을 유지하려 하지만, `Rebalancing` 이 필요한 경우 한 번에 모든 파티션을 재할당하지 않고, 
점진적으로 파티션을 재할당한다. 
이러한 동작으로 `Rebalancing` 시 발생할 수 있는 영향을 최소화하고 `Consumer Group` 의 안전성과 처리 성능 향상을 가져올 수 있다. 
그리고 점진적이라는 특성으로 `Rebalancing` 과정에서 파티션의 일부만 잃거나 얻기 때문에 전체 시스템의 부하를 줄이고 데이터 처리의 연속성을 유지할 수 있다.  

여기서 점진적인 리밸런싱이란 리밸런싱 동안에도 일부 컨슈머가 계속해서 파티션의 데이터를 소비할 수 있음을 의미한다. 
컨슈머 입장에서는 컨슈머가 데이터 소비를 중단하지 않고 새로운 파티션 할당을 수용 할 수 있음을 의미한다.  

점진적 리밸런싱에 대해 더 자세하 설명하면 아래와 같다. 
- `Rebalancing` 과정은 아래 2단계로 설명할 수 있다. 
- 1단계: 준비
  - 초기 리밸런싱 : `Consumer Group` 에 `Consumer` 추가/삭제와 같은 변경이 감지되면 리밸런싱 프로세스가 시작된다. 
  - 현재 상태 파악 : 각 `Consumer` 가 구독하고 있는 파티션에 대한 정보를 `Group Coodinate` 에게 공유한다. 리밸런싱의 기반 데이터로 구독 파티션, 토픽등의 정보가 있다. 
- 2단계 : 실제 리밸런싱
  - 파티션 재할당 : 새로운 파티션 할당 계획인 수립되면, 점진적으로 적용된다. 이는 기존 할당을 가능한 유지하며 필요한 재할당만 수행하는 방식이다. 
  - 새할당 적용 : `Consumer` 가 새로 할당된 파티션에 데이터를 소비할 수 있도록 한다. 이는 기존 할당을 유지하며 새로운 할당을 받아 들이는 방식이기 때문에 기존 파티션 중 일부는 계속 소비하게 될 수도 있꼬, 일부는 다른 컨슈머에게 넘어갈 수도 있다. 

`CooperativeStickyAssignor` 는 `Kafka Rebalancing Protocol` 중 `Incremental Rebalancing` 을 사용함을 기억하자. 

#### 테스트
`CooperativeSticky` 테스트의 파티션 할당 결과는 `Sticky` 의 결과와 동일하기 하다. 
그러므로 간단하게 출력 결과만 정리한다.  

- 첫 번째 `Consumer` 생성

```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group cooperative-sticky-strategy-group --describe

GROUP                             TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                               HOST            CLIENT-ID
cooperative-sticky-strategy-group strategy-topic-1 0          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 0          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 3          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 4          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 3          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 2          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 1          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 2          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 4          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 1          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
```  

- 두 번째 `Consumer` 생성


```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group cooperative-sticky-strategy-group --describe

GROUP                             TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                               HOST            CLIENT-ID
cooperative-sticky-strategy-group strategy-topic-1 0          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 1          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 2          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 3          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 4          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-2 0          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 1          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 2          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 3          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 4          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
```  

| Topic            | Partition | Consumer                 |Rebalcning
|------------------|-----------|--------------------------|---
| strategy-topic-1 | 0         | sticky-consumer-client-1 |X
|                  | 1         | sticky-consumer-client-1 |X
|                  | 2         | sticky-consumer-client-1 |X
|                  | 3         | sticky-consumer-client-1 |X
|                  | 4         | sticky-consumer-client-1 |X
| strategy-topic-2 | 0         | sticky-consumer-client-2 |O
|                  | 1         | sticky-consumer-client-2 |O
|                  | 2         | sticky-consumer-client-2 |O
|                  | 3         | sticky-consumer-client-2 |O
|                  | 4         | sticky-consumer-client-2 |O

- 세 번째 `Consumer` 생성


```bash
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group cooperative-sticky-strategy-group --describe

GROUP                             TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                               HOST            CLIENT-ID
cooperative-sticky-strategy-group strategy-topic-1 0          0               0               0               cooperative-sticky-consumer-client-3-589262ae-7d58-490b-ac0a-0a217f586334 /172.23.0.3     cooperative-sticky-consumer-client-3
cooperative-sticky-strategy-group strategy-topic-1 1          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 2          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 3          0               0               0               cooperative-sticky-consumer-client-1-4c648dc3-d5a2-42b3-bd81-23ef169f1fdb /172.23.0.3     cooperative-sticky-consumer-client-1
cooperative-sticky-strategy-group strategy-topic-1 4          0               0               0               cooperative-sticky-consumer-client-3-589262ae-7d58-490b-ac0a-0a217f586334 /172.23.0.3     cooperative-sticky-consumer-client-3
cooperative-sticky-strategy-group strategy-topic-2 0          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 1          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 2          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 3          0               0               0               cooperative-sticky-consumer-client-2-2054cda0-94a9-4c09-897d-2f7e5c4f73d7 /172.23.0.3     cooperative-sticky-consumer-client-2
cooperative-sticky-strategy-group strategy-topic-2 4          0               0               0               cooperative-sticky-consumer-client-3-589262ae-7d58-490b-ac0a-0a217f586334 /172.23.0.3     cooperative-sticky-consumer-client-3
```

| Topic            | Partition | Consumer                 |Rebalcning
|------------------|-----------|--------------------------|---
| strategy-topic-1 | 0         | sticky-consumer-client-3 |O
|                  | 1         | sticky-consumer-client-1 |X
|                  | 2         | sticky-consumer-client-1 |X
|                  | 3         | sticky-consumer-client-1 |X
|                  | 4         | sticky-consumer-client-3 |O
| strategy-topic-2 | 0         | sticky-consumer-client-2 |X
|                  | 1         | sticky-consumer-client-2 |X
|                  | 2         | sticky-consumer-client-2 |X
|                  | 3         | sticky-consumer-client-2 |X
|                  | 4         | sticky-consumer-client-3 |O



---  
## Reference
[partition.assignment.strategy](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#partition-assignment-strategy)  
[Rebalance & Partition Assignment Strategies in Kafka](https://medium.com/trendyol-tech/rebalance-and-partition-assignment-strategies-for-kafka-consumers-f50573e49609)  
[Kafka Partition Assignment Strategy](https://www.conduktor.io/blog/kafka-partition-assignment-strategy/)  
[RangeAssignor](https://kafka.apache.org/36/javadoc/org/apache/kafka/clients/consumer/RangeAssignor.html)  



