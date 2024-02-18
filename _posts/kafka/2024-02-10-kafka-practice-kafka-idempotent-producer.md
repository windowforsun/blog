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

## Idempotent Producer
`Kafka` 기반 애플리케이션을 개발하고 운용할때 
발생 할 수 있는 오류 시나리오 중 하나가 바로 중복 이벤트 발생이다. 
만약 중복 이벤트가 발행된다면 이후 프로세스의 중복 처리 뿐만아니라, 
메시지의 순서도 꼬일 수 있다.
[Idempotent Consumer]({{site.baseurl}}{% link _posts/kafka/2024-01-13-kafka-practice-kafka-duplication-patterns.md %}) 에서는 중복 이벤트를 소비하지 않는 방법에 대해 알아 봤다면,
본 포스팅에서는 어떤 상황에서 중복 이벤트가 발생 횔 수 있고, 
중복 이벤트를 방지 할 수 있는 `Idempotent Producer` 에 대해 알아본다.  
