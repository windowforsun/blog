--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Spring Boot"
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


## Kafka Streams Transactions
`Kafka Transactions` 는 메시지의 흐름 `Consume-Process-Produce` 인 3단계 과정이 한 번만 발생하는 것을 보장한다. 
이것을 `Kafka` 에서는 `Exactly-Once Messaging` 이라고 표현한다. 
[Kafka Transactions]()
에서 알아본 일반적인 `Kafka Consumer, Producer` 를 사용하는 애플리케이션 뿐만 아니라, 
`Kafka Streams API` 를 사용하는 `Kafka Streams Application` 에서도 
`Kafka Transactions` 를 활성화해 각 메시지의 처음부터 끝까지의 흐름이 정확히 한 번 처리되고, 
상태가 내구성 있고 일관되도록 할 수 있다.  

[Kafka Streams Spring Demo]()
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


.. 그림 ..

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


.. 그림 ..


`Kafka Streams` 처리 중 오류가 발생하게 되면, 
`Transaction Coorindator` 에 `abort transaction` 요청을 보내 `prepare abort` 가 
`Transaction Log Topic` 에 기록될 수 있도록 한다. 
그리고 `Outbound Topic`, `Changelog Topic`, `Consumer Offset Topic` 각각에 `abort marker` 를 기록해서 
`READ_COMMITED` 로 설정된 `downstream consumer` 는 이런 레코드를 소비하지 않도록 한다. 
최종적으로 `Transaction Coordinator` 는 `Transaction Log Topic` 에 `abort complete transaction` 을 기록해서 
실패 처리를 마무리한다.  


.. 그림 .. 
