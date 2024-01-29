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

