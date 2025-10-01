--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Interactive Queries"
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
toc: true
use_math: true
---  

## Kafka Streams Interactive Queries
`Kafka Streams` 애플리케이션에서 관리되는 `State Store` 즉 상태는 
여러 애플리케이션 인스턴스에 분산돼 각 인스턴스의 로컬에서 관리된다. 
그 구조를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-1.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-2.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-3.drawio.png)


그리고 이런 `State Store` 를 별도로 목적에 따라 조회하는 것을 `Interactive Quries` 라고 한다. 
이러한 구조를 갖는 `Kafka Streams` 에서 `State Store` 의 쿼리는 아래와 같이 `Local State` 와 `Remove State` 로 구분될 수 있다.  

- `Local State` : 애플리케이션 인스턴스 로컬에서 관리되는 전체 상태의 일부를 의미한다. 이런 로컬 상태는 직접 쿼리해 필요한 경우 바로 사용할 수 있다. 다만 여기서 별도로 로컬 상태에 접근하는 것은 `read-only` 모드로 일기 전용임을 기억해야 하고, 로컬 상태의 변경은 `Kafka Streams API` 에 의해서만 가능하다. 
- `Remote State` : `Consumer Group` 에서 사용하는 각 `State Store` 의 전체 상태를 의미한다. 이는 여러 `Local State` 를 연결해야 하기 때문에 `Local State` 쿼리, `Network` 기반 모든 애플리케이션 인스턴스 로컬 저장소 탐색, 모든 애플리케이션과 네트워크 통신의 구성이 필요하다. 

정리하면 `Local State` 는 `Kafka Streams` 에서 기본적으로 제공하는 `API` 를 통해 `Local State Store` 에 대한 `Interactive Queries` 가 가능하다. 
하지만 전체 상태에 대한 정보를 조회한다거나 현재 `Local State` 에 존재하지 않고 다른 인스턴스에 존재하는 값이 필요한 경우는 관련 있는 모든 인스턴스에서 
자신의 `Local State` 를 외부에서 접근할 수 있도록 노출하는 별도의 작업이 필요하다. 
그리고 노출된 `Local State` 를 원격으로 접속해 쿼리할 수 있는 `RPC` 구현도 있어야 한다. 
아래는 `Kafka Streams` 를 사용할 때 `Remote State` 사용 절차에 있어서 `Single Application` 와 `Entire Application` 의 `Kafka Streams` 의 기능 지원여부를 정리한 것이다.

구분 | Single Application |Entire Application 
---|--------------------|---
현재 인스턴스에서 현재 로컬 상태 조회| 지원|지원
현재 인스턴스를 다른 인스턴스에 발견 하도록 만들기|지원|지원
모든 실행 중인 인스턴스의 상태 저장소 발견|지원|지원
네트워크를 통한 전체 인스턴스 간 통신|지원| 지원하지 않음(별도 구성 필요)

위 정리 내용에 대해 좀 더 상세히 설명하면 아래와 같다. 

구분 | Single Application                          |Entire Application 
---|---------------------------------------------|---
현재 인스턴스에서 현재 로컬 상태 조회| 자신의 로컬 상태 저장소에는 직접 쿼리할 수 있다.                | 여러 인스턴스의 로컬 상태를 개별적으로 쿼리할 수 있다. 이때 필요한 데이트가 어느 인스턴스에 위치하는 지는 별도로 판별 후 해당 인스턴스에 쿼리해야 한다. 
현재 인스턴스를 다른 인스턴스에 발견 하도록 만들기| 각 인스턴스는 자신의 `호스트:포트` 등 메타데이터를 외부로 제공할 수 있다. |모든 인스턴스의 메타데이터를 관리하고 공유할 수 있다. 
모든 실행 중인 인스턴스의 상태 저장소 검색| 각 인스턴스는 다른 모든 인스턴스와 상태 저장소를 검색할 수 있다.       |네트워크를 통해 전체 인스턴스의 상태 저장소를 검색할 수 있고, 이를 통해 필요한 상태 저장소가 있는 인스턴스를 특정할 수 있다. 
네트워크를 통한 전체 인스턴스 간 통신| 각 인스턴스는 다른 인스턴스와 네트워크를 통해 통신할 수 있다. | 전체 애플리케이션 레벨에서 `RPC` 통신은 `Kafka Streams` 에서 기본 제공하지 않고, 직접 구현해야 한다. 

> 여기서 전체 애플리케이션이라 함은 `Kafka` 의 `Consumer Group` 단위로 봐도 무방하다.


이후 설명에 사용하는 모든 예제의 상새 내용은 [여기](https://github.com/windowforsun/kafka-streams-interactive-queries-exam)
에서 확인 할 수 있다.  

### Query local state stores of an app instance
`Kafka Streams` 에서 현재 인스턴스의 `Local State Store` 라는 것은 전체 `State Store` 의 일부이다. 
현재 로컬 상태의 조회가 필요한 경우 `KafkaStreams.store()` 를 사용해 로컬 상태 저장소를 이름과 저장소 유형에 따라 찾을 수 있다. 

> `Kafka Streams 3.5` 버전 기준으로 `VersionesStateStore` 는 지원되지 않는다. 

조회에 필요한 상태 저장소 이름은 `Processor API` 혹은 `Streams DSL` 을 사용할 때 멍시적으로 설정하거나, 
설정하지 않은 경우 암시적으로 생성되기 때문에 이를 인지하고 사용해야 한다. 
그리고 상태 저장소의 유형의 경우 `QueryableStoreType` 을 통해 결정할 수 있다.  


> `Kafka Streams` 는 스트림 파티션당 하나의 상태 저장소를 구성한다. 
> 즉 해당 애플리케이션 인스턴스가 `N` 개의 파티션을 할당 받았다면 로컬 상태 저장소도 파티션 수에 비례한다는 의미이다. 
> `Interactive Queries` 즉 `KafkaStreams.store()` 를 통해 얻은 상태 저장소 객체의 경우 
> 이름과 저장소 유형에 해당하는 각 파티션 별 상태 저장소가 통합된 상태로 제공하기 때문에 이러한 부분을 크게 고려할 필요는 없다. 

