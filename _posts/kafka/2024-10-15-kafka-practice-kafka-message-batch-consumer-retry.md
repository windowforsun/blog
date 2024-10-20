--- 
layout: single
classes: wide
title: "[Kafka] Kafka Message Batch Consumer Retry"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 에서 Message 를 Batch 방식으로 N개씩 한번에 소비하는 시나리오의 특징과 실패/재시도와 같은 상황에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Consumer
    - Consumer
    - Kafka Batch
toc: true
use_math: true
---  

## Kafka Batch Consume Retry
`Kafka Consumer` 는 `Kafka Broker` 로 부터 `Batch` 성으로 다수의 메시지를 한번에 받을 수 있다. 
이런 `Batch` 성 작업을 수행할 때 중요하게 고려돼야 하는 것이 바로 실패에 대한 시나리오이다. 
`Kafka Library` 가 이런 상황에서 어떠한 식으로 동작하는지에 대해 알아보고자 한다.  

`Kafka Consumer` 는 하나 이상의 `Topic` 혹은 하나 이상의 `Partition` 을 할당 받게 된다. 
이는 `Kafka Consumer` 는 주로 단일 `Topic` 을 구독해 처리하지만, 다수의 `Topic` 을 구독해 처리할 수 있다는 의미이고 
그 수는 제한되지 않는다. 
각 시나리오 설명을 위해 `Topic` 에서 `Batch` 로 메시지를 소비할 때 3개씩 받아 처리한다고 가정한다. 
그리고 명시적인 언급이 없다면 이후 시나리오에서 [enable.auto.commit](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#enable-auto-commit)
은 `true` 로 활성화 돼있다고 가정한다. 
`enable.auto.commit` 이 `false` 로 비성화 돼있다면 `Consumer` 의 `offset` 관리는 코드에서 직접 수행해야 한다.  

[max.poll.records](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-records)
와
[max.poll.interval.ms](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-interval-ms)
에서 보면 알 수 있듯이 `Kafka Consumer` 는 기본적으로 `batch` 방식, 
즉 `poll` 을 수행 했을 때 단일 메시지를 소비하는 것이 아니라 `N` 개의 메시지 소비한다는 점을 기억해야 한다.  

그리고 관련해서 혼동이 있을 수 있는데, 포스팅에서 다루는 내용은 `Kafka Consumer` 의 `max.poll.records` 와 연관된 `batch` 방식과 관련있다. 
`Spring Kafka` 를 사용해서 `Kafka Consumer` 관련 빈을 구성 할 때, `ConcurrentKafkaListenerContainerFactory` 에서 설정하는 `setBatchListener(true)` 와는 시나리오 별 차이가 있다는 점도 기억해야 한다.  


### Successfully consumed
첫 번째 시나리오는 단일 `Topic`, 단일 `Partition` 가 할당된 `Kafka Consumer` 가 
`Batch` 로 전달된 메시지들을 모두 정상적으로 소비하고 처리하는 경우이다.  

`Kafka Client` 는 `Inbound Topic` 으로 부터 3개의 메시지를 `poll` 하고, 
각 메시지를 연결된 `@KafkaListener` 에게 전달한다. 
그리고 리스너는 메시지를 하나씩 순서대로 처리하게 된다. 
`poll` 을 수행한 메시지들의 처리가 완료되면 `Kafka Client` 는 `Consumer Offset Topic` 에 
`poll` 메시지 중 마지막 메시지의(`m3`) 오프셋을 커밋하고, 이 시점부터 처리한 메시지들이 소비동작이 완료된다. 
그리고 그 다음 `poll` 둥작은 `m4` 메시지 부터 시작해 가져오게 된다. 


![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-1.drawio.png)


아래 도식화된 흐름은 `Kafka Client` 가 배치로 `Inbound Topic` 으로 부터 
메시지를 소비하고 이를 `Kafka Consumer` 로 전달 및 처리 후 오피셋을 커밋까지 과정을 보여준다. 
해당 예시에서는 단일 `Inbound Topic` 을 사용하므로 `Consumer Offset Topic` 의 커밋 또한 하나의 레코드로 수행된다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-2.png)



### Consumer Failed
이번 시나리오 또한 `Kafka Client` 가 `Inbound Topic` 으로 메시지를 소비하고, 
각 메시지를 `Kafka Consumer` 에게 전달하는 흐름까지는 동일하다. 
다만 첫 번째 `poll` 을 통해 소비한 3개의 메시지 중 2번째 메시지인 `m2` 에서 `Kafka Consumer` 가 중단된다. 
`Kafka Consumer` 는 아직 3개의 메시지 전체를 처리하기 전이기 때문에 `Consumer Offset Topic` 에 처리를 완료한 메시지 오프셋을 커밋하지 못하게 된다.  

`Kafka Consumer` 가 중단된 상황이 발생하면 `Kafka Broker` 에게 `Heartbeat` 를 보내지 않게 되고 이를 통해 `Rebalance` 가 이뤄진다. 
그리고 활성화된 `Kafka Consumer` 에게 소비 중이던 `Partition` 이 할당되게 된다. 
이 경우 새롭게 할당 받은 `Kafka Consumer` 가 처음 `poll` 하면, 
다시 동일한 `m1`, `m2`, `m3` 메시지가 소비되게 된다. 
즉 이전 `Kafka Consumer` 에서 처리가 완료된 메시지 `m1` 은 중복 처리된다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-3.drawio.png)


아래 흐름을 보면 `enable.auto.commit` 이 활성화 된 `Kafka Consumer` 가 메시지 처리 도중 중단될 경우
`poll` 로 소비한 전체 메시지를 모두 처리하기 전까지 `offset` 을 커밋하지 않고, 재시작 되면 다시 `poll` 을 수행하면 
이전과 동일한 메시지 배치를 수신하는 증상을 보여주고 있다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-4.png)


이러한 특성으로 중복 메시지가 전달되고 처리되는 가능성에 대한 고려가 필요하다. 
이는 `Kafka` 의 `at-least-once delivery` 의 특성을 보여주고 있는데, 
더 자세한 내용은 [여기](2024-03-12-kafka-practice-kafka-consum-produce-at-least-once.md)
에서 확인 가능하다.  


### Exception in Consumer
마지막 시나리오는 `Kafka Consumer` 처리 도중 예외가 발생하는 상황이다. 
이는 발생한 예외를 `Kafka Client` 로 까지 전달하는지, 전달하지 않는지에 따라 이후 `batch` 처리의 방식이 달라 질 수 있다. 
먼저 `Kafka Consumer` 에서 메시지 처리 도중 예외는 발생했지만, 해당 예외를 별도로 처리하고 `Kafka Client` 에게 던지지 않는다면 
해당 메시지는 정상적으로 처리된 것으로 간주된다. 
그리고 그 다음 배치가 기존처럼 수행된다. 
`Kafka Consumer` 에서 발생한 예외를 `Kafka Client` 로 던지지 않는 경우 일반적인 처리 방법으로는 `dead-letter-topic` 으로 전송하는 방안이 있다.  

하지만 `Kafka Consumer` 에서 발생한 예외를 그대로 `Kafka Client` 에게 던지게 된다면, 해당 메시지는 정상적으로 소비되지 않는 것으로 간주된다. 
이는 다음 `poll` 때 메시지를 다시 수신해 재처리 할 수 있도록 하기 위함으로 `API`, `DB` 요청과 같이 일시적인 에러 상황에서 재처리를 통한 복구를 수행할 수 있다.  

현재 예제에서 다루고 있는 상황에서 보면, 처음 `poll` 을 수행하면 `m1`, `m2`, `m3` 3개 메시지가 소비된다. 
그리고 `m2` 에서 예외가 발생하고 이를 `Kafka Client` 에게 던지게 된다. 
그러면 `Kafka Client` 는 `enable.auto.commit` 이 활성화 된 경우 처리가 완료된 `m1` 의 오프셋을 커밋한다. 
그리고 `Kafka Client` 의 다음 `poll` 에서는 `m2`, `m3`, `m4` 3개 메시지가 소비된다. 
이번에는 전체 메시지가 모두 정상적으로 처리 되므로 `Kafka Client` 는 마지막 메시지인 `m4` 의 오프셋을 커밋한다.   

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-5.drawio.png)


아래 도식화된 흐름은 `Kafka Client` 와 `Kafka Consumer` 간에 에외가 발생 했을 때의 오프셋 커밋과 재처리 과정을 보여주고 있다. 
만약 예외가 발생하는 `m2` 메시지에서 예외 발생전 `API` 호출이 있다고 가정한다면, 이는 중복 처리가 될 수 있음을 기억해야 한다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-6.png)


### Multiple Topic
`Kafka Consumer` 는 여러 `Partition` 또는 여러 `Topic` 을 한번에 구독해 메시지를 소비 할 수 있다. 
여기서 `Kafka Client` 가 `poll` 을 수행해 메시지를 소비할 때 각 `Topic` 에 대해서 받는 것이 아닌 전체 `Topic` 을 대상으로 
`batch` 방식으로 메시지를 수신하게 된다. 
즉 아래 그림으로 바탕으로 `poll` 을 수행 했을 때 3개의 메시지 소비한다면 `m1`, `m2`, `mm1` 메시지를 소비한다고 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-batch-consumer-retry-7.drawio.png)

이러한 여러 `Topic` 을 구독하는 `Kafka Consumer` 또한 앞서 알아본 3가지 경우의 증상은 모두 동일하다. 


### Kafka Consumer Retry
`Kafka Consumer` 를 사용해서 메시지를 처리할 때 예외 처리는, 예외 처리가 불가한 경우 `dead-letter-topic` 으로 전달하거나 
`Kafka Client` 에게 그대로 예외를 던지는 방낭이 있을 수 있다. 
이러한 재시도 구성에 있어서 다양한 고려 사항과 스펙이 있는데, `stateless retry` 와 `stateful retry` 등이 있다. 
이와 관련 자세한 내용은 [여기](2024-04-02-kafka-practice-kafka-consumer-retry.md)
에서 확인 할 수 있다.  
