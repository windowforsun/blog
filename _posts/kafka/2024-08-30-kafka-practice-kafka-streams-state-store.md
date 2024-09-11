--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams State Store"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 상태를 저장하고 관리할 수 있는 State Store 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - State Store
    - RocksDB
    - Changelog Topics
toc: true
use_math: true
---  

## Kafka Streams State Store
`Kafka Streams State Store` 은 스트림 처리 애플리케이션 내에서 상태 정보를 저장하고 관리하는 구성 요소이다. 
이는 각기 다른 시간에 저장하는 이벤트를 캡쳐해서 저장함으로써 이벤트 그룹화를 가능하게 한다. 
이런 그룹화를 통해 `join`, `reduce`, `aggregate` 등 과 같은 연산을 수행 할 수 있다. 
그리고 `State Store` 은 `Persistent State Store` 혹은 `In-Memory State Store` 를 지원하고, 
자체적으로 `Kafka` 의 변경 로그 토픽(`changelog topics`) 과 통합되어 고장 내성을 갖는다. 
그러므로 상태 저장소의 모든 변경 사항은 변공 로그 토픽에 기록되어, 시스템 장애시 상태를 복구 할 수 있다.  

`State Store` 의 사용의 몇가지 예는 아래와 같다. 

- `Aggregate` : 스트림의 데이터를 시간별, 카테고리별 등 다양한 기준으로 집계해서 상태 저장소에 저장한다. 
- `Joins` : 두 스트림 또는 스트림과 테이블 간 조인을 수행할 떄, 관련 데이터를 상태 저장소에 저장하여 두 데이터를 결합한다. 
- `Windowing` : 특정 시간 범위의 데이터를 분석하는 목적으로 이벤트를 시간별 또는 세션별로 그룹화할 때 사용한다. 
- `Reduce` : 스트림 데이터를 특정 키 값에 따라 축소하거나 합칠 때 사용한다. 


### RocksDB
`Kafka Streams` 는 기본적으로 `State Store` 로 `RocksDB` 를 사용한다. 
이는 `Embedded State Store` 로 메모리가 아닌 로컬 디스크에 데이터를 저장하므로 별도의 네트워크 호출이 필요하지 않다. 
그러므로 지연 시간이 발생하기 않기 때문에 스트림 처리에 있어 병목 현상을 제거할 수 있다. 
`key-value` 형식의 저장소로 주요 특징은 아래와 같다.  

- 고성능 : 외부 네트워크 없이 `SSD` 와 같은 고속 스토리지에 최적화돼 있어 빠른 읽기 쓰기 작업을 제공한다.  
- 내구성 : 재시작이나 시스템 장애가 발생해도 데이터가 유실되지 않는다. 
- 입축 및 효율성 : 데이터를 압축하여 디스크 공간을 효율적으로 사용하고, 다양한 압축 알고리즘을 지원한다. 
- 변경 로그 토픽 : `Kafka Streams` 의 상태 저장소는 변경 로그 토픽에 백업된다. `RocksDB` 에 저장되는 동시에 `Kafka Streams Changelog Topics` 에 기록되는 것이다. 이를 통해 시스템 장애시에도 상태 정보를 복구 할 수 있다. 
- 스냅샷과 체크포인트 : 정기적으로 스냅샷을 생성하고, 체크포인트를 통해 현재 상태의 일관된 뷰를 유지한다. 이를 통해 복구 과정에서 데이터 일관성을 보장하는데 도움이 된다.  



### Changelog Topics
`Changelog Topics` 는 상태 저장소에 발생하는 모든 변경 사항을 추적하는 중요한 메커니즘이다. 
이를 통해 `Kafka Streams State Store` 의 고장 내성(`fault tolerance`) 와 데이터 복구 능력(`recovery`)을 강화 할 수 있다. 
`State Store` 의 기본 저장소인 `RocksDB` 는 비동기적으로 디스크에 `flush` 해 상태 파일을 작성한다. 
`flush` 가 발생하기 전에 상태 저장소의 데이터가 메모리에만 존재하는 시간이 잠깐 존재하지만, 
이러한 시간 범위의 실패 시나리오에서도 해당 데이터는 `Changelog Topics` 를 통해 손실 되지 않는다.  

아래 도식화된 그림은 비트랜잭션 처리인 상황을 가정한 것이다. 
`Kafka Streams Application` 이 처음 구동되면 가장 먼저 `Checkpoint File` 을 찾게 된다. 
해당 파일에는 상태를 캡쳐한 마지막 오프셋이 있기 때문이다. 
파일이 존재한다면 `Changelog Topics Consumer` 를 통해 해당 오프셋을 사용해서 `Changelog Topics` 에서 상태를 읽어 복원한다. 
위 상황에서 `Changelog Topics` 의 상태는 앞서 언급한 메모리에만 존재하는 경우도 포함된다. 
파일이 없다면 `Changelog Topics` 의 모든 이벤트를 재생하는 방식으로 상태를 복원하게 된다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-1.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-2.png)


앞선 상황에 이어 서비스가 시작 후 상태 복원이 완료되면 스트림을 처리할 준비가 완료된 것이다. 
`Source Processor` 는 `Inbound Topic` 에서 배치로 메시지를 소비하고 이를 `Processor Topology` 를 통해 전달해서 스트림 처리를 수행한다. 
`State Store Processor` 는 변경 사항을 상태 저장소(e.g. `RocksDB`)에 쓰고, 
백업으로 `Changelog Topic` 에도 작성한다. 
그리고 배치단위 처리가 완료되면 `Consumer Offsets Topic` 에 처리한 마지막 오프셋을 작성하고, `Checkpoint File` 에도 오프셋을 작성한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-3.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-state-store-4.png)

`Changelog Topics` 는 압축이 활성화 돼있다. 
여기서 압축이란 토픽내에서 같은 키를 가진 메시지 중 오래된 것을 정리하고, 각 키에 대한 최신 값만 유지하는 것을 의미한다. 
이를 통해 저장 공간을 효율적으로 사용하고, 복구과정에서 필요한 데이터 양을 최소화한다. 
추가로 `Changelog Topics` 를 가능한 작게 유지하기 위한 방법으로는 `Tombstone` 이 있다. 
`Tombstone` 메시지는 키에 대한 값이 상제되었음을 나타내는 메커니즘으로, 
키는 있지만 값이 `null` 인 메시지로 특정 키의 이전 값들을 모두 무효화하고 해당 키를 삭제하려는 의도로 사용된다. 
다른 방법으로는 짧은 시간 범위를 갖는 `Window Store` 를 활용하는 방안도 있다.  


### Standby Replicas
`Stanby Replicas` 는 고가용성과 빠른 장애 복구를 위한 매커니즘으로, 
주 상태 저장소(`primary state store`)의 사본을 유지하여 주 상태 저장소 실패하는 경우 빠르게 복구 할 수 있도록 도와주는 역할을 한다. 
주요 기능을 나열하면 아래와 같다.  

- 데이터 복제 : 주 상태 저장소과 일치하는 사본을 유지한다. 
- 고가용성 : 주 상태 저장소에 장애가 발생한 경우, 스탠바이 복제본을 사용해 저장소를 빠르게 복구할 수 있다. 
- 장애 복구 시간 단축 : 이미 최신 상태 사본을 가지고 있으므로, 데이터를 다시 처리할 필요가 없어 복구 시간이 단축된다.  

`Standby Replicas` 는 `Kafka Streams` 애플리케이션을 구성할 때 `num.standby.replicas` 설정을 통해 스탠바이 복제본 수를 지정 할 수 있다. 
`num.standby.replicas=1` 이라면 각 상태 저장소에 대해 1개의 복제본을 유지하는 것을 의미한다. 
그리고 스탠바이 복제본은 주 상태 저장소외 동일한 `Changelog Topics` 를 구독해서 데이터를 동기화한다. 
만약 주 상태 저장소에 장애가 발생한다면 스탠바이 복제본은 즉시 주된 역할을 수행 할 수 있도록 준비한다. 
`Kafka Streams` 클라이언트는 자동으로 장애를 감지하고 스탠바이 복제본을 활성화해 처리를 계속 할 수 있다.  


### Consumer Group Rebalance
`Kafka Streams State Store` 와 `Consumer Group Rebalance` 는 중요한 관계를 가지고 있다. 
`Rebalance` 가 발생하면 `Topic Partition` 할당이 취소되고 동일 `Consumer Group` 의 `Consumer` 에게 다시 할당된다. 
이로 인해 `stop-the-world` 가 발생하고, `Realance` 가 완료 되기 전까지는 메시지 소비는 중단된다. 
그리고 `Rebalance` 가 완료되면 소비자는 `State Store` 와 `Changelog Topics` 를 사용해 로컬 상태 복원해야 한다. 
이러한 작업까지 완료된 이후에 메시지 스트림 처리가 가능하기 때문에 비용이 적지 않은 작업일 수 있다.  

이런 `Rebalance` 에 따른 추가적인 비용을 방지하기 위해 `Kafka Streams` 는 [StickyAssignor](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-partition-assignment-strategy/#stickyassignor)
를 사용하는 `StreamParitionAssignor` 를 사용한다. 
`Incremental Rebalancing` 방식으로 점진적으로 파티션 할당을 유지하고 수행하는 방식으로, 
실제 재할당이 필요한 파티션에 대해서만 메시지 소비가 중단되기 때문에 비교적 중단 영향을 적게 받을 수 있다.  

그리고 관련 비용을 줄이는 방법으로 또다른 옵션은 [Static Membership](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-consumer-group-membership/#static-membership)
을 사용하는 것이다. 
이는 정적으로 `group.instance.id` 를 지정해 두면 자신이 할당 받았던 `Topic Partition` 을 그대로 사용할 수 있으므로 복원시간에 큰 효과를 볼 수 있다.  

추가로 `Kafka Streams` 의 `Standby Replicas` 를 사용하면 `Rebalance` 시에 상태 저장소 복구 시간을 최소화 할 수 있다.  


### Persistent vs In-Memory State Store
`KafkaStreams` 에서 사용할 수 있는 `State Store` 는 `Persistent Stat Store` 와 `In-Memory State Store` 가 있다. 
이 두가지는 서로 `Trade-off` 되는 특성이 있기 때문에 구현하고자 하는 애플리케이션의 니즈에 맞춰 알맞은 선택이 필요하다.  

먼저 `Persistent State Store` 는 디스크에 저장하므로, 프로세스가 재시작되거나 장애가 발생해도 상태 데이터가 유지된다. 
이는 `Kafka Streams` 애플리케이션이 다운되는 상황에서도 상태 정보는 디스크에 있기 때문에 복구가 가능하다. (`Changelog Topics` 없이 복구가 가능하다는 의미)
그리고 메모리 제한에 구애받지 않기 때문에 큰 데이터 처리에도 활용될 수 있다.  
하지만 `Persistent State Store` 는 디스크 `I/O` 를 사용하기 때문에 메모리 방식과 비교했을 때 성능이 떨어질 수 있다.  

다음으로 `In-Memory State Store` 는 메모리 데이터를 관리하기 때문에 빠른 성능이 장점이다. 
하지만 애플리케이션이 종료되면 모든 상태 정보가 손실되어, 
항상 `Changelog Topics` 를 사용해 상태를 복구해야 하므로 더 많은 시간이 소요 될 수 있다. 
그리고 처리할 수 있는 데이터의 양이 시스템 메모리 크기에 제한되기 때문에, 큰 데이터 처리에 어려움이 있을 수 있다.  



---  
## Reference
[Kafka Streams: State Store](https://www.lydtechconsulting.com/blog-kafka-streams-state-store.html)  
[Kafka Streams Architecture State](https://docs.confluent.io/platform/current/streams/architecture.html#state)  



