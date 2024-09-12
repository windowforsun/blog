--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Spring Boot"
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


## Kafka Streams Transactions
`Kafka Transactions` 는 메시지의 흐름 `Consume-Process-Produce` 인 3단계 과정이 한 번만 발생하는 것을 보장한다. 
이것을 `Kafka` 에서는 `Exactly-Once Messaging` 이라고 표현한다. 
[Kafka Transactions]()
에서 알아본 일반적인 `Kafka Consumer, Producer` 를 사용하는 애플리케이션 뿐만 아니라, 
`Kafka Streams API` 를 사용하는 `Kafka Streams Application` 에서도 
`Kafka Transactions` 를 활성화해 각 메시지의 처음부터 끝까지의 흐름이 정확히 한 번 처리되고, 
상태가 내구성 있고 일관되도록 할 수 있다.  

[Kafka Streams Spring Demo]()
에서 알아본 것과 같이 `Kafka Streams Application` 은 
`Kafka Consumer/Producer` 를 바탕으로 구현되기 때문에 
`Kafka Transactions` 를 기존 방식화 동일하게 활성화 해준다면 동일한 효과를 얻을 수 있다. 
하지만 `Kafka Streams` 에서는 상태 저장과 같은 추가적인 동작이 있으므로 고민해야 할 부분이 더 있다. 
상태 저장 처리가 실패 했을 때 메시지 분실은 발생하지 않지만, 
메시지가 상태 저장소에 중복 쓰기를 발생시키지 않도록 하는 부분도 필요하다.   

