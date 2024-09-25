--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Transaction & Exactly-Once"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 이 Transaction 과 Exactly-Once 를 통해 스트림 처리의 내구성과 일관성을 보장하는 방식에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Transaction
    - Kafka Transaction
    - Exactly-Once
    - State Store
    - Changelog
toc: true
use_math: true
---  


## Kafka Streams Transactions
`Kafka Transactions` 는 메시지의 흐름 `Consume-Process-Produce` 인 3단계 과정이 한 번만 발생하는 것을 보장한다. 
이것을 `Kafka` 에서는 `Exactly-Once Messaging` 이라고 표현한다. 
[Kafka Transactions]({{site.baseurl}}{% link _posts/kafka/2024-05-05-kafka-practice-kafka-transaction-exactly-once.md %})
에서 알아본 일반적인 `Kafka Consumer, Producer` 를 사용하는 애플리케이션 뿐만 아니라, 
`Kafka Streams API` 를 사용하는 `Kafka Streams Application` 에서도 
`Kafka Transactions` 를 활성화해 각 메시지의 처음부터 끝까지의 흐름이 정확히 한 번 처리되고, 
상태가 내구성 있고 일관되도록 할 수 있다.  

[Kafka Streams Spring Demo]({{site.baseurl}}{% link _posts/kafka/2024-08-16-kafka-practice-kafka-streams-spring-demo.md %})
에서 알아본 것과 같이 `Kafka Streams Application` 은 
`Kafka Consumer/Producer` 를 바탕으로 구현되기 때문에 
`Kafka Transactions` 를 기존 방식화 동일하게 활성화 해준다면 동일한 효과를 얻을 수 있다. 
하지만 `Kafka Streams` 에서는 상태 저장과 같은 추가적인 동작이 있으므로 고민해야 할 부분이 더 있다. 
상태 저장 처리가 실패 했을 때 메시지 분실은 발생하지 않지만, 
메시지가 상태 저장소에 중복 쓰기를 발생시키지 않도록 하는 부분도 필요하다.   


### Kafka Transactional Flow
아래 그림은 `Kafka Streams` 이 `Kafka Transaction` 을 사용해서  `Inbound Topic` 에서 메시지를 소비하고, 상태 저장소에 저장하고, 
`Outbound Topic` 에 메시지를 전송하는 흐름을 보여주고 있다. 
이는 `Kafka Transaction` 처리를 수행해주는 `Kafka Broker` 에 위치하는 `Transaction Coordinator` 를 바탕으로 처리됨을 볼 수 있다.   


![그림 1]({{site.baseurl}}/img/kafka/kafka-transaction-exactly-once-1.drawio.png)


1. `Kafka Streams` 는 트랜잭션 시작을 알리기 위해 `Transaction Coordinator` 에게 `begin transaction` 요청을 전송한다. 
2. `Inbound Topic` 에서 `Source Processor` 는 메시지를 소비하고 처리한다.  
3. 스트림 상태를 저장하는 처리는(집계, 윈도우, ..) `State Store` 에 저장된다. 
4. 스트림 처리가 완료되면 `Sink Processor` 는 `Outbound Topic` 에 메시지를 전송한다. 
5. `State Store` 의 변경 내용은 `Changelog Topic` 에 기록된다. 
6. `Consumer Offsets Topic` 에 처리가 왼료된 메시지의 오프셋과 처리가 완료됨을 기록한다. 
7. `Kafka Streams` 는 트랜잭션을 커밋하기 위해 `Transaction Coordinator` 에게 `commit transaction` 요청을 전송한다. 
8. `Transaction Coordinator` 는 `Transaction Log Topic` 에 `prepare commit` 를 기록한다. 이 시점 부터는 실패가 발생하더라도 트랜잭션은 완료로 처리된다. 
9. 이어서 `Transaction Coorindator` 는 `Outbound Topic`, `Changelog Topic`, `Consumer Offset Topic` 에 커밋 마커를 기록한다.
`READ_COMMITTED` 로 설정된 `downstream consumer` 는 커밋 마커가 기록된 시점 부터 메시지 소비가 가능하다. 
10. 마지막으로 `Transaction Coordinator` 는 `Transaction Topic` 에 `complete commit transaction` 레코드를 기록한다. 
이 시점 부터 `Producer` 는 다음 트랜잭션 시작이 가능하다.  


위 도식화된 그림에서는 제외 되었지만 `Kafka Streams Application` 이 처음 시작하면, 
`Producer` 는 `Transaction Coordinator` 에 `Transaction Id` 를 등록하고 
이를 통해 `Transaction Coordinator` 는 고유한 `Producer Id` 를 할당해서 `Transaction Topic` 에 이를 기록한다.  

또한 `Outbound Topic` 과 `Consumer Offset Topic` 에 쓰기를 수행하기 전에, 
`Kafka Streams` 는 `Transaction Coordinator` 에 `add partition` 요청을 보내
해당 파티션들이 트랜잭션에 포함시키기 위해 `Transaction Log Topic` 에 기록한다.  

아래는 다이어그램은 위 내용까지 포함해 `Kafka Streams` 가 `Kafka Transaction` 을 수행하는 전체 흐름을 보여준다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-transactions-2.png)


`Kafka Streams` 처리 중 오류가 발생하게 되면, 
`Transaction Coorindator` 에 `abort transaction` 요청을 보내 `prepare abort` 가 
`Transaction Log Topic` 에 기록될 수 있도록 한다. 
그리고 `Outbound Topic`, `Changelog Topic`, `Consumer Offset Topic` 각각에 `abort marker` 를 기록해서 
`READ_COMMITED` 로 설정된 `downstream consumer` 는 이런 레코드를 소비하지 않도록 한다. 
최종적으로 `Transaction Coordinator` 는 `Transaction Log Topic` 에 `abort complete transaction` 을 기록해서 
실패 처리를 마무리한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-transactions-3.png)


### Changelog Topics(State Store)
`Kafka Streams State Store` 에서 `Changelog Topic` 은 `State Store` 의 변경사항을 기록해 
이후 복원력을 제공한다. 
`Kafka Streams` 는 메시지 처리가 완료되면 모든 변경 사항인 `State Store` 변경을 포함해서 
`Changelog Topic`, `Outbound tOpic`, `Consumer Offset Topic`
에 대해 트랜잭션을 커밋한다. 이 과정에서 나열한 모든 변경사항이 원자적으로 반영되기 때문에 이런 토픽 쓰기 작업은 모두 성공하거나 실패함을 보장 한다.  

위 설명과 같이 `Kafka Streams` 에서 `Changelog Topic` 은 기존 `Kafka Transaction` 의 `exactly-once` 와 
원자성을 보장해주기 때문에 이후 애플리케이션 재시작과 같은 상황에서 `Changelog Topic` 을 통해 `exactly-once` 특성이 적용된 상태 저장소를 복원 할 수 있게 된다.  



### Checkpoint File(State Store)
`Kafka Streams State Store` 에서 `Checkpoint File` 은 상태 저장소의 복구 시점 정보를(`offset`) 제공하는 역할을 한다. 
`Kafka Transaction` 을 사용하지 않는 경우 `Checkpoint File` 은 로컬 상태 정보의 오프셋을 추적하고,
애플리케이션이 재구동 됐을 때 상태 정보 재구축이 필요하다면 
`Checkpoint File` 의 오프셋의 정보를 활용해 `Changelog Topic` 에서 상태정보를 조회 할 수 있다.  

하지만 `Kafka Transaction` 을 사용한다면 `Checkpoint File` 의 활용의 방식이 달라진다. 
먼저 `Checkpoint File` 은 매번 로컬 상태 정보의 오프셋을 추적하지 않고, 
상태 정보를 안정적으로 관리하기 위해 `Rebalancing`, `Streams Task Shutdown` 의 시점에 업데이트 된다. 
즉 트랜잭션이 커밋되는 시점에는 `Changelog Topic` 에만 기록되고 `Checkpoint File` 에는 기록되지 않는다. 
이렇게 트랜잭션과 별도로 `Checkpoint File` 을 관리하는 이유는 `Checkpoint File` 은 로컬 상태 저장소의 복구 포인트(`offset`)을 저장하는 목적이기 때문이다. 
그러므로 `Rebalancing`, `Streams Task Shutdown` 과 같은 명시적인 로컬 상태 저장소 복구가 필요한 시점에만 저장해 옳바른 복구가 가능하도록 하는 것이다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-transactions-4.png)

### Failure Scenarios
`Kafka Streams` 에서 `Kafka Transaction` 을 사용하면 장애 상황에서도 `exactly-once` 메시징을 보장한다. 
`Kafka Transaction` 을 사용하지 않는경우 `Kafka Streams` 는 `at-least-once` 메시징을 보장해서 메시지 손실을 막는다. 
하지만 이는 `Outobund Topic` 및 `State Store` 에서 중복 처리가 발생 할 수 있다. 
만약 `State Store` 에 쓰고 `Outbound Topic` 에 메시지를 발송했지만 `Consumer Offset Topic` 작성전 장애가 발생하면, 
`State Store` 와 `Outbound Topic` 에는 동일한 메시지 처리가 수행 될 것이다.  

`Kafka Transaction` 을 사용한다면 데이터 손실 방지뿐만 아니라, `State Store`, `Outbound Topic` 의 중복쓰기 까지 방지 할 수 있다. 
아래 다이러그림은 `State Store`, `Outbound Topic` 에 메시지를 쓰고 `Consumer Offset Topic` 작성 전 실패가 발생하는 상황을 보여주고 있다. 
각 `Task` 는 `Producer` 에 설정된 `Transaction Id` 를 갖고 처리된다. 
실패 후 `Task` 가 재사작 되면 위 `Transaction Id` 를 `Transaction Coordinator` 에게 전달해 `init transaction` 요청을 수행한다. 
`Transaction Coorindator` 는 `Transaction Id` 와 관련된 모든 미완료 트랜잭션을 중단하고, 이전에 해당 트랜잭션으로 작성된 토픽 메시지에 `abort marker` 를 기록한다. 
이 상태가 되면 `Changelog Topic` 과 `Outbound Topic` 은 일관된 상태가 되며 상태 저장소 복구르 진행 할 수 있다. 
그 후 `Kafka Streams` 는 `Consumer Offset Topic` 에서 마지막으로 커밋된 오프셋부터 다음 배치 이벤트를 진행하는 방식으로 `exactly-onec` 메시징을 보장한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-transactions-5.png)


앞서 언급했던 것과 같이 `Transaction Coordinator` 가 `Transaction Log` 에 `prepare commit` 을 기록한 시점 부터는, 
어떤 실패 시나리오에서도 트랜잭션은 완료된다. 
아래 다이어그램은 해당 시나리오의 과정을 보여준다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-transactions-6.png)


### Performance
`Kafka Streams` 에서 `Kafka Transaction` 을 적용해 `exactly-once` 메시징을 보장받는 것은 성능과 절충이 필요하다. 
앞선 설명과 다이어그램에서 보았던 것과 같이 `Kafka Transaction` 적용을 위해서는 추가적인 로그 쓰기 등과 같은 처리가 수반되기 때문이다. 
메시지 생산 입장에서 성능을 끌어 올리기 위해서는 많은 수의 작은 트랜잭션 보다는 적은 수의 큰 트랜잭션이 성능적으로 유리하다.  

하지만 `Kafka Application` 을 개발할 때는 메시지 생산에 대한 성능만 고민해서는 안되고, 
`downstream consumer` 측면도 고민이 필요하다. 
그 이유는 `READ_COMMITTED` 로 설정된 `downstream cosnumer` 는 커밋이 완료된 메시지만 소비할 수 있기 때문이다. 
메시지 생산과 반대로 메시지 소비 입장에서는 `commit.interval.ms` 값을 낮춰 더 자주 작은 트랜잭션을 발생시키는 것으로 소비 지연시간 개선이 가능하다. 
하지만 이는 앞서 언급한 것과 같이 메시지 생산 입장에서 처리량에 영향이 가기 때문에 절충안으로 튜닝작업이 필요하다.  

추가적으로 `exactly-once` 메시징을 보장하는 방식과 보장하지 않는 방식의 성능 차이는 3% 정도로 간주된다고 알려져 있다.  


### Configuration
`Kafka Streams` 에서 `Kafka Transaction` 을 활성화해 `exactly-once` 메시징은 보장 받는 설정은 
`processing.guarantee` 옵션을 `exactly_once` 로 설정하는 것이다. 
`Kafka Streams 3.x` 버전의 경우 성능과 파티션/작업의 확장성을 보다 향상시킨 `exactly_once_v2` 도 도입되어 사용 할 수 있다. 
참고로 해당 옵션의 기본 값은 `at_least_once` 이다. 
또한 `Kafka Transaction` 과 마찬가지로 정확한 `exactly-once` 를 보장받기 위해서는 최소 3개의 `Kafka Broker` 로 클러스터가 구성돼야 한다.  


---  
## Reference
[Kafka Streams: Transactions & Exactly-Once Messaging](https://www.lydtechconsulting.com/blog-kafka-streams-transactions.html)  
[Exactly Once Semantics](https://docs.confluent.io/platform/current/streams/architecture.html#exactly-once-semantics-eos)  
[exactly_once processing guarantee deprecated](https://docs.confluent.io/platform/current/streams/upgrade-guide.html#improved-exactly-once-processing)  



