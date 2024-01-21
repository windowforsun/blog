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


## Rebalancing Protocol
`Kafka 2.4` 이전 버전의 `Kafka` 에서는 기존에 `Eager Rebalancing Protocol` 을 기본 
`Rebalancing Protocol` 로 사용했다. 
`Eager Rebalancing` 의 단점을 좀 더 보완한 `Incremental Rebalancing` 이 `Kafka 2.4` 버전 부터 나오게 되었고, 
기본은 동일하게 `Eager Rebalancing` 이지만 필요에 따라 `Incremental Rebalancing` 을 설정해 사용할 수 있다. 
본 포스팅에서는 두가지 `Reblaancing Protocol` 의 동작 방식와 어떠한 차이점들이 있는지 확인해 본다.  


> #### Group Coordinator
> - `Kafka Cluster` 의 브로커 중 하나이다. 
> - 특정 `Consumer` 에 대한 메타 데이터를 관리하고, `Rebalancing` 프로세스를 조정한다. 
> - `Consumer Group` 생성 또는 그룹 가입때 `Group Coordinator` 와 연결을 맺게된다. 
> - `Group Coordinator` 는 `Consumer` 가입/탈퇴를 처리하고, 필요시 `Rebalancing` 을 시작한다. 
> 
> #### Group Leader
> - `Consumer Group` 내 특정 `Consumer` 가 `Group Leader` 로 선정 될 수 있다. 
> - 해당 `Consumer` 는 그룹 내 다른 `Consumer` 를 대표해서 `Group Coordinator` 와 통신을 담당한다. 
> - `Rebalancing` 과정에서 `Group Leader` 는 새로운 `Partition Assignment` 계획을 수립하고 이를 다른 `Consumer` 들에게 전파한다.

### Eager Rebalancing
`Eager Rebalancing` 은 `Kafka` 의 기본 `Consumer Group` 의 `Rebalancing` 방식으로, 
`Consumer` 에 의한 모든 처리가 중단되고 `Partition` 이 재할당 된다. 
이는 애플리케이션의 `Throughput` 에 큰 영향을 미칠 수 있기 때문에 동작 과정과 영향에 대해 미리 인지할 필요가 있다.  

이래 그림은 새로운 `Consumer`(`Consumer B`) 가 `Consumer Group` 에 가입 할 떄, 
기존 `Consumer`(`Consumer A`) 에게 미치는 영향과 관련 소요 시간을 보여주고 있다.  

img/kafka/kafka-rebalancing-protocol-1.drawio.png

1. 기존 `Consumer A` 는 구독하는 `Topic` 의 `Partition` 중 자신에게 할당된 `Partition` 으로부터 메시지를 `Polling` 한다.
그리고 `Kafka Broker` 중 하나인  `Group Coordinator` 에게 `Heartbeat` 를 전송해 전상임을 알린다. 
`Group Coordinator` 에게 `Heartbeat` 의 응답 확인을 받으면 `Consumer` 는 주어진 처리를 계속 수행한다.  
2. 새로운 `Consumer B` 는 `Group Coordinator` 에게 `JoinGroup` 전송해서 `Rebalancing` 을 트리거한다.
(`Consumer B` 는 설명을 위해 `Processing Thread` 는 제외하고, `Heartbeat Thread` 만 사용한다.)
3. `Group Coordinator` 는 기존 `Consumer A` 에게 받은 `Heartbeat` 응답으로 `Rebalancing` 이 시작 되었음을 알린다. 
4. 기존 `Consumer A` 는 `max.poll.interval.ms` 설정 시간 이내에 `poll()` 을 통해 받은 메시지 처리를 완료하고, 
`JoinGroup` 요청을 `Group Coordinator` 에게 전송해야 한다. 
5. 기존 `Consumer A` 의 `poll()` 메시지 처리가 완료되면, 메시지 처리는 `Rebalancing` 이 완료될 때까지 중단된다. 
그리고 다음 `poll()` 수행하는 타이밍에는 `Group Coordinator` 가 `Rebalancing` 에 필요한 토픽, 파티션 등의 구독관련 정보를 `JoinGroup` 요청과 함께 전송한다. 
6. `Group Coordinator` 는 `Consumer Group` 에 포함된 모든 소비자와 해당되는 `Topic` 의 모든 `Partition` 의 정보를 바탕으로 재할당이 필요함을 인지했다. 
이는 `Consumer Group` 의 구성원이 결정되어 `Synchronization barrier` (동기화 장벽)에 도달했음을 의미한다. 
7. `Group Coordinator` 는 `Consumer Group` 에 포함된 모든 소비자에게 `JoinRequest` 를 전송한다. 그리고 한 소비자가 `Consumer leader` 로 선정되고,
해당 소비자가 `Partition` 할당을 계산하게 된다. 
소비자는 `SyncGroup` 요청으로 `Group Coordinator` 에게 응답하게 되는데, `Consumer leader` 의 `SyncGroup` 응답에는 `Partition` 할당 계산의 결과가 포함된다. 
8. `Group Coordinator` 는 각 소비자에게 `Partition` 할당을 알리는 `SyncResponse` 를 응답한다. 
이후 기존 `Consumer A` 는 할당 받은 `Partition` 의 데이터를 이어서 `Polling` 하고, 
새로운 `Consumer B` 는 이 시점부터 `Polling` 을 시작한다. 

`Consumer Group` 의 `Rebalancing` 은 모든 소비자가 `Partition` 할당을 수작할 때까지 `Consumer Group` 의 `Rebalancing` 은 완료되지 않음을 유의해야한다. 
위 도식화한 그림이 `Rebalancing` 이 이뤄지는 과정에서 일시정지되는 구간을 강조하듯이, `Consumer Group` 에 더 많은 소비자가 있을 수록 일시정지의 시간이 더 길어 질 수 있기 때문에 일시정지 시간의 중요성을 인지해야한다.    

