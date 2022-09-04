--- 
layout: single
classes: wide
title: "[Kafka 개념] Kafka 구조와 기본 개념"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: '고성능 메시지 브로커인 Apache Kafka 의 아키텍쳐와 구성 요소 및 기본 개념에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
  - Kafka
  - Concept
  - Kafka
  - Message Queue
  - Message Broker
  - MSA
  - Producer
  - Consumer
toc: true
use_math: true
---  

## Kafka
`Apache Kafka` 는 아파치에서 스칼라로 개발한 오픈소스 메시지 브로커 프로젝트로 `pub/sub` 모델의 메시지 큐를 지원한다.
`MSA` 에서 이와 같은 메시지 브로커는 `Message Bakcing Service` 로 동작하고,
메시지 처리 동작을 통해 비동기 애플리케이션, DB 동기화, 트랜잭션, `pub/sub` 등의 다양한 방식으로 사용될 수 있다.

대표적인 메시지 브로커는 `Kafka` 와 함께 `RabbitMQ` 가 있는데 이둘은 다소 큰 차이를 보여준다.
`RabbitMQ` 는 신뢰도 높은 메시지 브로커로 알려져 있다.
장애 발생시 데이터 복원에 용이하고, 반드시 한번의 전송을 보장한다.

하지만 이러한 `RabbitMQ` 는 `Kafka` 와 비교 했을 때 성능면에서 떨어진다는 점이 있다.
`Kafka` 는 대용량 실시간 처리에 특화돼 있고, 대량의 `Batch Job` 에 대한 일괄처리에 용이하다.


![그림 1]({{site.baseurl}}/img/kafka/concept-intro-1.png)

`Kafka` 는 다양한 애플리케이션에서 전달 되는 메시지를 데이터 스트림 파이프라인을 통해 실시간으로 관리하고 보내기 위한
분산 스트리밍 플랫폼이다.
데이터를 생산하는 애프리케이션과 소비하는 애플리케이션의 중간 중재자 역할로,
전송 제어, 처리관리 역할을 수행할 수 있다.
이런 `Kafak` 의 시스템은 여러 노드로 함께 구성돼 클러스터기반으로 운용되어 안정성과 성능을 더할 수 있다.


### Kafka Architecture(구성요소)
`Kafka` 의 `Architecture` 는 크게 `Kafka` 와 `Zookeeper` 로 구분할 수 있다.
이 두 모듈은 독립적인 구성으로 `Zookeeper` 는 분산 애플리케이션의 데이터 관리용도로 주로 메시지 큐 데이터 관리를 담당하고,
`Kafka` 는 메시지를 `TCP` 로 전송하기 위한 브로커를 제공하는 메시지 브로커 역할이다.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-4.drawio.png)


#### Message Broker
`Kafka` 클러스터는 로드밸런싱과 가용성을 위해 3개의 노드로 `Message Broker` 를 구성한다.
그리고 `Kafka Broker` 는 상태 비 저장이기 때문에 `Zookeeper` 를 사용해 클러스터 상태를 유지한다.


#### Topic
`Topic` 은 `Kafka` 에서 메시지를 주고 받기위해 사용되는 `Key` 와 같은 역할이다.
`Topic` 은 대용량 처리를 위해 여러개의 `Partition` 으로 나뉠 수 있고,
이를 통해 메시지를 병렬로 처리해서 성능을 향상시킬 수 있다.
`Partition` 에는 메시지가 누적돼 쌓이게 되고,
메시지의 정보를 `offset`(현재위치)과 비교해서 메시지 수신 여부를 결정한다.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-5.png)

`Topic` 에서 `Producer` 와 `Partition`, `Consumer` 의 관계는 아래와 같다.
- `Partition` 과 `Consumer` 는 `1:1 매핑되기 때문에 동일한 개수로 구성이 필요하다.
- `Partition` 이 여러개일 경우 순서 보장이 중요하다.
  그러므로 `hash key` 와 같은 순서 보장을 위한 API를 사용해서 동일 `Partition` 에 쌓이도록 해서 동일 `Consumer` 가 메시지를 받도록 해야 한다.
- `Producer` 는 `Zookeeper` 를 통해 `Message Broker` 의 `ID` 를 전달 받는다.
- 위에서 전달 받은 `Message Broker` 에게 메시지를 전송한다.
  전송과정에서 `Topic`, `Partition`, `Message` 단위로 나눠지고 병렬 처리를 위해 하나 이상의 `Partition` 으로 나눠 진다.
- `Consumer` 는 `offset` 정보를 `Zookeeper` 로 부터 전달 받고 누적된 `offset` 만큼 해당 `Topic` 의 `Paritition` 으로부터 메시지를 구독한다. 그리고 메시지 수신 만큼 `offset` 를 갱신한다.

#### Zookeeper
`Zookeeper` 는 `Kafka Broker` 를 관리하고 조정하는 역할을 한다.
`kafka Broker` 와 `1:1` 로 구성이 필요하기 때문에 3개의 노드로 `Message Broker` 가 구성돼 있다면,
`Zookeeper` 또한 3개의 노드로 구성해야 한다.
`Zookeeper` 가 하는 일을 나열하면 아래와 같다.
- `Kafka Message Broker` 신규 생성 알림
- `Kafka Message Broker` 실패 알림

`Producer` 와 `Consumer` 는 `Zookeeper` 에게 위와 같은 알림 받아 다른 `Message Broker` 와 작업을 조정하게 된다.

#### Producer
`Producer` 는 `Message Broker` 에게 메시지를 `Push` 한다.
새로운 `Message Broker` 가 시작되면 모든 `Producer` 는 자동으로 새로운 `Message Broker` 에게 메시지를 보낸다.
`Kafka` 의 `Producer` 는 별도의 승인 없이 `Message Broker` 가 처리 할 수 있는 한 빨리 메시지를 보낸다.

#### Consumer
`Consumer` 는 `Producer` 가 `Message Broker` 에게 `Push` 한 메시지를 `Pull` 한다.
이때 `Kafka Broker` 는 상태를 저장하지 않기 때문에 `Consumer` 가 `Partition` 의 `offset` 을 사용해서 자신이 `Pull` 한 메시지의 수를 유지해야 한다.
`Consumer` 가 특정 `offset` 값을 유지하고 있다는 것은 `offset` 값 보다 작은(이전) 메시지들은 모두 사용한 상태임을 의미한다.
그리고 `Consumer` 는 `offset` 값만 제공하면 `Partition` 에서 원하는 지점으로 되감거나 건너 뛸 수 있다.
`Consumer` 의 `offset` 값은 `Zookeeper` 에서 알려준다.

#### Consumer Group
`Producer` 에서 `Push` 한 메시지를 여러 `Parition` 에 저장된다.
이때 `Consumer` 입장에서도 여러 `Consumer` 가 메시지를 `Pull` 하는 것이 효율적일 것이다.
하나의 `Topic` 을 소비하는 여러 `Consumer` 들의 모음을 `Consumer Group` 이라고 한다.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-6.png)

이렇게 효율적으로 메시지를 소비할 수 있는 `Consumer Group` 에는 아래와 같은 룰이 있다.
`Topic` 의 `Partition` 의 수는 `Consumer Group` 을 구성하는 `Consumer` 보다 항상 같거나 많아야 한다는 점이다.
만약 `Partition 1` 을 `Consumer Group` 에 포함되는 `Consumer 1` 이 구독하고 있다면 `Partition 1` 에는 동일 그룹내 다른 `Consumer` 들은 구독할 수 없다는 것을 의미한다.
더 쉬운 이해를 위해 아래 그림을 보자.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-7.png)

`4:3`, `3:3` 인 경우는 `Partition` 의 수가 같거나 더 많기 때문에 `offset` 정보를 통해 순차적으로 데이터를 소비하게 된다.
문제가 되는 경우는 `3:4` 인 경우로 `Consumer Group` 내 `Consumer` 수가 더 많은 경우이다.
이때 `Consumer 4` 는 아무런 메시지도 받지 못하고 놀고 있기만 하게 된다.

하지만 `Consumer Group` 을 병렬 수신외 다른 목적으로도 사용할 수 있는데 바로 `Failover` 와 같은 목적으로도 사용할 수 있다.
`Consumer Group` 내 특정 `Consumer` 에게 문제가 생겼을 때 다른 `Consumer` 가 이를 대체해서 역할을 수행하도록 할 수 있다.
즉 이는 `Rebalancing` 을 통해 장애상황을 극복하는 것을 의미한다.


#### Replication
`Kafka` 에서는 `Replication` 수를 임의로 정해서 `Topic` 생성이 가능하다.
`replication-factor` 를 3으로 지정했다면 `Replication` 수는 3이 된다.

`Kafka Cluster` 에 아래와 같이 3개의 `Message Broker` 와 3개의 `Topic` 이 있다고 가정해본다.
이때 각 `Topic` 에 대한 `Replication` 수가 아래와 같을 때 다음과 같이 배치 될 수 있다.

Topic 이름|Replication 수
---|---
Topic-1|1
Topic-2|2
Topic-3|3

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-8.png)

`Replication` 의 용도는 `RDB` 에서 필요한 상황과 비슷하다.
특정 `Message Broker` 에 문제가 발생했을 때,
다른 `Message Broker` 가 즉각적으로 대신역할을 수행할 수 있도록 하기 위함이다.


#### Replication - leader, follower
`Replication` 의 구성요소는 `leader` 와 `follower` 가 있다.
`leader` 는 말그대로 대표의 역할로 `Topic` 에 대한 모든 데이터의 `Read/Write` 는 오직 `leader` 에서 이뤄진다.
`follower` 는 보조의 역할로 `leader` 와 `sync` 를 유지하면서 `leader` 에게 문제가 생겼을 때 `follower` 중 하나가 `leader` 의 역할을 하게 된다.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-9.png)

위 처럼 구성된 상태에서 `Broker 2` 에 장애가 발생하게 된다면,
`Topic-2` 에서 새로운 `leader` 가 필요하기 때문에 `Broker 1` 의 `follower` 가 아래처럼 `leader` 가 된다.

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-10.png)

`Broker 1` 에 있던 기존 `Topic-2` 의 `follower` 에는 복제된 데이터가 있기 때문에,
메시지의 유실없이 `leader` 역할을 이어서 수행할 수 있다는 장점이 있다.
하지만 이는 역설적으로 `sync` 에 대한 추가적인 네트워크 등의 비용이 들어간 다는 점이 있기 때문에,
`ack` 옵션을 활용해서 성능과 데이터의 중도에 맞춰 세부 설정이 필요할 수 있다.

`ack 옵션`
- `0` : `Producer` 는 `Kafka Server` 로 부터 어떤 `ack` 도 기다리지 않는다. (유실율은 가장 높지만, 처리량도 가장 높다.)
- `1` : `leader` 는 데이터를 기록, 모든 `follower` 는 확인하지 않는다. (기본 설정 값)
- `-1`(all) : 모든 `ISR` 확인, 무손실



### 메시징 모델과 Kafka
`Message Backing Service` 모델로는 `Queue` 와 `pub/sub` 이 있다.

- Queue : 단일 송신자가 단일 수신자에게 데이터를 전송하는 방법이다.
  `Queue` 의 메시지는 한명의 수신자에 의해 한번만 읽혀질 수 있고, 읽혀진 메시지는 제거된다.
  이와 같은 `Queue` 방식은 `Event-Driven` 방식에 적합하다.

- pub/sub : 여러 송신자가 메시지를 발행하고 여러 수신자가 구독하는 방식이다.
  모든 메시지는 `Topic` 을 중심으로 구독하는 모든 수신자들에게 전송가능하다.
  실질적으로는 송신자가 `Topic` 에 메시지를 전송하면 수신하는 `Polling` 하는 방식으로 구성돼 있다.


여기서 `Kafka` 는 `pub/sub` 구조를 구현했다.
대용량 처리 성능과 실시간 처리를 위해 스트리밍 데이터를 전송할 수 있도록 `Queue` 와 `pub/shb` 의 기본 컨셉을 구현하면서도,
각 모델이 가지는 단점을 극복하는 방안으로 두가지를 통합하는 유연성을 제공한다.

### Kafka 메시징 컨셉
위에서 살펴본 것처럼 `Kafka` 는 `Queue` 와 `pub/sub` 의 장점을 모두 제공한다.
기본적으로는 `pub/sub` 을 구현하지만 `Queue` 구조를 동시에 사용할 수 있다.

#### Kafka Queue

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-2.png)

> CG1 ~ CG2 는 소비자 그룹, P0 ~ P3은 단일 `Kafka` 에 구성된 파티션을 의미한다.

`Kafka` 에서 `Queue` 방식을 구현하기 위해서는 여러 소비자가 하나의 `Consumer Group` 에 포함되고,
`Topic` 을 구독하도록 하면 된다.
발행된 메시지는 `Consumer Group` 에서 하나의 소비자에게만 전달된다.
기존 `Queue` 와 다른점은 발행된 메시지는 이후 소비되도라도 `broker` 내부 토픽에서 사라지지 않고 보유된다는 점이 있다.

`Consumer Group` 에 포함된 모든 소비자들은 생산자로 부터 발행된 메시지의 소비작업을 공유한다.
이러한 특징으로 만약 한 소비자가 생산 속도를 따라가지 못하는 경우 소비자 인스턴스를 추가하는 식으로 수평확장 방법을 통해 해결 할 수 있다.

한 `Topic` 의 병렬처리 최대양은 `Partition` 수에 의해 정해진다.
`Topic` 에 4개의 `Partition` 있고 2개의 소비자가 있다면,
소비자마다 2개의 `Partition` 을 할당 받게 된다.
필요한 경우 소비자를 `Consumer Group` 에 `Partition` 수 만큼 추가해 전체 처리량을 늘릴 수 있다.
이럴 경우 소비자당 하나의 파티션이 할당되게 된다.


#### Kafka Topic

![그림 1]({{site.baseurl}}/img/kafka/concept-intro-3.png)

`pub/sub` 시스템은 `Topic` 혹은 `Channel` 에 대해 모든 구독자 혹은 소비자에게 메시지를 전달하는 것을 의미한다.
이는 `Queue` 에서 여러 소비자가 있을 때 서로 다른 데이터를 순서대로 전달받는 것과는 상관되는 구조이다.

위 그림에서 보이는 것처럼 `Consumer Group A` 와 `Consumer Group B` 는 서로 다른 애플리케이션이다.
하지만 이 두 애플리케이션은 `Topic` 에서 생산되는 모든 데이터를 함께 수신하게 된다.


---
## Reference
[Kafka Documentation](https://kafka.apache.org/documentation/)  
[Kafka: Is it a Topic or a Queue?](https://abhishek1987.medium.com/kafka-is-it-a-topic-or-a-queue-30c85386afd6)  
[Kafka: Kafka Architecture and Its Fundamental Concepts](https://data-flair.training/blogs/kafka-architecture/)  
[Kafka: Apache Kafka Queue 101: Messaging Made Easy](https://hevodata.com/learn/kafka-queue/)  
