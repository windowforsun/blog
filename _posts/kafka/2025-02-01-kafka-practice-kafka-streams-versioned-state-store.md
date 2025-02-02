--- 
layout: single
classes: wide
title: "[Kafka] "
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

## Kafka Streams Versioned State Store
`Kafka Streams` 는 기본적으로 오프셋 순서대로 데이터를 처리한다. 
만약 일련의 이유로 데이터의 순서가 뒤바뀐 경우, 오프셋 기준 순서는 타임스탬프의 기준과 순서가 달라지게 된다. 
그러므로 기존에는 이런 타임스탬프 순으로 데이터 스트림을 처리가 필요한 비지니스에서는 주의가 필요했다. 
`Apache Kafka 3.5` 부터는 `Kafka Streams` 에 버전 관리 상태 저장소(`Versioned State Store`)가 도입되어, 
타임스탬프 기준 순서가 중요할 때 크게 활용 할 수 있다.  

`Versioned State Store` 는 아래와 같은 주요 특징이 있다. 

- 타임스탬프 기반 조회 : 버전된 상태 저장소는 특정 시점에 데이터가 어떤 상태였는지를 조회할 수 있다. 이는 과거 데이터를 분석하거나 특정 시점의 상태로 롤백하는 데 유용할 수 있다. 
- 다중 버전 관리 : 동일한 키에 대해 시간에 따라 변하는 여러 버전을 저장할 수 있다. 
- 기록 보존 기간(`Retention Period`) : 버전된 상태 저장소는 각 데이터 버전을 특정 기간 동인 유지한다. 이 기간이 지나면 해당 버전은 자동으로 삭제된다. 이런 기록 보존 기간은 저장소를 생성할 때 설정 가능하다. 
- 성능 : 기존 상태 저장소와 달리 여러 버전을 저장하고 관리하기 때문에, 성능은 비버전 저장소보다 다소 낮을 수 있다. 하지만 최신 데이터는 별도의 저장소에 관리되므로 조회 시 성능저하가 크지 않다.  

