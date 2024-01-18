--- 
layout: single
classes: wide
title: "[Kafka] Kafak Rebalancing 개념"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 에서 Consumer Group 맴버의 변경이 있을 때 수행되는 Rebalancing 에 해대 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Rebalancing
    - Consumer Group
    - Partition
toc: true
use_math: true
---  

## Kafka Rebalancing
`Kafa Rebalancing` 은 `Kafka` 의 주 목적인 `Distributed Streaming Platform` 으로, 
대규모 데이터 스트림을 처리/저장/전송하는 데 있어 `Kafka Cluster` 의 `Consumer Group` 내에서 
`Topic` 의 `Partition` 할당을 동적 조정하는 과정을 의미한다. 
`Consumer Group` 에 `Consumer` 추가나, `Partition` 수 변경 등에 `Rebalancing` 이 수행 될 수 있고, 
이러한 `Rebalancing` 의 동작 목적은 아래와 같다. 

- 부하 분산 : `Consumer Group` 에 `Consumer` 가 추가/삭제 될떄, `Kafka` 는 `Topic` 의 `Partition` 을 `Consumer` 에게 균등하게 할당해 부하를 분산한다. 
- 고가용성 : 특정 `Consumer` 가 실패하거나 제거 될 때, 해당 `Consumer` 가 처리하던 `Partition` 을 다른 가용한 `Consumer` 에게 재할당한다. 이를 통해 스트림 데이터 처리의 연속성과 고가용성을 보장한다. 
- 스케일링 및 유연성 : `Rebalancing` 을 통해 `Kafka Cluster` 는 `Consumer` 추가/삭제에 유연하게 대응가능하고, `Partition` 변경에 대해서도 유연한 대응이 가능하다. 이를 통해 시스템의 수평적 확장을 가능하게 한다.  

### Rebalancing working
`Rebalancing` 이 수행되는 과정을 설명하는데 `Rebalancing Protocol` 중 `Eager Rebalancing` 을 기준으로 간단한 절차만 나열한다. 

1. `Consumer Group` 변화 감지 : `Consumer` 의 가입/탈퇴/실패가 일어나면 감지가 된다. 
2. `Rebalancing` 시작 : `Group Coordinator` 는 `Rebalancing` 시작을 알린다. 이떄 `Consumer` 는 `Partition` 의 메시지 처리를 중단하고 소비를 중단한다.
3. `Offset Commit` : `Consumer` 는 현재 처리중이던 `Partition` 의 메시지 오프셋을 커밋한다. 
4. `Group Leader` 선정 : `Consumer Group` 내의 하나의 `Consumer` 가 `Group Leader` 로 선정된다. 리더는 `Rebalancing` 과저에서 `Partition` 할당을 조정하는 역할을 수행한다. 
5. `Partition` 할당 계획 수립 : `Group Coordinator` 는 변경된 `Consumer Group` 구성에 따라 새로운 `Partition` 할당 계획을 수립한다. 
6. 할당 정보 전파 : 새로운 `Partition` 할당 정보가 각 `Consumer` 들에게 전달된다. 
7. 데이터 처리 재개 : `Consumer` 들은 새로 할당받은 `Partition` 의 마지막으로 커밋된 어프셋으로 부터 메시지 처리를 시작한다. 
8. `Rebalancing` 완료 : 모든 `Consumer` 가 새로운 할당에 따라 정상동작이 되면 종료된다. 


### Rebalancing trigger
`Rebalancing` 이 필요한 상황은 아래와 같다.

- `Consumer` 추가 및 제거 : `Consumer Group` 의 멤버가 변경되는 상황으로 구성원이 변경되면 `Rebalancing` 이 동작 된다. `Consumer` 의 `Hearbeat` 가 실패하는 경우도 포함된다.
- `Consumer` 실패 : 특정 `Consumer` 가 실패하거나 네트워크 등의 문제로 `Consumer Group` 에서 제거되는 경우, 해당 `Consumer` 가 소비하던 `Partition` 데이터의 연속성을 위해 `Rebalancing` 이 수행된다.
- `Consumer` 의 긴 유휴 상태 : `Consumer` 가 긴 유휴 상태에 있다면 실패한 `Consumer` 로 간주하고 `Consumer Group` 에서 제거할 수 있다. 
- `Topic Partition` 변경 : `Kafka Topic` 의 `Partition` 수가 변경되는 경우 `Rebalancing` 이 수행된다.
- `Partition Assignment Strategy` 변경 : `Consumer Group` 에서 사용하는 `Partition Assignment Strategy` 가 변경되면 `Rebalancing` 이 수행된다.
- `Application Trigger` : 특정 애플리케이션에서 의도적으로 `Rebalancing` 을 트리거할 수 있다.


### Rebalancing effects
`Kafka` 에서 `Rebalancing` 은 데이터 처리의 연속성과 고사용성을 유지하기 위해 필수적인 과정이지만, 
일시적으로 아래와 같은 시스템 영향을 일으킬 수 있음에 주의해야 한다. 

1. 데이터 처리 중단 : `Rebalancing` 이 진행되는 동안 `Consumer Group` 의 모든 `Consumer` 는 `Partition` 의 메시지 수신을 중단하고, 새로운 `Partition` 을 할당 받고 인식할 떄까지 이어진다.
2. 메시지 지연 : 위와 같은 데이터 처리 중단으로 인해 `Rebalancing` 이 시작되고 종료되는 동안 실시간 데이터 처리나 스트림 처리에는 영향을 줄 수 있다. 
3. 부하 증가 : `Rebalancing` 이 완료되고 나서 `Consumer` 들은 새로 할당 받은 `Partition` 의 마지막 커밋 오프셋에서 부터 메시지 처리를 재개한다. 이떄 초기 부하가 증가할 수 있다. 
4. 네트워크 및 리소스 사용 증가 : `Rebalancing` 과정에서는 `Consumer Group` 상태 변경, `Partition` 할당 계획 전송 등 추가적인 네트워크 통신이 필요하므로 추가적인 부하를 `Kafka` 에게 줄 수 있다. 
5. 오프셋 관리 : `Rebalancing` 이 발생하게 되면 `Consumer` 들은 매번 오프셋을 커밋하고, 새로운 `Partition` 의 오프셋을 재설정하게 되므로 오프셋 관리에 대한 오버해드가 증가 할 수 있다. 
6. 데이터 처리 분연속성 : `Rebalancing` 으로 인해 일부 데이터가 누락되거나 중복 처리 될 수 있다. 이러한 현상을 방지하기 위해서는 적절한 오프셋 관리 정책을 적용해야 한다. 


### Rebalancing reducing
`Rebalancing` 은 `Kafka` 에서 필요한 과정이지만, `Kafka` 의 최적화 및 시스템의 안정성을 위해서는 불필요하게 자주 발생하거나 
비효율적으로 진행될 경우 시스템 성능에 부정적인 영향을 줄 수 있으므로 피해야 한다. 

- `session.timeout.ms` : 적절한 값으로 설정해 일시적인 네트워크 지연이나 `Consumer` 의 일시적인 지연이 `Rebalancing` 을 유발하지 않도록 해야 한다. 해당 값은 `Consumer` 가 `Kafka` 에게 `Heartbeat` 를 보내는 최대 시간으로 세션시간이 짧으면 실패 빈도가 높아 질 수 있고, 너무 높으면 비활성 상태 판별이 늦어진다. 적절하게 큰 값으로 설정하는 것이 좋다. 
- `num.partitions` : `Topic` 당 `Partition` 의 수가 너무 많으면 `Rebalancing` 의 빈도가 높아질 수 있다. 그러므로 병렬성과 처리량을 위해서 초반부터 너무 큰 값의 설정은 피하는게 좋다. 
- `max.poll.interval.ms` : `Consumer` 의 메시지 처리는 다수의 외부 `I/O` 와 복잡한 로직으로 인해 처리시간이 길어 질 수 있다. 이런 경우 `Consumer` 는 더욱 빈번하게 `Consumer Group` 에서 제거되고 `Rebalancing` 발생 빈도가 높아 질 수 있으므로, `Consumer` 의 유휴 상태를 판별하는 해당 값을 높여 최적화 할 수 있다. 


---  
## Reference
[Kafka rebalancing](https://redpanda.com/guides/kafka-performance/kafka-rebalancing)  
[Understanding Kafka’s Consumer Group Rebalancing](https://www.verica.io/blog/understanding-kafkas-consumer-group-rebalancing/)  
[Apache Kafka Rebalance Protocol, or the magic behind your streams applications](https://medium.com/streamthoughts/apache-kafka-rebalance-protocol-or-the-magic-behind-your-streams-applications-e94baf68e4f2)  

