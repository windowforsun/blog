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


## Consumer Group Membership
`Consumer Group Membership` 은 `Consumer Group` 의 메세치 처리 효율성과 확장성에 관련이 있다. 
`Consumer Group` 에 가입/탈퇴하는 `Consumer` 를 관리하는 방식에 2가지 방법이 있는데 방식에 따른 차이점과 
기본 개념에 대해 알아본다. 

### Dynamic Membership
`Dynamic Membership` 은 `Consumer Group` 의 `Consumer` 들의 가입/탈퇴에 대한 처리를 동적으로 처리하는 방식이다. 
`Consumer` 들이 그룹에 자유롭게 가입/탈퇴가 가능하고 이는 `Kafka` 에 자동으로 적용한다.  

주요 특징을 나열하면 아래와 같다. 

- 자유로운 가입/탈퇴 : `Consumer` 는 언제든지 `Consumer Group` 에 기압/탈퇴가 가능하다. 이 과정은 자동으로 관리된다. 
- 자동 `Rebalancing` : 새로운 `Consumer` 가 그룹에 가입하거나 탈퇴할 때마다, `Kafka` 는 자동으로 `Rebalancing` 을 수행한다. 이 과정은 토픽의 `Partition` 을 그룹내 `Consumer` 들에게 다시 할당하는 과정이다. 
- 유연성/확장성 : 요구사항이나 `Consumer` 사항이 변경 할 때 더 유연하게 대응 가능하다. 트래픽이 증가하여 더 많은 `Consumer` 가 필요 할 떄, 새로운 `Consumer` 를 추가하기만 하면 자동으로 `Rebalancing` 이 되고 부하는 분산된다. 
- 장애 내성(Fault Tolerance) : `Consumer` 중 하나가 실패하거나 네트워크 등의 문제로 분리될 경우, `Kafka` 는 `Reblaancing` 을 통해 해당 `Consumer` 가 처리하던 `Partition` 을 다른 `Consumer` 에게 재할당한다. 이를 통해 메시지 처리의 연속성과 시스템 안정성이 유지될 수 있다. 
- `Hearttbeat` : `Consumer` 는 주기적으로 `Kafka` 브로커에 `Heartbeat` 를 보내 자신의 상태를 알린다. 만약 `Heartbeat` 가 일정 시간 수신되지 않으면 해당 `Consumer` 는 탈퇴한 것으로 간주하고 `Rebalancing` 이 사작 된다. 

단점을 정리하면 아래와 같다. 

- `Rebalancing` 지연 : `Consumer Group` 의 변경이 잦은 경우 메시지 처리가 일시적 중단되고 지연이 발생 할 수 있다. 
- `Consumer Group` 안전성 : `Consumer Group` 의 잦은 변경은 `Kafka` 시스템 전체에 영향을 줄 수 있고, 이는 처리량이 크고 실시간성이 중요할 수록 더 민감해 질 수 있다. 
- `Offset` 관리 : `Rebalancing` 이 자주 발생 할 경우 각 `Offset` 관리는 더욱 복잡해지고, 관리가 잘못 되면 데이터 유실 혹은 중복 처리 증상이 발생하여 데이터 신뢰성에 영향을 미칠 수 있다. 

`Dynamic Menber` 의 수행 과정은 []()
에서 살펴본 도식화된 처리 과정들을 참고 할 수 있다. 


### Static Membership
`Static Membership` 은 `Kafka 2.3.0` 부터 도입된 새로운 `Consumer Group` 의 관리 방식이다. 
이름 그대로 `Dynamic Membership` 과 달리 정적인 방식으로 그룹을 관리하는데 주요 특징은 아래와 같다.  

- 고정된 `Member ID` : `Static Membership` 은 고유한 `Member ID` 를 가진다. 이는 `Consumer` 가 그룹에 재가입 할 떄도 동일하게 유지된다. 즉 `Consumer` 가 그룹을 탈퇴 후 재가입 하더라도 새로운 맴버로 간주하지 않는다. 
- `Rebalancing` 감소 : `Consumer` 가 잠시 가용 불가 상태였다가 재연결될 떄, 고정된 `Member ID` 를 통해 `Rebalancing` 빈도를 줄일 수 있다. 이는 네트워크가 불안정하거나 `Consumer` 의 재시작과 같은 장애 복구 상황에서 유리하다. 
- `Partition Assignment` 효율 : `Consumer` 가 그룹을 탈퇴 후 재가입 할때 해당 `Consumer` 는 이전에 할당 받았던 `Partition` 을 우선적으로 할당 받는다. 이는 파티션의 일관성을 유지하고 시스템 전반 효율을 올릴 수 있다. 
- 장애 내성(Fault Tolerance) : 일시적인 네트워크 이슈 혹은 `Consumer` 재시작 등으로 인한 잦은 그룹 변경이 있더라도, 그룹의 안전성을 유지 할 수 있다. 
- `Rebalancing` 지연 감소 : `Reblaancing` 으로 인한 메시지 처리 지연을 최소화 할 수 있다. 

단점을 정리하면 아래와 같다. 

- 유연성 제한 : 고정된 `Member ID` 를 사용하므로 그룹의 변경이 있더라도 `Dynamic Membership` 과 비교해서 유연성이 떨어진다. 새로운 `Consumer` 가 자주 가입하고, 기존 `Consumer` 가 계속해서 탈퇴하는 환경이라면 `Static Membership` 사용은 다시 검토해 봐야할 수 있다. 
- 초기 설정 복잡성 : 고유한 `Member ID` 를 각 `Consumer` 에 할당해야 하므로 초기 설정이 복잡해 질 수 있다. 특히 규모가 큰 시스템일 경우 `Consumer` 의 수가 많으므로 관리에 대한 리소스가 발생 할 수 있다.  
- 장애 복구 지연 : `Consumer` 가 가용 불가 상태가 됐을 떄, 해당 `Consumer` 가 처리 중이던 `Partition` 은 바로 가용한 다른 `Consumer` 에게 할당되지 않는다. 이러한 특징은 장애 복구를 지연시킬 수 있다. 
- `Rebalancing` 민감도 : `Rebalancing` 의 빈도를 줄 일 수 있지만, 발생 할 때 큰 영향을 줄 수 있다. 잘못된 `Member ID` 가 사용되는 경우는 더욱 치명적 일 수 있다. 

#### 처리 과정
`Static Membership` 은 `Consumer` 에게 고유한 `group.instance.id` 를 구성해 정적인 멤버로 색별한다. 
`Group Coordinator` 는 `group.instance.id` 를 내부 `member.id` 와 매핑하고, 
`Consumer` 가 다시 시작하는 경우라도 식별 가능한 아이디가 담긴 `JoinGroup` 요청이 `Group Coordinator` 로 전송되기 때문에 `Rebalancing` 을 수행하지 않고 기존 파티션을 그대로 할당 받는다.  

그리고 `Consumer` 가 종료된 시나리오에서는 `session.timeout.ms` 에 설저된 세션 만료시간까지 `Consumer Group` 에서 해당 식별 가능한 아이디는 제거되지 않는다. 
해당 `Consumer` 가 `session.timeout.ms` 초과해서 다시 `Consumer Group` 에 참여하면 `Rebalancing` 은 트리거되고, 
이내 참여하면 `Rebalancing` 은 수행되지 않기 때문에 다른 `Consumer` 의 메세지 처리에 영향을 주지 않게된다.  

아래는 `Static Membership` 의 과정을 도식화한 그림이다. 
두 `Consumer` 는 동일한 `Consumer Group` 에 속하고 각자 고유한 `group.instance.id` 를 가지고 있다. 
그리고 두 `Consumer` 모두 동일한 `Topic` 에서 서로 다른 `Partition` 을 구독 중이다. 
위 상태에서 `Consumer B` 가 잠시 떠나고 `session.timeout.ms` 이내 다시 `Consumer Group` 에 참여하면 기존 `Partition` 을 그대로 할당 받기 때문에 `Rebalancing` 은 발생하지 않는다.  

kafka-rebalancing-protocol-3.drawio.png

한번 더 강조하지만 `Static Membership` 은 `Consumer` 가 반복해서 재시작 되는 환경이라면 `Rebalancing` 을 줄이고, 
메시지 처리 효율을 높일 수 있는 좋은 방법이다. 
하지만 `session.timeout.ms` 가 만료될 때까지 떠난 `Consumer` 가 소비하던 `Partition` 은 재할당되지 않으므로, 
해당 값이 너무 길게 설정 돼있거나 한다면 오히려 처리 효율이 좋지 않을 수 있다. 
반대로 `Consumer` 재시작에 소요되는 시간 보다 짧게 설정하더라도 호율이 안좋을 수 있기 때문에, 
적절한 최적의 값으로 설정이 필요하다. 




---  
## Reference
[Dynamic vs. Static Consumer Membership in Apache Kafka](https://www.confluent.io/blog/dynamic-vs-static-kafka-consumer-rebalancing/)  
