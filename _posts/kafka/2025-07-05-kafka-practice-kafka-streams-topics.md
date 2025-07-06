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

## Kafka Streams Topics
`Kafka Streams` 애플리케이션은 `Kafka Broker` 의 다양한 `Topic` 들과 지속적으로 데이터를 주고 받으면 데이터 스트림을 구현한다. 
이는 사용자가 필요에 의해 정의한 명시적인 `Topic` 도 포함되지만 `Kafka Streams` 에서 필요에 의해서 생성되는 `Topic` 도 포함된다. 
이번 포스팅에서는 `Kafka Streams` 애플리케이션을 구현하고 사용할 때 어떠한 유형의 `Topic` 들이 사용되고 필요하지에 대해 알아본다. 
크게 아래 2가지의 `Topic` 으로 구분될 수 있다.  

- `User Topics`
- `Internal Topics`
