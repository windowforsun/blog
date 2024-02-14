--- 
layout: single
classes: wide
title: "[Kafka] Kafka Consumer Duplication Patterns"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Consumer 에서 중복 메시지 상황과 해결 할 수 있는 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Consumer
    - Consumer
    - Duplication Patterns
    - Idempotent Consumer
    - Transactional Outbox
    - Kafka Transaction
toc: true
use_math: true
---  

## Duplication Patterns
`Duplication Patterns` 는 중복 이벤트를 관리할 수 있는 `Kafka` 애플리케이션 패턴인 `중복 이벤트 방지 패턴` 을 의미한다. 
중복 이벤트는 `Kafka` 기반으로 분산 메시지를 처리할 때 불기파한 동작이다. 
하지만 `Duplication Patterns` 에서 소개하는 내용을 접목시키면 이러한 중복 이벤트 처리를 최소화할 수 있다.  

### Duplicate events
아래 도식화된 그림은 `Kafka Consumer` 가 `Topic` 으로부터 메시지를 받아 처리하는 과정을 표현한 것이다. 
폴링한 메시지를 바탕으로 `Third party Service` 에 `POST` 로 트리거하고, 
`DB` 에 새로운 레코드를 `INSERT`, 
마지막으로 `Outbound Topic` 에 이벤트를 전달한다.  


![그림 1]({{site.baseurl}}/img/kafka/duplication-patterns-1.drawio.png)


`Kafka Consumer` 가 수행하는 동작을 나열하면 아래와 같다.  

1. `Kafka inbound topic` 으로 부터 메시지 폴링
2. `Third party Service` 에게  `POST` 요청 수행
3. `DB` 에 데이터 `INSERT`
4. 다른 `Kafka outbound topic` 에게 이벤트 전달 

위 `Kafka Consumer` 가 처리 중간에 실패(가용 불가 상태 등)을 가정해보자. 
`Kafka inbound topic` 에서 메시지를 받은 후 실패하는 시점에 따라, 
메시지가 누락될 수 있고, 이후 처리가 중복으로 처리될 수 있다. 
그러므로 구현시 메시지를 소비하고 나서 모든 처리가 완료된 후 메시지가 완전히 사용된 것으로 `Kafka offset` 에 커밋해야 한다. 
이런 과정을 통해 `Kafka Consumer` 가 실패하더라도, 
`Consumer Group` 에 있는 다른 `Consumer` 가 이어서 메시지를 이어서 소비할 수 있고, 
이후 설명하는 `Duplication Patterns` 중 어떤 것을 적용하느냐에 따라 중복으로 수행되는 처리를 방지 할 수 있다.  

### Duplication patterns
이후 설명하는 몇가지 패던을 조합해서 사용하는 것도 가능하다.  

`Idempotent Consumer` 과 `Transactional Outbox` 는 모두 `DB` 트랜잭션 기반이므로 함께 사용 할 수 있다. 
하지만 앞서 언급한 `DB` 트랜잭션을 사용하는 패턴과 `Kafka Transaction` 을 사용하는 패턴은 함께 사용할 수 없다. 
`DB Transaction` 과 `Kafka Transaction` 은 원자적으로 커밋 될 수 없기 때문이다. 
물론 `DB Transaction` 과 `Kafka Transaction` 을 코드상에서 커밋을 2번 수행 할 수는 있지만, 
두 커밋이 원자적으로 완료되는 것을 보장되지 않으며 결과적으로 서로 다른 상태의 리소스가 일관되지 않게 남을 수 있다. 
그러므로 `Idempotent Consumer` 와 `Kafka Transaction` 등을 결합해서 사용하는 것은 불가하다. 

#### Idempotent Consumer
`Kafka` 의 `message id` 필드를 `DB` 에 `Unique` 로 잡아 관리한다. 
`Kafka Consumer` 가 메시지를 폴링하고 가장 먼저 `DB` 에 `message id` 추가 쿼리를 수행하며 트랜잭션을 시작한다. 
만약 해당 `message id` 가 처리된 적이 있다면 롤백이 수행되며 이후 작업들은 처리되지 않을 것이고, 
`message id` 없다면 이후 처리가 수행된다. 
즉 `Kafka Consumer` 의 첫 시작과 완료까지를 `message id` 를 추가하는 트랜잭션안에 두어 중복 이벤트 처리를 방지하는 방식이다.  


#### Transactional Outbox
`Kafka Outbound topic` 에 이벤트를 전달하는 처리를 직접 `Kafka topic` 에 전달하는 것이 아니라, 
별도 `DB` 의 `Outbox table` 에 레코드를 추가하고 이를 `CDC` 로 `Kafka Outbound topic` 이 폴링하도록 구현하는 패턴이다. 
그리고 `Kafka Consumer` 에서 수행하는 `DB` 의 `INSERT` 동작의 `Outbox table` 에 레코드 추가 동작과 트랜잭션을 하나로 묶어 관리한다. 
그러면 해당 이벤트가 이미 `Outbox table` 에 추가된 이벤트라면 롤백으로 이후 `DB INSERT` 의 중복처리를 방지할 수 있고, 
`Outbox table` 에 추가되지 않은 이벤트라면 이후 처리는 정상적으로 수행된다.  

#### Kafka Transaction API
`Kafka Transaction` 은 `exactly-once` 를 보장한다. 
여기서 `exactly-once` 의 의미는 `Kafka` 메시지를 소비하고 처리하고 발생하는 3개 요소를 하나의 트랜잭션으로 묶어 단일 처리를 보장하는 것이다. 

```
consume - process - produce
```  

`Kafka Consumer` 가 위와 같이 메시지를 폴링한 후 처리해서 다른 `Outbound topic` 으로 전달한다고 했을 떄, 
이를 하나의 트랜잭션으로 묶어서 폴링한 메시지가 정상적으로 발생될 때 `offset` 을 커밋해 `consume` 했지만 `produce` 되지 않은 경우를 방지 할 수 있다.  
하지만 기억해야 할 것은 `Kafka Consumer` 실패시 `consum - process - produce` 의 처리는 중복으로 처리 될 수 있음이다.  


### Duplication Patterns Summary
아래 다이어그램은 `Kafka Consumer` 가 메시지 처리 플로우를 도식화 한 것이다.  

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-1.png)

아래 표는 앞서 설명한 `Duplication patterns` 를 적용했을 때 각 처리 과정 실패시 발생할 수 있는 중복 처리를 정리한 내용이다.  

patterns|No failure|failure after POST & INSERT|failure after produce|failure after db committed|Consume time out
---|---|---|---|--|---
no pattern|None|- POST<br>- INSERT|- POST<br>- INSERT<br>- PRODUCE|해당없음|- POST<br>- INSERT<br>-PRODUCE
Idempotent Consumer|None|POST|- POST<br>- PRODUCE|None|None
Transactional Outbox|None|POST|해당없음|- POST<br>- INSERT<br>- PRODUCE|- POST<br>- INSERT<br>- PRODUCE
Idempotent Consumer & Transactional Outbox|None|POST|해당없음|None|None
Kafka Transaction|None|POST|- POST<br>- INSERT|해당없음|- POST<br>- INSERT

`Consme time out` 상황은 `Kafka Consumer` 의 `poll()` 동작이 타임아웃 전까지 수행되지 못해, 
`Kafka Broker` 입장에서는 해당 `Kafka Consumer` 가 가용 불가상태로 판단돼서 `Consumer Group` 중 
다른 `Kafka Consumer` 가 이어서 메시지를 처리 할 수 있도록 할당하는 경우를 의미한다.  

#### no pattern
앞서 설명한 `Duplication patterns` 가 적용되지 않은 일반적인 경우이다. 
각 중복 이벤트 처리가 될 수 있는 경우마다 흐름을 살펴보면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-nopattern.png)


#### Idempotent Consumer
메시지를 식별할 수 있는 값을 `DB` 에 저장해 중복 메시지를 판별하는 경우이다.
각 중복 이벤트 처리가 될 수 있는 경우마다 흐름을 살펴보면 아래와 같다.

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-idempotent-consumer.png)


#### Transactional Outbox
`Outbound topic` 이벤트 전달을 직접 전달하지 않고 `DB` 와 같은 저장소에 추가한 뒤 `CDC` 를 통해 `Outbound` 에 이벤트를 전달하는 경우이다.
각 중복 이벤트 처리가 될 수 있는 경우마다 흐름을 살펴보면 아래와 같다.

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-transaction-outbox-pattern.png)


#### Idempotent Consumer & Transactional Outbox
`Idempotent Consumer` 와 `Transactional Outbox` 를 함께 사용한 경우로, 
메시지를 식별 할 수 있는 값을 `DB` 에 넣는 식으로 메시지의 중복을 판별하면서 
`Outbound topic` 이벤트 전달 또한 `DB` 에 넣고 `CDC` 로 전달하는 경우이다. 
각 중복 이벤트 처리가 될 수 있는 경우마다 흐름을 살펴보면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-idempotent-transactional-outbox-pattern.png)


#### Kafka Transaction
`Outbound topic` 에 이벤트를 직접 전달하지만 `Kafka Transaction` 을 사용해서 `exactly-once` 를 보장하는 경우이다.
각 중복 이벤트 처리가 될 수 있는 경우마다 흐름을 살펴보면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/duplication-pattern-kafka-transaction.png)


---  
## Reference
[Kafka Deduplication Patterns](https://www.lydtechconsulting.com/blog-kafka-deduplication-patterns-part1.html)  
[Idempotent Consumers](https://tugrulbayrak.medium.com/idempotent-consumers-b8629fd361d2)  
