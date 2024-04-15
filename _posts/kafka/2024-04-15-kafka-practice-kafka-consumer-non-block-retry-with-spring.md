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

## Consumer Non-Blocking Retry
앞선 포스트에서 [Consumer Retry]()
에 대한 내용을 살펴보았다. 
이는 `Blocking retry` 방식으로 재시도를 수행해야하는 메시지가 있는 `Topic Partition` 은 
재시도가 수행되는 시간동안 `Blocking` 된다. (이후 메시지가 소비되지 못한다.)
하지만 `Non-blocking retry` 를 사용하게 되면 재시도를 수행하는 메시지가 있더라도, 
해당 `Topic Partition` 은 이후 메시지를 이어서 처리 할 수 있는 방식이 있다. 
`Spring Kafka` 를 사용해서 `Non-blocking retry` 를 최소한의 코드 변경으로 적용 할 수 있지만, 
`Non-blocking retry` 의 특징으로 메시지 순서는 보장되지 못함을 기억해야 한다.  

