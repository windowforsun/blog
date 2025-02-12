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

## Kafka Streams State Store Type
`Kafka Streams` 에서 `State Store` 은 스트리밍 애플리케이션이 상태를 유지할 수 있도록 하는 중요한 구성 요소이다. 
`State Store` 은 데이터 처리 중에 필요한 상태(e.g. 집계, 윈도우, 조인, ..)를 저장하고, 
이를 통해 스트림 처리를 바탕으로 필요한 비지니스를 구현할 수 있다. 
`Kafka Streams` 에서는 크게 `In-Memory State Store` 와 `Persistent State Store` 라는 두 가지 유형의 상태 저장소가 있다.  

이후 설명하는 `State Store` 의 설명은 각 유형별 특징과 차이점에 초점을 맞춘 내용이다. 
`State Store` 에 대한 전반적인 내용은 [여기]()
에서 확인 가능하다.  


### In-Memory State Store
`In-Memory State Store` 은 애플리케이션의 메모리에 상태 데이터를 저장한다. 
메모리에 저장된 데이터는 디스크에 기록되지 않기 때문에 애플리케이션이 재시작되거나 장애기 발생하면 해당 데이터는 사라져 `State Persistent` 를 제공하지 않는다. 
이렇게 데이터가 메모리에 저장되는 만큼 읽고 쓰기가 매우 빠른 접근 속도를 제공한다. 
이는 지연을 최소화하고 빠른 실시간 처리가 필요한 애플리케이션에서 유리하다. 
그리고 모메리 용량에 따라 최대로 저장할 수 있는 데이터의 양이 제한된다. 
큰 데이터를 다루거나, 상태 크기가 커지는 경우에는 적합하지 않을 수 있다.

### Persistent State Store
`Persistent State Store` 는 상태 데이터를 디스크에 저장한다. 
`RocksDB` 와 같은 `key-value` 저장소를 사용해 데이터를 관리하며, 
애플리케이션 재시작이 되더라도 상태가 유지될 수 있는 `State Persistent` 를 제공한다. 
디스크에 상태를 저장하기 때문에 애플리케이션 종료나 장애상황에서도 데이터가 유지될 수 있기 때문에 복구 관점에서 매우 유리하다. 
하지만 읽고 쓰기의 동작이 디스크 I/O에 크게 의존하기 때문에 `In-Memory State Store` 보다는 성능적으로 불리할 수 있다. 
그렇지만 `RocksDB` 는 고성능 데이터저장소이므로 이런 성능 저하를 최소화 할 수 있다.   


> 여기서 주의해야할 점은 `In-Memory State Store` 와 `Persistent State Store` 의 가장 큰 차이점은 `State Persistent` 의 제공 여부이다. 
즉 `Fault-Tolerance` 보장 관점에서는 두 저장소 유형 모두 이를 제공한다는 의미이다. 
`Kafka Streams State Store` 는 `change-log topic` 을 바탕으로 `State Store` 의 `Fault-Tolerance` 를 제공한다. 
`State Store` 의 변경상태를 `change-log topic` 에 기록하고 이러한 싱태변경 기록을 바탕으로 애플리케이션이 재시작되거나 
장애가 발생했을 때 애플리케이션에서 상태를 복구할 수 있도록 한다. 
이는 `In-Memory State Store`, `Persistent State Store` 모두 재시작, 장애 상황에서도 현 상태를 복구할 수 있는 매커니즘은 존재한다는 의미이다. 
하지만 `Persistent State Store` 는 해당하는 상태파일이 저장소에 있다면 `change-log topic` 을 바탕으로 복구를 진행하지 않고, 
`In-Memory State Stre` 는 매번 `change-log topic` 을 바탕으로 상태 복구가 진행될 수 있기 때문에 저장소 크기에 따라 복구 성능에는 차이가 있을 수 있다. 
이에 대한 자세한 내용은 이후에 다시 다루도록 한다.  


### State Store Type
`Kafka Streams` 을 사용해서 `Topology` 를 구성할 때 사용할 수 있는 `State Store` 에는 어떤 유형이 있는지 알아본다. 
이와 관련된 전체 소스 코드는 [여기]()
에서 확인 할 수 있다.  

사용 가능한 `State Store` 종류별 특징을 확인하기 위해 예제 스트림은 투표결과를 카운트하는 비지니스를 구현한다. 
이를 통해 동일한 투표 스트림 데이터가 들어올 때 각 `State Store` 가 어떤 결과를 도출하는 지 확인하는 과정으로 각 상태 저장소의 특징과 차이를 알아본다.  
아래는 예제에 대한 메시지와 `State Store` 기반 처리 과정을 도식화한 것이다. 

.. 그림 1..

위 메시지 스트림에서 주의해야할 부분은 `voter5` 의 투표이다. 
`a` 로 투표한 메시지가 먼저 도착한 후, `b` 로 변경된 투표가 도착한 것을 볼 수 있다. 
하지만 먼저 도착한 `a` 의 시간값이 더 최신이고, 
`b` 로 변경된 투표는 과거이므로 `voter5` 는 실제로는 `b` 로 투표를 한 후 `a` 로 변경했지만 시스템의 문제로 메시지 순서가 변경된 것이다. 
각 `State Store` 마다 이런 상황에서 어떠한 결과를 보이는지도 함께 살펴보고자 한다. 
추가로 이러한 순서가 바뀐 경우 순서를 보장하도록 처리할 수 있는 방안이 `VersionedStateStore` 인데 해당 포스팅에서는 간단한 개념만 다루고, 
자세한 내용은 [여기]() 에서 확인 할 수 있다.  
