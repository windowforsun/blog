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

## CDC
`CDC` 는 `Change Data Capture` 의 약자로 데이터베이스에서 발생하는 모든 변경 사항을 실시간으로 갭쳐하고 이를 다른 시스템으로 전파하는 기술이다. 
`CDC` 는 데이터 웨어하우스, 실시간 분석, 데이터 복제 등 다양한 사례에서 활용 할 수 있다. 
이렇게 `CDC` 는 데이터베이스의 데이터를 지속적으로 모니터링하고, 
`CRUD`와 같은 변경 사항을 캡쳐해 이를 다른 데이터베이스나 시스템으로 실시간 전송 및 동기화 와 통합을 간소화하고 효율적으로 구성할 수 있다. 
이런 `CDC` 에는 `polling-based(query-based) CDC` 와 `log-based CDC` 가 있는데 여기서 `log-based CDC` 가 `polling-based CDC` 와 
비교했을 때 어떠한 장점이 있는지 살펴보고자 한다.  
