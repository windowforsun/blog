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


### Incremental Rebalancing
`Consumer Group` 의 구성원이 많을 수록 `Reblaancing` 에 소요되는 시간은 더 길어 질 수 있다. 
위에서 먼저 알아본 `Eager Rebalancing` 은 `Consumer Group` 의 `Rebalancing` 동안 기존 `Consumer` 의 메시지 처리 또한 중단한다. 
만약 `Consumer Group` 이 매우 크거나 실시간성이 중요해서 해당 영향이 너무 크다고 판단된다면, `Incremental Rebalancing` 을 사용할 수 있다. (이는 다른 말록 `Coorperative Rebalancing` 이라고도 한다.)
해당 방법은 `Group Coordinator` 로 부터 `Rebalancing` 수행 중이라는 메시지를 받은 기존 `Consumer` 들은 메시지 처리를 중단하지 않는다. 
대신 `Rebalancing` 은 2단계에 걸쳐 수행되게 되는데, 아래는 `Group Coordinator` 에게 기존 `Consumer` 가 `Rebalancing` 시작을 받은 이후 처리 과정이다. 

1. 기존 `Consumer` 는 `Group Coordinator` 에게 기존과 동일하게 `JoinGroup` 요청을 보내지만 자신에게 할당된 `Partition` 의 메시지는 계속해서 처리한다.
  - `JoinGroup` 요청에는 자신이 구독하는 `Topic` 별 `Partition` 할당 정보가 포함된다. 
2. `Group Coordinator` 가 기존 모든 `Consumer` 에게 `JoinGroup` 요청을 받으면(혹은 시간 초과) `JoinResponse` 를 소비자들에게 보내고 새로운 `Consumer leader` 를 선정한다. 
3. 새로운 `Consumer leader` 는 `Partition` 의 할당 정보를 `SyncGroup` 요청에 담아 응답한다. 
4. `Group Coorindator` 는 `Partition` 해제가 필요한 소비자들에게는 `SyncResponse` 응답에 해당 데이터를 담아 함께 응답한다. 
5. 재할당이 필요한 `Partition` 을 사용하는 기존 `Consumer` 만 실제로 메시지 처리를 중지한다. (`SyncResponse` 에 `Partition` 해제를 받은 소비자)
6. 2번째 `JoinGroup` 요청에는 모든 소비자(기존 + 신규)가 `Group Coordinator` 에게 자신이 할당 받은 `Partition` 과 해제한 `Partition` 에 대한 정보를 포함해서 요청한다. 
7. 이 시점에는 `Consumer Group` 이 안정된 상태로 `Rebalancing` 은 `Synchronization Barrier` 에 도달 했다고 판단하고 `Partition` 할당 완료를 진행한다. 

정리하면 실제로 처리가 중단되는 기존 `Consumer` 는 1단계 `Rebalancing` 에서 재할당이 필요한 `Partition` 을 사용중인 기존 `Consumer` 로 한정된다. 
즉 메시지 처리는 재할당이 필요한 `Partition` 만 중단된다고 할 수 있고, 그외 `Partition` 은 모두 정상 처리 된다. 

img/kafka/kafka-rebalancing-protocol-2.drawio.png

1. 기존 `Consumer A` 는 새로운 `Consumer B` 가 참여하기 전에는 토픽의 모든 `Partition` 을 `Polling` 한다. 
2. 새로운 `Consumer B` 가 `Consumer Group` 에 가입하게 되면, `Incremental Rebalancing` 이 트리거 된다. 
3. 기존 `Consumer A` 는 `Partition` 할당 계획으로 부터 `Partition` 해제가 나온 `Parition 2` 을 해제한다. 
4. `Partition 2` 는 새로운 `Consumer B` 로 할당되어 메시지를 이어서 소비한다. 
기존 `Consumer A` 가 소비하던 `Partition 1` 의 메시지는 중단되지 않고 계속해서 처리된다. 

   
위와 같이 `Incremental Rebalancing` 은 기존 `Consumer` 들의 처리를 모두 중단히지 않기 때문에 `Rebalancing` 에 따른 영향이 크지 않을 수 있다. 
하지만 2번에 걸친 `Rebalancing` 으로 인해 시작으로 부터 완료까지의 시간이 더욱 오래 소요 될 수 있다. 
`Incremental Rebalancing` 을 적용하고 싶다면 `Partition assignment strategy` 를 [CooperativeStickyAssignor]({{site.baseurl}}{% link _posts/kafka/2023-11-30-kafka-practice-kafka-partition-assignment-strategy.md %}) 로 지정하면 된다.  



---  
## Reference
[Kafka Consumer Group Rebalance - Part 2 of 2](https://www.lydtechconsulting.com/blog-kafka-rebalance-part2.html)  
[What happens when a new consumer joins the group in Kafka?](https://chrzaszcz.dev/2019/06/kafka-rebalancing/)  
