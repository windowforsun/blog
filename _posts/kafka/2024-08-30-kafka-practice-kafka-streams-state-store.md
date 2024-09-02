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

