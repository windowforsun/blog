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
