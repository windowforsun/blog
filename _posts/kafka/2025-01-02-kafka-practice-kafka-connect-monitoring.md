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

## Kafka Connect Monitoring
`Kafka Connect` 를 모니터링 하는 것은 `Kafka` 기반 데이터 통합 파이프라인의 
안전성, 성능, 신뢰성을 보장하는데 필요한 작업이다. 
모니터링을 통해 `Kafka Connector` 들의 성능과 상태를 추적하고 관리할 수 있다. 
이를 위해 `Kafka Connect` 에서 제공하는 `JMX Metrics` 를 수집해 이를 식각화 하는 환경을 구축하는 방법에 대해 알아보고자 한다.  

`Kafka Connect` 의 모니터링을 구축하기 위해 `JMX` 로 메트릭을 `Export` 하고 이를 `Prometheus` 를 통해 수집한 후 `Grafana` 를 통해 
모니터링 대시보드를 구축한다. 
이를 도식화해 그리면 아래와 같다.  

구축의 상세 내용이 있는 예제는 [여기]()
에서 확인 할 수 있다.  
