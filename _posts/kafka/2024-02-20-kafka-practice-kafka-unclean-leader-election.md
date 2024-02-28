--- 
layout: single
classes: wide
title: "[Kafka] Kafka Unclean Leader Election"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Cluster 에서 약간의 메시지 손실로 내구성을 줄이고 가용성을 높이는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Cluster
    - Kafka Broker
    - Availability
    - Durability
toc: true
use_math: true
---  

## Unclean Leader Election
`unclean.leader.election.enable` 은 `Kafka Cluster` 의 구성원인 `Leader` 와 `Follower` 가 있을 떄, 
`Leader` 의 복제본이 아직 모두 동기화되지 않은 `Follower` 가 오류 시나리오에서 `Leader` 가 될 수 있는지를 결정하는 옵션이다. 
만약 해당 옵션을 활성화 시킨다면 `Leader` 로 선정된 `Follower` 에는 모든 복제본의 데이터가 존재하지 않을 수 있으므로, 
메시지의 손실이 발생 할 수 있다.  

해당 옵션을 설정여부에 따라 `Kafka Cluster` 가 보장하는 범위나 장단점에 확실한 차이가 있으므로 
옵셜 설정에 따른 결과를 이해하는 것이 중요하다.  

### Availability and Durability
`Kafka Cluster` 가 3개의 브로커 노드로 구성돼 있다고 가정하자. 
`Node 1` 의 브로커가 `Leader` 이고 나머지 `Node 2` 와 `Node 3` 의 `Follower` 인 상태이다. 
그리고 `Leader` 에 위치하는 `Partition` 에 쓰여진 메시지들은 `Follower` 의 복제 `Partition` 에 동기화 돤다.  

각 `Partition` 은 `ISR` 이라는 `in-sync replicas` 목록을 가지고 있다. 
일반적으로 `Leader` 는 `ISR` 를 가지고 있는 `Broker` 중에 선정된다. 
만약 `ISR` 을 가진 `Broker` 가 없다면 `OSR(out-of-sync replica)` 중에서 `Leader` 를 선정할지
`unclean.leader.election.enable` 옵션 값을 통해 판단하게 된다. 

아래 그림은 위 상황을 도식화 한 것이다.  
`message 1`, `message 2`, `message 3` 의 3개의 메시지가 있을 때 `Node 2` 에는 모든 메시지가 정상적으로 동기화 됐지만 
`Node 3` 에는 아직 `meesage 3` 이 동기화되지 않은 상태이다.  

![그림 1]({{site.baseurl}}/img/kafka/unclean-leader-election-1.drawio.png)


`unclean.leader.election.enable` 이 활성화 된 상태에서 아래 시나리오로 인해 메시지 손실이 발생 할 수 있다.  

1. `Node 2` 는 `Leader` 의 모든 메시지가 동기화된 상태이다. 
2. 하지만 `Node 3` 은 `Lag` 발생으로 인해 아직 모든 메시지가 동기화 되지 않았다. (`message 1`, `message 2` 만 `in-sync` 상태)
3. `Node 1`, `Node 2` 가 일련의 이슈로 가용 불가 상태이다. `Kafka Cluster` 에서 가용 가능한 것은 `Node 3` 뿐이다. 
4. `Node 3` 이 `Leader` 로 선정되고, `Node 3` 에 동기화 되지 않은 `message 3` 은 유실된다.  

`unclean.leader.election.enable` 을 활성호 했기 때문에 위 상황에서도 `Kafka Cluster` 는 가용 상태이므로 읽기/쓰기 동작은 가능하다. 
하지만 메시지 일부 손실이 발생할 수 있으므로 `Availability` 는 높아지고 `Durability` 를 일부 희생하게 된다.  

만약 위 시나리오에서 `unclean.leader.election.enable` 이 비활성화 된 상태라면 `Kafka Cluster` 가 일정시간 가용불가 상태로 읽기/쓰기 동작이 불가능 할 수 있다. 
`Kafka Cluster` 는 해당 `Partition` 의 데이터가 완전한 복제본이 활성화 될때까지 대가히기 때문이다. 
즉 `unclean.leader.election.enable` 옵션은 활성화하지 않은 것은 `Availability` 보다는 `Durability` 를 더 강조하는 설정이다.  

### Configuration
`unclean.leader.election.enable` 옵션은 `Kafka Broker` 에서 여러 `Topic` 에 적용되는 글로벌한 설정을 할 수 있다. 
그리고 각 `Topic` 별로도 해당 옵션을 수정해서 재정의가 가능하다. 
`Kafka Cluster` 에서 해당 옵션의 기본값은 `false` 이다. 
일반적인 `Kafka Cluster` 들은 가용성보다는 내구성을 우선시하고 있는 것이다. 
`0.11.0.0` 이전 버전의 경우 해당 옵션의 기본값이 `true` 였던 적도 있었다. 
그리고 일부 플랫폼에서 제공하는 `Kafka Cluster` 는 기본 제공 옵션 값이 다를 수 있으므로 
사용전 확인이 필요하다.  

메시지 손실을 허용하지 않는 시스템이라면 `unclean.leader.election.enable` 을 활성화시켜서는 안된다. 
그리고 어느정도 메시지 손실은 허용하고 시스템의 고가용성에 초점을 좀 더 맞추고 싶다면 해당 옵션을 활성화 할 수 있다. 
그리고 각 `Topic` 별로 특성에 따라 해당 옵션을 달리 적용하는 유연성도 제공하기 때문에 비지니스에 맞춰 옵션 값을 설정해주면 된다.  



---  
## Reference
[Kafka Idempotent Producer](https://www.lydtechconsulting.com/blog-kafka-idempotent-producer.html)  
[Broker unclean.leader.election.enable](https://docs.confluent.io/platform/current/installation/configuration/broker-configs.html#unclean-leader-election-enable)  
[Topic unclean.leader.election.enable](https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html#unclean-leader-election-enable)  

