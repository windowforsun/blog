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

## Kafka Rebalancing 이해를 위해 몇가지 배경 지식
[Kafka Rebalancing]()
에서 설명한 것과 같이 `Rebalancing` 은 `Kafka` 에서 중요한 개념이고 처리량에 영향을 줄 수 있다. 
그러므로 이에 대한 과정과 원리를 어느정돈 이해할 필요가 있는데 알아두면 `Rebalancing` 이해에 도움이 되는 몇가지 배경지식을 소개하고자 한다. 

### Consumer Groups
`Kafka` 의 `Consumer` 는 기본적으로 애플리케이션이 `Topic` 으로부터 메시지를 소비하는 역할을 하는데, 
이런 각 `Consumer` 들은 하나의 `Consumer Group` 에 속하게 된다. 
그리고 `Consumer Group` 안에서 `Topic` 에 있는 여러 `Partition` 중 하나의 `Partition` 을 할당 받게 된다. 
즉 `Topic` 은 `Consumer Group` 과 연결되고, `Consumer` 는 `Partition` 과 연결된다 이렇게 비유할 수 있다. 
여기서 `Group Membership` 인 `Consumer Group` 에 가입/탈퇴에 대한 관리는 `Kafka Brokder` 에 의해 결정되고, 
`Topic` 의 `Partition` 의 할당에 대한 관리는 클라이언트인 `Conumser` 측에서 관리된다. 
이러한 이유로 `Kafka Broker` 측면에선 어떤 `Consumer`(클라이언트) 가 어떤 `Partition` 과 관계가 있는지 알지 못한다. 
이러한 `Kafka Client` 의 특징은 `Kafka Client` 가 왜 무겁고 클라이언트라고 불리는지 알려주는 좋은 예이다.  

`Consumer` 는 동일한 `group.id` 를 갖는 `Consumer` 인스턴스들과 동일한 `Consumer Group` 에 속하게 된다. 
이를 통해 `Topic` 을 구독하는 `Consumer` 의 수를 확장하고 축소하는데 용의하기 때문에 메시지 처리량에도 큰 장점이라고 할 수 있다.  

그리고 `Group Coordinator` 는 `Consumer Group` 과 이를 구성하는 `Consumer` 를 관리하는 `Kafka Brokder` 의 구성요소 중 하나이다. 
`Group Coordinator` 는 `Consumer Group` 의 구성원 중 하나의 `Consumer` 를 `Leader` 로 만들어 구독하는 `Topic` 의 `Partition` 을 
어떤 식으로 `Consumer` 에게 할당할지 계산하는 역할을 하게 된다. 
그 결과는 다시 `Group Coordinator` 에게 반환되고 각 `Consumer` 가 `Partition` 을 할당 하도록 한다.  

아래 그림은 `group.id` 가 `foo` 인 `Consumer` 가 6개의 `Partition` 으로 구성된 `first` `Topic` 을 구독하는 예시이다. 

kafka-rebalancing-bg-info-1.drawio.png

이 후 `Consumer Group` 의 두 번째 멤버가 그룹에 참여한다. 
해당 `Consumer` 또한 `group.id` 는 `foo` 이고 `first` `Topic` 를 구독한다. 
해당 `Consumer` 는 `JoinGroup` 요청을 `Group Coordinator` 에게 전송하고, 
`Partition` 은 `Consumer Group` 멤버 전체를 대상으로 재할당 된다. 
결과적으로 2개 멤버가 각 3개의 `Partition` 을 아래와 같이 할당 받는다.

kafka-rebalancing-bg-info-2.drawio.png

한가지 주의할 점은 `Partition` 의 수보다 `Consumer Group` 의 멤버 즉 인스턴스 수가 더 많은 경우, 
몇 인스턴스는 `Partition` 할당을 받지 않을 수 있다. 
즉 이는 `Topic` 의 `Partition` 이 6개인 상태에서 `Consumer Group` 의 멤버 수가 10명이면 2명의 `Consumer` 는 `Idel` 상태에 있게 된다.  


만약 다른 `group.id` 를 사용하는 `Consumer` 가 동일한 `Topic` 인 `first` 를 구독하게 되면, 
이는 별도의 `Consumer Group` 이 추가된 상황이 된다. 
즉 `Partition` 의 할당은 `group.id` 가 `foo` 인 할당과 별개로 `group.id` 가 `bar` 인 `Consumer` 들에게 할당이 수행된다.  

kafka-rebalancing-bg-info-3.drawio.png


### Rebalancing trigger
`Rebalancing` 의 시작을 알리는 `Reblaancing Trigger` 의 요인에는 몇가지가 있다. 
`Consumer Group` 에 새로운 `Consumer` 가 가하거나 탈퇴하거나 혹은 `Brokder` 가 특정 `Consumer` 의 실패를 탐지 했을 때이다. 
그 외에도 `Partition` 수 조정 등과 같은 리소스 재할당 필요하는 등의 상황에서 트리거 될 수 있다. 
한 예시로 `Consumer` 가 구독을 할때 `Topic` 이름의 특정 패턴을 통해 여러 `Topic` 을 구독하는 상황이 있다.  

새로운 `Consumer` 가 `Consumer Group` 에 가입할 떄 `Broker` 인 `Group Coordinator` 에게 
`JoinGroup` 요청을 전송한다. 
그 후 `Rebalancing` 은 `Consumer Group` 에 포함된 모든 `Consumer` 를 대상으로 이뤄진다. 
위와 같이 기존 `Consumer` 가 `Consumer Group` 을 떠날 떄도 `Group Coordinator` 에게 `LeaveGroup` 요청을 보내게 되고, 
떠난 후 `Consumer Group` 에 남겨진 모든 `Consumer` 를 대상으로 `Rebalancing` 이 수행된다.  

그리고 `Group Coorindator` 가 `Consumer Group` 멤버 중 특정 `Consumer` 의 `Healthbeat` 를 
설정 시간 내에 정상 수신하지 못하는 경우에도 `Rebalancing` 은 트리거된다.  

`Rebalancing` 의 주체는 `Consumer Group` 임을 기억해야 한다. 
즉 `Consumer Group` 멤버가 각 다른 `Topic` 을 구독하고 있는 상태에서 어떠한 이유에서든 `Rebalancing` 이 트리거 되면, 
`Consumer Group` 멤버 전체 `Consumer` 가 해당하고 `Partition` 은 모두 재할당 된다.  
그 예로 다음을 보자. 
만약 `group.id` 가 `foo` 로 동일한 `Consumer Group` 에 2개의 `Consumer` 가 있다. 
`group.instance.id` 가 `1` 인 `Consumer` 는 `first` 라는 `Topic` 을 구독하고, 
`gorup.instance.id` 가 `2` 인 `Consumer` 는 `second` 라는 `Topic` 을 구독하는 상태이다. 
위 상황에서 `1번 Consumer` 의 처리시간이 너무 오래 걸려 `Rebalancing` 이 트리거 된다면 `2번 Consumer` 의 파티션도 함께 재할당 된다.  

kafka-rebalancing-bg-info-4.drawio.png

위와 같은 상황을 피하기 위해서는 `Consumer Group` 의 전체 멤버가 서로 다른 `Topic` 을 처리하는 상황은 피하는 것이 좋다.  

### Rebalancing config
`Rebalancing` 과 관련된 `Kafka Client` 의 설정 값을 정리하면 아래와 같다.  

설정|설명|기본값
---|---|---
`session.timeout.ms`|`Consumer` 의 실패 상태를 탐지하는 최대 시간값이다. 해당 시간값 이내 `Group Coordinator` 가 `Heartbeat` 를 수신해야 한다.|45s(Kafka 3.0.0)
`heartbeat.interval.ms`|`Consumer` 가 `Group Coordinator` 에게 `Heartbeat` 의 전송 간격이다. 위 세션 유지 여부를 결정하는 데 사용된다.|3s
`max.poll.interval.ms`|`Consumer` 가 `poll` 을 수행하는 최대 시간 간격으로 해당 시간 값 이내 `poll` 이 수행돼야 한다.|5m

#### Heartbeat 와 Session timeout
`Consumer` 는 `Broker` 에 위치한 `Group Coordinator` 에게 주기적인 `Hearbeat` 를 전송하고, 
`Group Coordinator` 는 이 `Heartbeat` 를 통해 `Consumer Group` 멤버들의 상태를 모니터링한다.
`Heartbeat` 의 전송주기는 `heartbeat.interval.ms` 이고,
`Heartbeat` 는 `session.timeout.ms` 이내 수신해야 하는데, 
`Hearbeat` 를 수신하면 `session.timeout.ms` 의 값을 리셋되고 다시 타임아웃을 리셋한다.  

kafka-rebalancing-bg-info-5.drawio.png

`heartbeat.interval.ms` 는 `session.timeout.ms` 의 `1/3` 수준으로 설정하는 것이 좋다. 
이러한 방식은 일시적인 네트워크 오류등으로 `Heartbeat` 가 한두번 실패하더라도, `Consumer` 실패로 이어지지 않기 때문이다. 
아래 도식화된 그림을 보면 한번의 일시적인 실패에도 `session.timeout.ms` 이전에 `Heartbeat` 가 성공했기 때문에, 
`Group Coordinator` 는 `Consumer` 을 가용상태로 인식한다.  

kafka-rebalancing-bg-info-6.drawio.png

아래와 같이 `session.timeout.ms` 가 만료될때까지 `Heartbeat` 가 `Group Coordinator` 로 전달되지 않으면, 
`Consumer` 를 가용할 수 없는 상태로 인식하고 `Reblancing` 이 트리거 된다. 

kafka-rebalancing-bg-info-7.drawio.png

#### Poll interval
`Consumer` 에서 `Hearbeat` 를 수행하는 스레드는 데이터를 처리하는 메인 스레드와 별도의 스레드로 이뤄진다. 
메인 스레드는 구독하는 `Topic` 의 `Partition` 을 `poll()` 하고, 
이러한 폴링 동작은 `max.pool.interval.ms` 내에 수행돼야 한다. 
아래 그림은 앞선 그림에서 데이터를 처리하는 스레드를 추가했을 떄의 그림이다.  

kafka-rebalancing-bg-info-8.drawio.png

첫 번째 `poll()` 수행은 구독 데이터에 대한 기존 `poll()` 동작 뿐아니라 `Parition` 할당과 같은 변경사항 정보도 함께 포함한다. 
그리고 첫 번째 `poll()` 이 수행된 이후 `Heartbeat` 스레드가 시작되고, 
이후에 반복해서 수행되는 `poll()` 동작이 완료되면 `max.poll.interval.ms` 의 시간을 리셋한다.  

`Heartbeat` 스레드는 `Consumer` 의 상태를 확인하고 `poll()` 수행 사이에 
`max.poll.interval.ms` 가 초과됐다면 `LeaveGroup` 요청을 `Group Coordinator` 에게 보내 해당 `Consumer` 가 제거되고 
`Rebalancing` 이 트리거 될 수 있도록 한다.  

kafka-rebalancing-bg-info-9.drawio.png

`Rebalancing` 이 수행되면 기존 `Consumer` 들은 `Rebalancing` 이라는 응답을 다음 `Heartbeat` 의 응답으로 받는다. 
그리고 각 `Consumer` 들은 `max.poll.interval.ms` 타임아웃 전까지 `poll()` 을 수행해서 `Group Coordinator` 에게 
`JoinGroup` 요청을 보내 그룹에 다시 가입해야 한다. 
`Kafka Connect` 의 경우 `rebalance.timeout.ms` 라는 별도의 타입아웃 설정 값이 존재한다.  

위와 같은 이유들로 `max.poll.interval.ms` 의 값 설정은 신중함이 필요하다. 
너무 낮은 값은 메시지 처리에 오래걸리는 경우에 빈번한 `Rebalancing` 이 트리거 될 수 있고, 
너무 높은 값은 `Consumer` 가 실패 상황에서 `Kafka Broker` 가 이를 인지하는 시간이 오래걸려 
필요한 `Rebalancgin` 동작이 늦게 트리거 될 수 있다. 


### Consumer health
`Consumer` 의 상태점검의 정확성을 위해서는 `max.poll.interval.ms` 와 `session.timeout.ms` 두 가지 타임아웃 값을 적절하게 조절해야한다. 
메인 스레드에 오류가 있고 `Heartbeat` 스레드가 정싱안 경우에는 `max.poll.interval.ms` 타임아웃이 만료되며 에러를 감지할 수 있다. 
`Consumer` 가 수행되는 애플리케이션 자체적으로 문제가 있다면 `session.timeout.ms` 타임아웃 이내 `Heartbeat` 가 수신되지 않으며 감지 가능하다.  


---  
## Reference
[Dynamic vs. Static Consumer Membership in Apache Kafka](https://www.confluent.io/blog/dynamic-vs-static-kafka-consumer-rebalancing/)  
[Kafka Consumer Group Rebalance - Part 1 of 2](https://www.lydtechconsulting.com/blog-kafka-rebalance-part1.html)  
