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

## Kafka Keys with Message Ordering
`Kafka Messaging Broker` 를 사용해 메시지를 처리할 떄, 
메시지는 키와 함께 사용해서 동일한 키를 가진 메시지들에 한해 메시지 순서를 보장 할 수 있다. 
`Kafka` 의 메시지는 `key-value` 쌍으로 구성되는데, 키는 메시지가 저장되는 파티션을 결정하는데 사용될 수 있다. 
그러므로 `Producer` 가 키와 함께 발송된 메시지는 동일한 `Topic Partition` 에 기록되고, 
`Consumer` 는 각 `Partition` 의 메시지는 순서대로 읽어들이기 때문에 메시지 순서가 유지 될 수 있다.  

본 포스팅과 관련된 예제는 []()
에서 전체 코드를 확인해 볼 수 있다.
