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

## Producer Message Batching
`Kafka Producer` 에서 메시지를 `Batch`(일괄) 처리를 적용하면 `Producer` 의 `Throughput` 를 늘릴 수 있다. 
이는 `Producer` 에서 발생하는 네티워트 요청 수르 줄이는 방법이다. 
하지만 이런 방식으로 `Throughput` 를 늘리면 요청에 대한 응답시간인 `Latency` 도 늘어 날 수 있으므로, 
해당 옵션을 변경하고자 할때 처리량과 응답시간 등이 적절한 균형과 최적화가 필요하다.  

`Kafka Prdocuer` 의 `Troughput` 를 향상 시키는 방법으로는 파티션 수, 메시지 압축, `Producer acks` 등이 있자민 
이번에 알아볼 것은 `Producer` 와 `Consumer` 간 전송되는 데이터 양을 최대화 하는 작업에 대해 알아본다.  
이러한 방식은 더 적은 수의 요청(더 큰 배치 사이즈)를 바탕으로 데이터를 전송하면 양 종단간 성능 향상을 기대할 수 있다.  

`Producer` 에서는 메시지르 `Batch` 처리하기 위해서 `batch.size` 까지 메시지가 채워지길 기다리거나, `linger.ms` 시간값이 만료될 때까지 기다려야 할 것이다. 
이러한 대기 시간은 처리량과 비례해서 증가할 수 있음을 기억해야 한다. 
또한 `Consumer` 는 자신이 구독하는 `Partition` 을 일괄 `poll()` 하게 되는데 이러한 `Consumer` 의 `Batch` 관련 처리 내용도 전에 서비스의 `Throughput` 큰 영향을 준다. 
하지만 본 포스트 내용은 `Producer` 에 대한 것만 보고 `Consumer` 에 대한 내용은 이후 포스팅에서 다루도록 한다.  

