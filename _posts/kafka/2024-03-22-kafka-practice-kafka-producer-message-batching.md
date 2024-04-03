--- 
layout: single
classes: wide
title: "[Kafka] Kafka Producer Message Batching"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Producer 의 Throughput 을 튜닝 할 수 있는 옵션인 batch.size 와 linger.ms 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Producer
    - batch.size
    - linger.ms
    - Producer
    - Producer Batch
toc: true
use_math: true
---  

## Producer Message Batching
`Kafka Producer` 에서 메시지를 `Batch`(일괄) 처리를 적용하면 `Producer` 의 `Throughput` 를 늘릴 수 있다. 
이는 `Producer` 에서 발생하는 네트워크 요청 수르 줄이는 방법이다. 
하지만 이런 방식으로 `Throughput` 를 늘리면 요청에 대한 응답시간인 `Latency` 도 늘어 날 수 있으므로, 
해당 옵션을 변경하고자 할때 처리량과 응답시간 등이 적절한 균형과 최적화가 필요하다.  

`Kafka Prdocuer` 의 `Troughput` 를 향상 시키는 방법으로는 파티션 수, 메시지 압축, `Producer acks` 등이 있자민 
이번에 알아볼 것은 `Producer` 와 `Consumer` 간 전송되는 데이터 양을 최대화 하는 작업에 대해 알아본다.  
이러한 방식은 더 적은 수의 요청(더 큰 배치 사이즈)를 바탕으로 데이터를 전송하면 양 종단간 성능 향상을 기대할 수 있다.  

`Producer` 에서는 메시지르 `Batch` 처리하기 위해서 `batch.size` 까지 메시지가 채워지길 기다리거나, `linger.ms` 시간값이 만료될 때까지 기다려야 할 것이다. 
이러한 대기 시간은 처리량과 비례해서 증가할 수 있음을 기억해야 한다. 
또한 `Consumer` 는 자신이 구독하는 `Partition` 을 일괄 `poll()` 하게 되는데 이러한 `Consumer` 의 `Batch` 관련 처리 내용도 전체 서비스 `Throughput` 큰 영향을 준다. 
하지만 본 포스트 내용은 `Producer` 에 대한 것만 보고 `Consumer` 에 대한 내용은 이후 포스팅에서 다루도록 한다.  


### Configuration
메시지 `Batch` 처리와 관련된 `Producer` 의 옵션은 아래와 같이 `batch.size` 와 `linger.ms` 가 있다. 

option|desc|default
---|---|---
batch.size|일괄 처리할 최대 bytes 크기|16384bytes
linger.ms|batch.size 에 도달할 때까지 기다리는 최대 시간|0ms

`batch.size` 이상으로 쌓이게 되면 일괄 전송을 수행하게 된다. 
그리고 `batch.size` 를 키울 수록 `Producer` 에서 수행하는 요청수는 줄어들 것이다. 
하지만 무한정 기다릴 수 있기 때문에 `linger.ms` 라는 최대 대기시간을 두고, 타임아웃이 되는 경우에는 일괄 전송이 수행된다.  

![그림 1]({{site.baseurl}}/img/kafka/producer-message-bathcing-1.drawio.png)

만약 `batch.size` 가 너무 큰 값이고, `linger.ms` 는 `100ms` 라고 가정해보자. 
그러면 메시지 일괄 전송을 위해서 매번 `100ms` 대기시간이 발생하게 되고, 
이는 `batch.size` 를 `1` 로 두었을 떄 보다 큰 메시지 처리 지연이 발생하게 되는 것이다. 
말그대로 `linger.ms` 는 `Batching` 처리에 따른 지연 시간에 대한 상한을 두는 설정이다.  

### Tuning
애플리케이션의 `Throughput` 향상을 위한 `batch.size` 및 `linger.ms` 설정 최적화를 위해서는 다음 내용과 같은 고려가 필요하다. 
가장 먼저 필요한 것은 생산하는 메시지의 평균적인 크기와 메시지의 양을 알야아 한다. 
이러한 사전 파악은 `batch.size` 를 설정하는데 주요한 요소로 적용되기 때문이다. 
그리고 `Partition` 의 수와 메시지에 키가 포함돼 있는지도 설정값 결정에 있어 함께 고려해야 한다. 
만약 `Producer` 가 `Round-robin` 방식으로 다수의 `Partition` 에 메시지를 발송하는 경우라면, 
`batch.size` 에 도달하는 시간이 단일 `Partition` 에 메시지를 발송하는 경우보다 더 긴 시간이 소요 될 수 있기 때문이다.  

만약 메시지를 키를 사용해서 동일한 `Partition` 에 메시지를 발송하도록 돼 있는 상태라면, 
`linger.ms` 시간 내에 발송되는 메시지는 일괄로 처리될 수 있다. 
키값이 `Null` 인 경우에는 `Sticky Assignor` 파티션 할당 전략을 사용해서 단일 `Partition` 으로 발송하도록 할 수 있다. 
그리고 메시지 얍축을 사용하는 경우에는 메시지 당 크기가 줄어들기 때문에 `batch.size` 도달까지 좀 더 긴 시간이 필요할 수 있음을 기억해야 한다.  

요청 수를 줄이기 위해 큰 `batch.size` 를 사용 중이라면 해당 애플리케이션이 필요로하는 메모리 또한 함께 증가한다. 
여기서 `linger.ms` 를 `0` 으로 설정한다면 `Producer` 는 배치에 메시지가 누적 되는 즉시 혹은 `batch.size` 에 도달하는 즉시 메시지를 전송한다. 
이는 메시지 생산이 아주 빠른 경우 여전히 메시지가 배치에 누적되는 현상이 발생 할 수 있기 때문이다.  



---  
## Reference
[Kafka Producer Message Batching](https://www.lydtechconsulting.com/blog-kafka-producer-message-batching.html)    
[Kafka Producer Batch Properties](https://www.geeksforgeeks.org/apache-kafka-linger-ms-and-batch-size/)    
[batch.size](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html#batch-size)    
[linger.ms](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html#linger-ms)    