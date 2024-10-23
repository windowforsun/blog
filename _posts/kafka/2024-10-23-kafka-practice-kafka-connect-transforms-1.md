--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 1"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT와 HoistField, ValueToKey, InsertField, Cast, Drop 예제를 살펴보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Retry
    - Non Blocking
    - Consumer
    - HoistField
    - ValueToKey
    - InsertField
    - Cast
    - Drop
toc: true
use_math: true
---

## Kafka Connect Transforms
`Kafka Connect` 에서 `Transforms` 는 `Single Message Transforms(SMT)` 기능 제공을 의미한다. 
`Kafka Connect` 는 데이터 스트리밍 플랫폼 `Apache Kafka` 의 주요 구성요소로, 
다양한 데이터 소스와 싱크로 데이터를 쉽게 이동할 수 있도록 도와준다. 
이러한 과정에서 요구될 수 있는 데이터 변환 혹은 필터링에 필요한 몇가지 기능을 제공한다.  

`SMT` 는 `Kafka Connect` 에서 단일 메시지 수준에서 변환 작업을 수행하는 기능을 의미한다. 
`SMT` 는 `Kafka` 에 데이터가 작성되기 전이나 데이터 소스에서 읽어 올 때, 
혹은 싱크에게 데이터를 쓰기 전에 변환/필터링이 필요한 경우 사용 할 수 있다.  

`Kafka Connect` 의 개념과 기본적인 사용법은 [여기](https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/?session_ref=https://www.google.com/&_gl=1*1vjg9z4*_ga*MjA0NzkyNTk2MC4xNzAxMzgyMDQ3*_ga_D2D3EGKSGD*MTcxNzkzMjc5My44Mi4xLjE3MTc5MzM0NjAuNTQuMC4w&_ga=2.7783026.1766080457.1717921898-2047925960.1701382047)
를 통해 확인 할 수 있다.  

또한 기본적으로 제공되지 않는 변환/필터링의 경우 [Custom Transforms](https://docs.confluent.io/platform/current/connect/transforms/custom.html)
를 통해 자체 구현 할 수도 있다.  

포스팅에서는 몇가지 `SMT` 의 역할과 동작에 대해서 데모 구성을 통해 사용법과 동작 결과에 대해 알아볼 것이다.  

