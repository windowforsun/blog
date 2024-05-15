--- 
layout: single
classes: wide
title: "[Kafka] Kafka Transaction Exactly Once"
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
    - Transaction
    - Kafka Transaction
    - Exactly-Once
    - Isolation Level
    - Consumer
    - Producer
toc: true
use_math: true
---

## Kafka Transaction Exactly-Once
`Kafka` 는 `Transacitonal API` 을 통해 `exactly-once` 메시징을 제공한다. 
이러한 방법론을 실제 애플리케이션에 적용하기 전에 `exactly-oncd` 가 어떤 것을 의미하고, 
`Kafka Transaction` 이 어떻게 동작하는지, 적용전 고려사항에는 어떠한 것들이 있는지에 대해 알아보고자 한다. 
그리고 추가적으로 `Spring Boot` 기반 애플리케이션을 구현 할때 
`Kafka Transaction` 을 어떠한 구성을 통해 적용할 수 있는지에 대해서도 알아본다.  

`Kafka Transaction` 은 `Kafka` 의 `Exactly-Once` 를 구현하는 주요기능이다. 
하지만 이 기능의 애플리케이션에 대한 적합성은 요구사항, 구성에 따라 달리질 수 있다. 
`Kafka` 관점에서 메시지는 `Exactly-Once` 로 동작할 수 있지만, 
애플리케이션에서 수행하는 `DB`, `REST API` 호출 등에는 중복 처리가 발생 할 수 있다는 의미이다. 
또한 `Kafka Transaction` 을 통해 메시지를 발생하는 `Topic` 을 사용하는 `downstream` 
애플리케이션도 어떤 `isolation.level` 아냐에 따라 커밋 전까지 여러번 메시지를 작성 할 수 있기 때문에 
중복 메시지를 받을 수도 있다는 점도 인지가 필요하다.  

### Exactly-Once
`Kafka Broker` 에서 메시지를 소비하는 `Consumer` 는 `at-least-once` 특성으로 메시지를 소비하기 때문에 최소 한번 메시지 소비를 보장 받을 수 있다. 
일시적인 실패 시나리오에서 메시지는 재전송되어 손실이 발생하지 않기 때문이다. 
하지만 이는 최소 한번은 메시지를 소비할 수 있지만 재전송 과정에서 중복 메시지가 발생할 수 있는 상황이 있다. 
이러한 중복 메시지에 대한 처리는 [Idempotent Consumer]()
를 통해 처리가 필요하다. (하지만 이는 추가적인 오버헤드가 있다.)  

`Kafka` 에서 `exactly-once` 는 여러 단계의 처리 과정이 정확히 한번 수행된다는 것을 의미한다. 
이는 `Inbound Topic` 에서 메시지가 한번 소비되고, 가공 후 `Outbound Topic` 에 그 결과를 한번만 쓴다는 의미이다. 
즉 `Outound Topic` 에서 결과 메시지를 받아 이어서 처리하는 `downstream` 은 결과를 단 한번만 받는 것이 보장된다. 
메시지 처리 과정에서 재전송 등의 동작은 발생 할 수 있어 `Inbound Topic` 에서 메시지가 여러번 소비 될 수는 있지만, 
결과가 쓰여지는 `Outbound Topic` 에 정확히 한번 결과가 쓰여지고 중복 이벤트가 발생하는 결과를 방지 할 수 있다.  

.. 그림 ..


### Failure Case
`Kafka` 는 메시징 스트림상 실패 시나리오에서도 `exactly-once` 를 보장하기 위해 `Transactional API` 를 제공한다. 
`Transactional API` 는 메시지 소비와 생산을 하나의 `Transaction` 으로 묶어 소비한 메시지의 결과가 `Outbound Topic` 에 정상적으로 
쓰여지기 전까지 `Transaction` 의 `commit` 을 하지 않고 이를 통해 `Transaction` 내 메시지 처리는 원자적으로 수행된다. 
`Inbound Topic` 의 메시지 소비와 `Outbound Topic` 의 결과 메시지 쓰기는 함께 성공/실패하기 때문에, 
실패한다면 `Transaction` 은 `Commit` 을 수행하지 않아 타임아웃을 통해 롤백되거나, 메시지 재전송, 트랜잭션 재개를 통한 성공의 상황으로 처리 될 수 있다.  


### Enable Kafka Transaction
`Kafka Transaction` 의 활성화는 `Producer` 에 활성화 설정을 구성해야 하는데, 
`Producer` 에 `Transaction ID` 를 설정함으로써 설정 가능하다. 
위 설정이 완료되면 `Producer` 는 `Transaction` 기반 메시지를 작성하게되는데, 
이는 [Idempotent Producer]()
의 특성을 가지게 된다. 
간단히 설명하면 메시지 생성 중 발생하는 일시적 에러가 중복 메시지로 이어지지 않음을 의미한다. 
`Producer` 에 `Transaction` 을 활성화하면 아래와 같은 흐름으로 동작한다.  

1. `begin Trnasaction` 호출
2. `Producer` 에서 메시지 발행
3. `Consumer` 의 `Offset` 도 `Producer` 에게 전송되어 `Transaction` 에 포함
4. `commitTrnasaciton` 호출을 통해 `Transaction` 완료

`Spring Kafka` 환경이라면 위와 같은 보일러플레이트 코드는 어노테이션을 통해 처리된다. 
`Outbound Topic` 에 메시지를 작성하는 메소드에 `@Transactional` 을 명시적으로 선언하면 된다. 
그리고 `KafkaTransactionManager` 를 `Spring Context` 와 연결하면 `Transaction` 관리를 프레임워크에게 위임할 수 있다.   


### About DB, REST API
`Transactional API` 를 사용한 `Exactly-Once` 는 `Outbound Topic` 의 메시지 쓰기에 대해서만 정확하 한번을 보장한다는 것을 기억해야 한다. 
`Inbound Topic` 에서 메시지를 소비하고, 가공하고, `DB INSERT`, `REST API` 와 결과를 `Outbound Topic` 에 쓰는 애플리케이션을 가정해보자. 
만약 일시적인 에러로 메시지 재전송이 발생한다면 `DB INSERT`, `REST API` 은 여러번 수행될 수 있고, 
`Outbound Topic` 을 소비하는 `downstream` 에서만 중복 메시지 처리를 할 필요가 없는 것이다. 
바로 이러한 이유 때문에 이러한 패턴이 실제 애플리케이션 요구사항을 충족하기에 적합하지 않을 수 있다.  

이러한 외부 시스템과의 연동성에서도 중복 처리를 피하기위해서는 `Kafka Trnsaction` 과 [Idempotent Consumer]() 
를 함께 사용하는 방법이 있다. 
이는 소비한 메시지의 유니크한 아이디를 기반으로 중복 관리가 되는 `DB` 테이블을 사용하는 방법으로, 
`DB Transaction` 을 사용해 메시지 처리를 원자적으로 묶는 방식이다. 
하지만 `Kafka Transaction` 과 `DB Transaction` 은 별개이기 때문에, 
이를 원자적으로 묶기 위해서는 `Spring` 의 `ChainedTransactionManager` 를 통해 분산 트랜잭션을 수행해야 한다. 
분산 트랜잭션은 성능 저하, 코드 복잡성 증가 등의 많은 단점이 있기 때문에 피하는 것이 좋다.  


### Consumer Isolation Level
`Consumer` 는 `Exactly-Once` 를 보장하기 위해 읽기에 대한 `isolation.level` 을 `READ_COMMITTED` 로 설정해야 한다. 
이는 `Kafka Transaction` 을 통해 `Topic` 에 쓰여진 메시지가 `Commit` 으로 표시될 때까지 읽지 않도록 보장하는 설정이다.
(하지만 `Non-Transactional` 메시지는 커밋과는 상관없이 소비 할 수 있다.) 
`Topic Partition` 은 `Commit` 이 수행 될 떄까지 해당 `Consumer Group` 의 `Consumer Instance` 가 추가적인 읽기 수행을 막게된다.  

읽기에 대한 `isolation.level` 은 기본적으로 `READ_UNCOMMITTED` 설정이다. 
그러므로 `Kafka Transaction` 기반 메시지가 쓰여지더라도 `Commit` 과는 상관없이 바로 소비함을 기억하자. 


### Using Kafka Transaction
아래 그림은 `Kafka Transaction` 을 기반으로 메시지가 쓰여지는 과정을 보여준다. 
`Inbound Topic` 에서 메시지를 소비하고, 처리 후, `Outbound Topic` 에 결과를 작성하는 흐름으로 
결과를 발생하는 `Outbound Topic` 은 2개를 사용한다.  

.. 그림 .. 

`Kafka Transaction` 이 사용될 때 `Kafka` 구성으로는 `Outbound Topic` 이 작성되는 `Topic Partition` 과 
`Transaction Coordinator` 가 있다. 
`Transaction Coordinator` 는 `Transaciton Log`를 통해 진행 상황을 추적하며 트랜잭션 수행을 전체적으로 조정하는 역할을 수행한다. 
`Transaction Log` 는 `Kafka` 에서 제공하는 별도의 내부 토픽으로, 
일관성을 보장하기 위해 최소 세 개의 `Kafka Broker` 인스턴스가 구성되어야 한다.  

그리고 `Consumer Coordinator` 는 `Consumer` 들이 소비한 메시지의 `offset` 을 관리한다. 
이런 `offset` 또한 메시지 처리의 원자성을 보장하기 위해 트랜잭션 내에 `offset` 갱신이 이뤄져야 한다. 
일반적인 상황에서 `Consumer Coorindator` 에게 `offset` 갱신 요청은 `Consumer` 의 책임이지만, 
`Kafka Transaction` 가 활성화된 상태에서는 동일 트랜잭션 내에 이를 포함시키기 위해 `Producer` 가 직접 `Consumer Coorindator` 에게 
요청을 수행한다.  

아래 그림은 위 다이어그램이 수행하는 동일 처리를 각 컴포넌트로 도식화 한것이다. 

.. 그림 ..  

위 그림은 `Producer` 가 두 `Topic Partition` 에 `m1`, `m2` 이라는 2개의 메시지를 작성하는 상황이다. 

아래는 `Producer` 가 수행하는 동작을 표현한 것이다. 

P-1. `Producer` 는 트랜잭션 초기화를 위해 `Transaction Coordinator` 에게 초기화 요청을 보낸다. 
이 요청에 `producerId` 와 `transactionId` 가 함께 전달되고, 요청을 받은 `Transaction Coordinator` 는
트랜잭션이 초가화 됐다는 것을 `Transaction Log` 에 작성한다.  

P-2. `Prodocuer` 는 두 결과 메시지 `m1`, `m2` 를 각 `Topic Partition` 인 `tp1`, `tp2` 에 작성한다. 
그리고 `Consumer Coordinator` 에게 `Consumer` 의 `offset` 갱신을 요청하면 이를 `tp1-offset-m1`, `tp2-offset-m2` 와 같이 표시한다.

> `Producer` 와 `Transaction Coordinator` 와의 트랜잭션 상태 추적에 대한 추가 호출이 수행되지만, 그림은 이는 표현하지 않았다. 

P-3. `Producer` 는 트랜잭션 커밋을 수행하기 위해 `Transaction Coordinator` 에게 관련 요청을 수행한다. 

아래는 `Transaction Coordinator` 가 커밋 요청 단계부터 수행하는 동작을 표현하 것이다. 

TC-1. `Transaction Coordinator` 는 트랜잭션 커밋 준비를 위해 `Prepare` 이라는 로그를 작성한다. 
이 시점 부터 트랜잭션은 커밋된다.  

TC-2. `Transaction Coordinator` 는 메시지를 작성한 `tp1`, `tp2` 에 `commit marker` 를 작성하고, 
`Consumer Offset` 에도 `commit marker` 를 작성해 해당 메시지들이 포함된 트랜잭션이 커밋 됐음을 마킹한다.  

TC-3. `Transaction Coordinator` 는 트랜잭션 커밋 로그를 남긴다. 

최종적으로 트랜잭션이 커밋되어 완료 됐고, `READ_COMMITTED` 로 `isolation.level` 이 설정된 `downstream` 은 메시지를 소비할 수 있는 상태가 된다.  
