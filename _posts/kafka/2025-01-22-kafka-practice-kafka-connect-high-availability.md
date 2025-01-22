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

## Kafka Connect HA
`Kafka Connect` 는 데이터 스트림의 수집과 변환을 자동화하는 분산 시스템이다.
이를 활용하면 다양한 데이터 소스를 `Kafka` 로 수집하거나 `Kafka` 에서 다른 데이터 소스로 싱크를 구성 할 수 있다. 
대규모 실시간 데이터 처리 환경에서는 고가용성(`High Availability`)을 보장하여 시스템 중단 없이 안정적인 운영을 유지하는 것이 중요하다.
이번에는 `Kafka Connect` 를 구성하고 사용 할 때 `HA(High Availability)` 를 어떠한 방법으로 보장 받을 수 있는지 알아본다.  

- 무중단 : 데이터 수집 및 전달의 중단 없이 지속적으로 운영될 수 있어야 한다. 
- 장애 복구 : `Worker` 나 `Connector` 가 장애를 겪는 상황에서 빠르게 복구 할 수 있어야 한다. 
- 성능 및 확장성 : 여러 `Worker` 가 작업을 분산 처리함으로써 성능을 향상시키고, 시스템 확장성이 용이하다. 
