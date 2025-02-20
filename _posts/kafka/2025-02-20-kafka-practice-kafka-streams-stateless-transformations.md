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
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Stateless Transformations
`Kafka Streams` 에서 `Stateless Transformations` 는 데이터 처리에 이전 상태나 컨텍스트를 참조하지 않고, 
각 이벤트나 메시지를 독립적으로 변환하는 방식으로 상태 저장소(`State Store`)가 필요하지 않다. 
이러한 데이터 변환은 특정 레코드나 처리 결과가 다른 레코드의 처리 결과에 영향을 주지 않고, 
즉작적이고 독립적인 처리가 이뤄지는게 특징이다.  

`Stateless` 라는 용어는 말 그대로 상태를 유지하지 않는다는 의미로, 
데이터의 처리 과정에서 앞서 처리한 레코드나 상태를 기억하지 않고 각 레코드를 독릭접으로 변환한다. 
이는 비교적 빠르고 단순한 데이터 변환에 적합하고, 
데이터의 순서나 이전 레코드의 상태를 고려하지 않는다. 
주요 특징을 정리하면 아래와 같다.  

- 독립적인 레코드 처리 : 각 레코드는 독립적으로 처리되고, 앞서 처리된 레코드나 다른 레코드의 결과와 관계없이 별도의 처리 로직이 적용된다. 
- 상태 정보 없음 : 상태를 기억하지 않기 때문에 상태 유지 및 동기화 관련 복잡성이 제거된다. 이에 따라 처리 로직이 단순해지며, 성능도 형상된다. 
- 빠르고 간결한 처리 : 간단한 필터링, 매핑, 변환 등의 작업을 빠르게 처리할 수 있어 대용량 스트리밍 데이터 처리에 유리하다.  
- 분산 처리 : 상태를 유지하지 않기 때문에 더 쉽게 분산 처리가 가능하여 시스템 확장성에 유리하다.  

`Stateless Transformation` 이 갖는 한계는 아래와 같다.  

- 복잡한 처리 : 상태를 고려해야 하는 복잡한 처리 로직은 구현이 어렵다. 이런 경우에는 `Stateful Transfomration` 을 활용해야 한다. 
- 순서 의존적인 처리 : 각 레코드가 독립적으로 처리되기 때문에 레코드의 순서를 고려해야 하는 처리 로직에는 적합하지 않다. 
