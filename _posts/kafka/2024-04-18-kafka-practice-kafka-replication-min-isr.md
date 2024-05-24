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


### Min In-Sync Replica
`Kafka` 가 `Producer` 로 부터 메시지를 받고 `Topic Partition` 에 메시지를 작성 할떄, 
`Topic Partition` 의 `Leader` 에게 메시지를 작성한다. 
여기서 `Leader Replica` `Topic Partition` 의 `Replica` 중 `ISR` 에 의해 `Leader` 로 선출된 `Replica` 를 의미한다. 
`Leader Replica` 에 작성된 메시지는 이후 `Follower Replica` 들에게 복제된다. 
이런 `Topic Partition` 의 복제 수는 `replication.factor` 에 의해 정해진다. 
만약 `replication.factor` 이 3이라면 데이터가 `Leader Replica` 에 써진 뒤 2개의 `Follower Replica` 에 데이터가 복제되어, 
총 3개의 `Replica` 에 데이터가 저장됨을 의미한다.  

`Replica` 에 데이터가 복제 될 때, 
`min.insync.replicas` 설정 만큼 데이터가 복제 된 후 `Producer` 가 `acks` 를 받을 수 있다. 
즉 이는 전송한 데이터가 성공적으로 작성을 보장하는 최소한의 `Replicas` 수를 결정한다. 
해당 값이 `all` 로 설정되면 모든 `Replica` 에 복제된 후에야 `Producer` 는 `acks` 를 수신해 이후 메시지 처리가 가능하다. 

### ISR
`ISR(In-Sync Replicas)` 는 `Partition Replica` 중 `Leader Replica` 와 동기화 된 `Replica` 를 의미한다. 
`Follower Replica` 들은 동기화를 위해 `Leader Replica` 에게 최신 데이터를 얻고, 
각 `Partition` 의 `Leader Replica` 는 `Follower Replica` 의 `Lag` 상태를 추적한다. 
그리고 `Leader Replica` 가 `replica.lag.time.max.ms` 내 `Follower Replica` 에게 최신 데이터 `Fetch` 요청을 받지 못하거나, 
해딩 시간내에 `Leader Replica` 의 가장 최신 데이터까지 동기화를 못한 경우에는 동기화 되지 않은 걸로 간주해 `ISR` 에서 제거된다.  

### Under Replicated Partition
`Topic Partition` 에 `replication.factor` 만큼 존재하지만, 
`Follower Replica` 가 `Leader Replica` 와 비교해서 `Lag` 이 크게 발생한 경우 이를 `Under Replicated Partition` 이라고 한다. 
운영중 모니터링에 중요한 항목이기 때문에 `Topic Partition` 중 `Under Replicated Partition` 이 존재하는지 확인이 필요하다.  

### HWM(High Watermark) Offset
`High Watermark Offset` 은 `ISR Topic Partition` 에 작성된 최신 `Offset` 을 의미하고, 
이는 `Replica` 에 `Commit` 되었고 `Durability` 를 보장하는 가장 최신 데이터이다. 
`Producer` 는 `Topic Partition` 중 `Leader Replica` 에게만 데이터르 작성하고, 
`Consumer` 는 `Leader, Follower Replica` 모두에게 메시지를 읽을 수 있다. (별도 설정 필요)
그리고 `Consumer` 가 소비할 수 있는 최신 메시지의 위치가 `High Watermark Offset` 이므로 해당 위치까지만 읽을 수 있다. 
`Follower Replica` 는 `Leader Replica` 로 부터 `High Watermark Offset` 을 전달 받는데 그 시점은 아래와 같다. 
`Leader Replica` 에게 쓰여진 데이터가 `ISR` 의 모든 `Follower Replica` 에게 작성되고 확인되면, 
`Leader Replica` 는 최신화된 `High Watermark Offset` 을 `Follower Replica` 들에게 `Fetch` 요청의 응답으로 알린다.    


### Replication Config
아래표는 `Kafka` 에서 `Topic Partition` 의 `Replication` 과 연관된 주요한 설정들을 정리한 것이다.  

Properties|Type|Desc|Default
---|---|---|---
acks|Producer|`Producer` 가 `Broker` 에게 메시지 작성을 요청했을 때 얼만큼의 `Replication` 의 `acks` 를 받을지|all
default.replication.factor|Broker|자동생성되는 `Topic` 의 `Replication Factor` 적용 값|1
min.insync.replicas|Broker/Topic|유지할 `ISR` 의 최소값으로, `acks=all` 일떄 해당 수 만큼의 `Replication` 의 작성이 완료돼야 한다. `Topic` 에 설정되지 않은 경우 `Broker` 의 설정 값을 사용한다. 
replica.lag.time.max.ms|Broker|`ISR` 에 제거되기까지 `Replication` 의 `lag` 복구를 기다리는 최대 지연 시간|3000

자동으로 생성되는 `Topic` 의 경우 `default.replication.factor` 에 의해 복제본 수가 정해지고, 
이후 조정이 필요하다면 `Kafka Admin` 을 사용해서 `--replication-factor` 를 사용해 지정해야 한다.  

### Example
`Replication` 의 예시를 보기위해 3개로 구성된 `Kafka Broker` 와 아래 설정 값을 사용한다고 가정한다.  

Properties|Value
---|---
acks|all
default.replication.factor|3
min.insync.replica|2

아래 도식화된 그림을 보면 `T1` 이라는 `Topic` 은 `default.replication.factor` 의해 3개의 `Replication` 을 갖고 이는 각 노드에 존재한다. 
`M1`, `M2` 메시지가 쓰여진 상태로 이 두 메시지는 모든 `Replication` 에 복제가 성공했으므로 `High Watermark Offset` 은 2인 상황이다.  

이어서 `M3` 가 `Producer` 로 부터 `Leader Replica` 에게 전달되고 `offset` 3으로 저장된다. 
그리고 2번 노드 `Follower Replica` 까지는 복제가 성공했지만, 
3번 노드 `Follower Replica` 는 아직 복제가 되지 않은 상태이다.  

.. 그림 ..

위와 같이 3번 노드의 `Follower Replica` 가 동기화 상태가 아닌 경우라도, 
`ISR` 인 1번/2번 노드가 있기 때문에 메시지 쓰기는 성공으로 간주된다. 
그리고 `Leader Replica` 는 `Producer` 에게 `ack` 를 보내 메시지 쓰기 성공을 응답하게 된다.  

만약 3번 노드의 `Follower Replica` 도 동기화 상태인 `ISR` 이라고 가정하더라도, 
`min.insync.replicas=2` 이기 때문에 3번 노드의 쓰기까지 `Leader Follower` 가 쓰기 성공 여부 응답을 기다리거나 검사하지는 않는다. 
즉 이는 `min.insync.replicas` 가 클수록 `Leader Follower` 는 많은 `Follower Replica` 가 `ISR` 상태인지 확인 해야 해서 추가적인 대기시간이 발생할 수 있고, 
값이 적을 수록 더 적은 수의 확인만 수행하면 돼서 더 적은 대기시간이 발생한다.  

### Producer Acks
[Producer Acks]()
에서 더 자세한 설명을 확인 할 수 있다. 
`acks=all` 이 아닌 경우에는 `min.insync.replica` 설정은 중요하지 않다. 
`acsks=1` 이라면 `Leader Replica` 만 메시지를 쓰면 성공으로 간주하고 `ack` 를 전송한다. 
그리고 이후 혹은 동시에 나머지 `Follower Replica` 에 복제가 수행될 텐데, 
복제가 성공하기 전에 `Leader Replica` 가 가용 불가능 상태가 된다면 `Producer` 는 해당 메시지를 쓴 것으로 인지하지만 실제로는 유실된 상황이 발생할 수 있다. 
만약 `acks=0` 인 경우에는, `Producer` 는 메시지 쓰기 요청을 보낸 후 어떠한 응답도 기다리지 않는 `fire and forget` 으로 동작한다.  


### Producer Retry
`Producer` 가 `Leader Replica` 에게 메시지 쓰기 요청을 해 수행하던 중 
`min in-sync replicas` 수를 충족할 수 없는 복제본 상태가 발생하면 예외가 발생하고 `Producer` 는 재시도를 수행한다. 
이는 `min.insync-replicas=3` 인 설정에서 3번 노드가 가용 불가상태라고 가정한다면, 
2번 노드가 복제 쓰기를 승인하기 전에 실패가 발생한다. 
이후 복제본 중 하나를 `ISR` 로 승격시키고 재시도 요청을 통해 복제가 성곡하게 되어 최종적으로 쓰기도 성공하게 된다.  

`Producer Retry` 동작은 중복 메시지를 생성할 수 있는데 이는
[Idempotent Producer]()
에서 확인 할 수 있다.  


### Replication Flow
아래 그림은 복제흐름을 도식화한 것이다. 
`replication.reactor=2`, `min.insync.replicas=2`, `acks=all` 일 때 
`Leader Replica` 로 부터 쓰여진 메시지가 `Follower Replica` 에게 복제되는 흐름을 보여준다. 
설정된 내용으로 메시지 쓰기는 2개의 `Replication` 모두 쓰기가 완료됨을 확인해야 하고, 
현재 `High Watermark Offset` 은 2인 상태에서 새로운 메시지가 쓰여지는 시나리오이다.  

.. 그림 ..

`Follower Replica` 는 `Consumer` 와 동일한 방식(poll())으로 `Leader Replica` 로 부터 최신 메시지를 가져오는데, 
해당 요청을 통해 `Leader Replica` 에세 자신의 현재 `offset` 정보를 알린다. 
`Leader Follower` 는 `Follower Replica` 의 `offset` 을 통해 `lag` 이 얼마나 발생하는지, 
어떤 레코드를 필요로하는지 판단한다. 
그리고 `Leader Replica` 는 `Follower Replica` 에게 필요한 메시지와 최신 `High Watermark Offset` 도 함께 응답한다. 
위 시나리오에서 최신 메시지는 `offset` 이 3인 메시지이고, `High Watermark Offset` 은 2인 값이 된다. 
`Follower Replica` 는 해당 응답 값을 통해 메시지 쓰기를 수행하고 다시 최신 메시지를 가져오는 동작을 반복한다. 
그리고 `Leader Replica` 는 `Follower Replica` 의 동기화가 완료 됐음을 인지하고, 
`ISR` 에 있는 모든 복제본들의 동기화가 완료 됐기 때문에 `High Watermark Offset` 을 3으로 업데이트하고 다음 `Follower Replica` 요청에 응답으로 보내게 된다. 
최종적으로 `Leader Replica` 는 `Producer` 에게 쓰기 요청에 대한 `ack` 를 응답하고, 
이어서 `Follower Replica` 는 `High Watermark Offset` 을 업데이트하게 된다.  


이제 `Replication` 이 한개 더 추가된 좀 더 복잡한 시나리오이다. 
총 3개의 복제본이 있고 모두 `ISR` 에 속해 있으며, `Producer` 쓰기가 성공하기 위해서는 모든 복제본에 쓰기가 성공해야 한다.  

.. 그림 .. 

위 시나리오에서는 `Leader Replica` 가 2개의 `Follower Replica` 로 부터 쓰기 확인을 각각 받은 후에 `High Watermark Offset` 을 갱신할 수 있다. 
해당 시점에 `Producer` 는 쓰기 요청 성공을 확인 하고, `Follower Replica` 는 다음 요청 `Leader Replica` 의 응답을 통해 자신들의 `High Watermark Offset` 을 갱신할 수 있다. 
이러한 시나리오를 통해 `ISR`(`min in-sync replicas`) 의 수가 늘어날 수록 `Leader/Follower Replica` 에서 데이터를 읽는 `Consumer` 들의 지연시간이 늘어 날 수 있음을 의미한다. 
`Consumer` 가 읽을 수 있는 가장 최신 데이터는 `High Watermark Offset` 이기 때문에 해당 값이 업데이트 되기 전까지는 메시지를 얻을 수 없다. 
만약 직접 `Follower Replica` 로 부터 메시지를 읽는 `Consumer` 라면 이러한 지연 시간은 좀더 증가 할 수 있다.  


### Durability, Availability and Latency
`min.insync.replicas` 의 값을 늘리게 되면 해당 `Topic Partition` 의 `Durability` 는 증가하는데, 
이는 메시지가 다수의 노드/브로커에 걸쳐 복제 됐음을 보장하기 때문이다. 
그리고 특정 노드/브로커가 가용 불가상태가 되더라도 신규 `Leader Replica` 를 선출하거나 `Follower Replica` 에서 메시지를 읽는 등으로 
더 높은 `Availibility` 를 보장 할 수 있다. 
하지만 이는 앞서 설명한 것처럼 지연시간을 늘릴 수 있다. 
`Producer` 로 부터 전달된 메시지 쓰기가 완전히 쓰여진걸로 확인 되기 까지, 
혹은 `Consumer` 가 `Producer` 로 부터 전달된 메시지를 읽기까지 `ISR` 구성이 모두 동기화 된 상태(리더로 부터 동기화 확인)를 확인해야 하기 때문이다. 


### Unclean Leader Election
`Kafka Cluster` 를 구성하는 여러 노드 중 하나의 노드가 실패하면, 
해당 노드에 있는 복제본, `Leader Replica` 들은 재할당이 필요하다. 
`unclean.leader.election.enable=false` 라면, 
`Leader Replica` 의 노드가 죽기전 동기화가 완료된 `Follower Replica` 중에서만 새로운 `Leader Replica` 가 선출 될 수 있다. 
`unclean.leader.election.enable=true` 일 때는, 
`Leader Replica` 의 노드가 죽기전 동기화가 완료되지 않은 `Follower Replica` 도 `Leader Replica` 로 선출 될 수 있다. 
이는 복제되지 않은 메시지들로 인해 메시지 유실이 발생 할 수 있고, 
데이터의 일관성과 가용성간의 상관관계를 의미한다.  


### Consumer Offsets
`__consumer_offsets` 는 `Kafka Consumer` 가 메시지를 성공적으로 소비한 `offset` 을 관리하는 내부 토픽이다. 
이는 `Kafka Broker` 설정 값인 `min.insync.replicas` 를 통해 `offset` 쓰기 요청에 대한 요구사항을 검증하고, 
`offsets.topic.replication.factor` 값을 통해 복제본을 얼만큼 유지할지 결정한다. 
그리고 `offsets.commit.required.acks` 는 `Producer` 의 `acks` 설정과 동일한 역할로, 
기본값은 -1(`all`)로 이는 `offset` 쓰기가 `ISR` 로 복제되고 `min.insync.replicas` 설정을 만족할 때까지는 성공으로 간주하지 않는다는 것을 의미한다.  

Properties|Type|Desc|Default
---|---|---|---
offset.topic.replicaiton.factor|Broker|`__consumer_offsets` 토픽의 복제본 수|3
offset.commit.required.acks|Broker|`offset` 쓰기 요청을 성공으로 판별하기 위해 `ISR` 의 최소 수|-1(all)


### Transaction State Log
`Kafka` 는 `__transaction_state` 라는 내부 토픽을 사용해서 트랜잭션 상태 로그를 추적하고 관리한다. 
해당 토픽의 `ISR` 요구사항은 브로커의 `transaction.state.log.min.isr` 설정값으로 결졍된다. 
그리고 `transaction.state.log.replication.factor` 설정값을 통해 얼만큼의 복제본을 유지할지 결정한다.  

Properties|Type| Desc      |Default
---|---|-----------|---
transaciton.state.log.replication.factor|Broker| 유지할 복제본 수 |3
transaction.state.log.min.isr|`ISR` 의 최소 수| 2         



---  
## Reference
[Kafka Replication & Min In-Sync Replicas](https://www.lydtechconsulting.com/blog-kafka-replication.html)  


