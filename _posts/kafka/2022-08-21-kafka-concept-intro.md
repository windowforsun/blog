--- 
layout: single
classes: wide
title: "[Kafka 개념] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
  - Kafka
  - Concept
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

> Kafka Architecture 사진

#### Message Broker

#### Topic
partition

#### Zookeeper

#### Producer

#### Consumer

#### Consumer Group

#### Replication



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
