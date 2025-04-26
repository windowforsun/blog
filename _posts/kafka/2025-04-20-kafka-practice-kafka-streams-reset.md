--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Reset"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams Reset Tool 을 사용해 스트림 애플리케이션을 리셋하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Kafka Streams Reset
    - Kafka Streams Reset Tool
    - Reset Tool
toc: true
use_math: true
---  

## Kafka Streams Reset
`Kafka Streams Reset` 이란 `Kafka` 의 `Streams API` 애플리케이션이 입력 데이터를 처음부터 다시 처리하도록 설정하는 것을 의미한다. 
실제 운영/개발 환경에서 스트림 처리를 처음 부터 다시 처리하는 것은 흔한 상황으로, 
개발 테스트 중 혹은 프로덕션의 버그 발생, 데모 구현 등 목적은 다양할 수 있다.  

`Kafka Streams` 는 이런 작업을 수동으로 개발자가 직접 처리할 수도 있지만, 
간편한 리셋 도구를 제공한다. 
그러므로 개발자는 `Apache Kafka 0.10.0.1` 버전 이후라면 리셋도구를 사용해 보다 손쉽고 오류 발생없이 스트림을 처음부터 다시 처리할 수 있다.  

`Kafka Streams Reset Tool` 은 `Kafka Streams Application` 을 구성하는 각 유형의 토픽에 대해서 아래와 같은 동작을 수행한다. 

- `Input Topics` : 오프셋을 툴에서 지정한 위치로 리셋한다. 기본적으론 토픽의 시작위치로 리셋한다. 
- `Intermediate Topics` : 토픽 끝으로 건너뛴다. 애플리케이션의 커밋된 소비자 오프셋을 모든 파티션의 로그 크기로 설정한다. (`applicaiton.id` 이름으로 구성된 토픽)
- `Internal Topics` : 내부 토픽은 삭제한다.  

위와 다르게 수행하지 않는 동작은 아래와 같다.  

- `Output Topics` : 애플리케이션이 출력 토픽으로 결과를 쓰는 경우 해당 토픽은 리셋하지 않는다. `upstream` 애플리케이션이 리셋되어 `downstream` 으로 중복데이터가 전달되는 것에 대한 처리는 사용자의 책임이다. 
- `Local State Store` : 애플리케이션 인스턴스의 내부 상태 저장소는 리셋하지 않는다. 
- `Schema Registry for Internal Topics` : 내부 토픽이 사용하는 스키마 레지스트리의 스키마는 삭제하지 않는다. 이는 수동으로 삭제해야 한다. `Reset Tool` 에서 `dry-run` 옵션으로 초기화기 필요한 `Internal Topics` 를 확인 할 수 있다.  


`Kafka Streams` 의 `Reset` 은 아래와 같은 경우에 필요할 수 있다.  

- 개발 테스트 목적 : 애플리케이션 로직을 변경하고 처음부터 데이터를 다시 처리해야 할 때 사용한다. 이는 새로운 기능을 테스트하거나 버그를 수정한 후 결과를 확인하는데 유용하다. 
- 토폴로지 변경 : 스트림 처리 토폴로지를 크게 변경한 경우, 특히 상태 저장 연산이나 조인 연산이 변경되었을 때 리셋이 필요할 수 있다. 
- 데이터 재처리 : 입력 데이터의 처리 방식을 번경하거나, 과거 데이터를 새로운 로직으로 다시 처리해야 할 때 사용한다. 
- 상태 초기화 : 애플리케이션의 상태를 완전히 초기화하고 처음부터 다시 시작해야 할 때 사용한다. 
- 버그 수정 : 버그 수정 후, 잘못 처리된 데이터를 바로잡기 위해 전체 데이터를 재처리해야 할 때 사용한다. 
- 스키마 변경 : 입력/출력 데이터의 스키마가 크게 변경되어 기존 처리 결과와 호환되지 않을 때 사용한다. 
- 설정 변경 : 주요 설정(상태 저장소 구성)을 적용한 후 애플리케이션을 리셋해야 할 때 사용한다. 

### Before Reset
`Reset Tool` 을 사용하기 전 리셋 대상인 `application.id` 에 해당하는 모든 인스턴스는 중지돼야 한다. 
실행된 상태에서는 `Reset Tool` 이 에러를 출력한다. 
이는 `kafka-consumer-groups.sh` 을 사용해 리셋 대상인 `application.id` 이 조회되는 지로 확인 할 수 있다.  

`Intermdiate Topics` 의 경우 이를 구독하는 `downstream` 이 있는 경우 혹은 개발환경과 같이 테스트인 경우를 제외하고는 
삭제 및 재생성을 해주는 것이 좋다.  


### Topology Changes
`Kafka Streams Reset` 관점에서 `Compatible` 과 `Incompatible` 의 의미는 아래와 같다. 
여기서 호환 된다는 것은 스트림 데이터의 관점에서 변경된 토폴로지로 리셋 후 재시작을 수행 했을 때, 
데이터의 호환성과 일관성 관점에서 문제가 없다는 것을 의미한다. 


#### Compatible Topology Changes
`Kafka Streams Reset Tool` 의 호환성은 주로 토폴로지 변경 후 데이터의 처리의 일관성과 정확성을 유지할 수 있는지에 관한 것이다. 

1. 필터 조건 변경
2. 새로운 필터 추가(레코드별 작업)
3. 데이터 타입이 호환되는 새로운 `map()` 연산 추가 (`repartition` 이 발생하지 않는 상황)
4. 값 타입을 변경하지 않는 `mapValues()` 연산 추가
