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


