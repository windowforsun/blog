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

## Duplication Patterns
`Duplication Patterns` 는 중복 이벤트를 관리할 수 있는 `Kafka` 애플리케이션 패턴인 `중복 이벤트 방지 패턴` 을 의미한다. 
중복 이벤트는 `Kafka` 기반으로 분산 메시지를 처리할 때 불기파한 동작이다. 
하지만 `Duplication Patterns` 에서 소개하는 내용을 접목시키면 이러한 중복 이벤트 처리를 최소화할 수 있다.  
