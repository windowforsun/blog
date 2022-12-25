--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams DSL"
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
    - Kafka Streams
    - Kafka Streams DSL
toc: true
use_math: true
---  

## Streams DSL
`Kafka Streams DSL` 은 `Processor API` 를 사용해서 구현 돼있고, 
대부분의 데이터 처리를 `DSL` 을 사용하면 초보자도 간단하게 표현 할 수 있다. 

`Streams DLS` 을 `Processor API` 와 비교하면 아래와 같다. 

- `Streams DSL` 은 `KStream`, `KTable`, `GlobalKTable` 과 같은 스트림과 테이블의 추상화 구현체르 제공한다. 
- `Streams DSL` 은 `stateless transformation`(`map`, `filter`, ..) 와 `stateful transformation`(`count`, `reduce`, `join`, ..) 에 대한 동작을 함수형 스타일로 제공한다. 

`Streams DLS` 을 사용해서 `Topology` 를 구성하는 절차는 아래와 같다. 
1. `Kafka Topic` 에서 데이터를 읽은 `Input Stream` 을 하나 이상 생성한다. 
2. `Input Stream` 을 처리하는 `Transformation` 을 구성한다. 
3. `Ouput Stream` 을 생성해서 결과를 `Kafka Topic` 에 저장한다. 혹슨 `Interactive queries` 를 사용해서 결과를 `REST API` 와 같은 것으로 노출 시킨다. 

`Streams DSL` 에는 레코드의 흐름을 추상화한 3가지 개념인 `KStream`, `KTable`, `GlobalKTable` 이 있다. 
위 3가지 개념은 `Consumer`, `Producer`, `Processor API` 에는 사용되지 않는 `Streams DSL` 에만 사용되는 개념이다.  

### KStream (input topic -> KStream)
`input topic` 역할을 하는 `Kafka Topic` 에 `KStream` 을 생성해서 데이터를 읽어올 수 있다. 
`KStream` 은 분할된 레코드 스트림이다. 
여기서 분할된 레코드 스트림이란 스트림 애플리케이션이 여러 서버에 실행 된다면, 
각 애플리케이션의 `KStream` 은 `input topic` 의 특정 파티션 데이터로 채워진다. 
분할된 `KStream` 을 하나의 큰 집합으로 본다면 `input topic` 의 모든 파티션의 데이터가 처리된다.  

`KStream` 은 레코드의 흐름을 표현한 것으로 메시지 키와 메시지 값으로 구성돼 있다. 
이를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-1.drawio.png)  


### KTable (input topic -> KTable)
`KTable` 은 `KStream` 과 다르게 메시지 키를 기준으로 묶어 사용한다. 
`KStream` 은 토픽의 모든 레코드를 조회 할 수 있었지만, `KTable` 은 유니크한 메시지 키를 기준으로 
가장 최신 레코드만 조회 가능하다. 
즉 `KTable` 로 데이터를 조회하면 메시지 키를 기준으로 가장 최신에 추가된 레코드의 데이터가 출력된다. 
새로 데이터를 적재할 때 동일한 케시지 키가 있을 경우 데이터가 업데이트되었다고 볼 수 있다.  

`KStream` 과 동일하게 `Kafka Topic` 에 `KTable` 을 생성할 수 있다. 
`KTable` 은 메시지 키의 최신 데이터만 유지하므로 `Changelog Stream` 이라고 해석 할 수 있다. 
메시지 키가 존지하지 않으면 `INSERT`, 이미 동일한 메시지 키가 존재하면 `UPDATE`, 
메시지 키가 존재하지만 메시지 값이 `null` 이라면 `DELETE` 가 수행된다. 
토픽으로 부터 데이터를 읽어올 때 `auto.offset.reset` 속성에 따라 데이터를 읽어오는 위치를 정하게 된다. 
그러므로 `auto.offset.reset` 에 따라 출력되는 결과는 달라질 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-2.drawio.png)  


### GlobalKTable
`GlobalKTable` 은 `KTable` 과 동일하게 메시지 키를 기준으로 묶어 최신 값만 사용한다. 
`KTable` 로 선언된 토픽은 1개 파티션이 1개 태스크에 할당되어 사용된다. 
하지만 `GlobalKTable` 로 선언된 토픽은 모든 파키션 데이터가 각 태스크에 할당되어 사용된다.  

`GlobalKTable` 의 용도를 가장 잘 설명할 수 있는 예는 `KStream` 과 `KTable` 데이터를 `Join` 하는 동작 수행이다. 
`KStream` 과 `KTable` 을 조인하기 위해서는 코파티셔닝(`co-partitioning`)돼야 한다. 
코파티셔닝이란 조인하는 2개 데이터의 파티션 개수가 동일하고 파티셔닝 전략(`partitioning strategy`)을 동일하게 맞추는 작업이다. 
파티션 개수, 파티셔닝 전략이 동일하면 동일한 메시지 키가 동일한 테스크에 들어가는 것을 보장되기 때문에 `KStream` 과 `KTable` 조인이 가능하다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-3.drawio.png)  

문제는 조인을 수행하려면 `KStream`, `KTable` 2개 토픽이 파티션 개수 및 전략이 다를 수 있는데(코파티셔닝 되지 않음), 
이런 상황에서 `Streams Application` 에서는 `TopologyException` 이 발생한다. 
이런 상황에서는 리파티셔닝(repartitioning`) 과정을 거쳐야 하는데, 
리파티셔닝은 새로운 토픽에 새로운 메시지 키를 가지도록 재배열 하는 과정을 의미힌다.  

코파티셔닝 되지 않은 `KStream` 과 `KTable` 을 조인해서 사용하고 싶다면 `KTable` 대신 `GlobalKTable` 을 사용하면, 
위 복잡한 과정없이 간단하게 사용할 수 있다. 
`GlobalKTable` 로 정의된 데이터는 `Streams Application` 의 모든 태스크에 동일하게 공유돼 사용되기 때문에 별도 작업없이 조인 동작이 가능하다.  

하지만 `GlobalKTable` 은 추가적인 부하를 가져다 줄 수 있기 때문에 사용에 주의가 필요하다. 
각 태스크마다 `GlobalKTable` 로 정의된 모든 데이터를 지정하고 사용하기 때문에 
`Streams Application` 의 로컬 스토리지의 사용량 증가, 네트워크, 브로커 부하가 생길수 있다. 
그러므로 용량이 작은 데이터인 경우에만 사용을 하는게 좋다. 
비교적 많은 양의 데이터를 가진 토픽을 조인해야 하는 경우라면 리파티셔닝을 통해 `KTable` 을 하는 것을 권장한다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-4.drawio.png)  


### Streams DSL 옵션
`Streams DSL` 에는 필수 옵션과 선택 옵션이 있다. 
필수 옵션은 반드시 설정해야 하는 옵션이고, 
선택 옵션은 사용자의 설정을 필수로 받지 않는다. 
하지만 선택 옵션은 기본값으로 설정되기 때문에 옵션을 설정할 때 기본값을 잘 파악해서 설정해야 한다.  

아래 설명한 옵션외 추가적인 옵션은 [여기](https://kafka.apache.org/documentation/#streamsconfigs)
에서 확인 가능하다.  



#### 필수 옵션
- `bootstrap.servers` : 프로듀서가 데이터를 전송할 대상 카프카 클러스터에 속한 브로커의 호스트를 1개 이상 작성한다. (이름:포트)
- `application.id` : `Streams Application` 을 구분하기 위한 고유한 아이디를 설정한다. 다른 로직을 가진 애플리케이션이라면 다른 `application.id` 를 가져야 한다. 

#### 선택 옵션
- `default.key.serde` : 레코드의 메시지 키를 직렬화, 역직렬화하는 클래스를 지정한다. 기본값은 바이트 동작인 `Serdes.ByteArray().getClass().getName()` 이다. 
- `default.value.serde` : 레코드의 메시지 값을 직렬화, 역직렬화 하는 클래스를 지정한다. 기본값은 바이트 동작인 `Serdes.ByteArray().getClass().getName()` 이다. 
- `num.streams.threads` : 스크림 프로세싱 실행 시 실행될 스레드 개수를 지정한다. 기본값은 1이다. 
- `state,dir` : `rocksDB` 저장소가 위치할 디렉토리를 지정한다. 기본값은 `/tmp/kafka-streams` 이다. 리얼 환경에서는 `/tmp` 디렉토리가 아닌 별도로 관리되는 디렉토리를 지정해야 안전한 데이터 관리가 가능하다.  



















---  
## Reference
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#transform-a-stream)  