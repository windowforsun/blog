--- 
layout: single
classes: wide
title: "[Kafka] Kafka Keys Message Ordering"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 을 사용해 메시지를 생산하고 소비할 때 토픽의 파티션 및 소바자 구성에 따른 메시지 순서에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Producer
    - Kafka Consumer
    - Partition
    - Topic
    - Keys
toc: true
use_math: true
---  

## Kafka Keys with Message Ordering
`Kafka Messaging Broker` 를 사용해 메시지를 처리할 떄, 
메시지는 키와 함께 사용해서 동일한 키를 가진 메시지들에 한해 메시지 순서를 보장 할 수 있다. 
`Kafka` 의 메시지는 `key-value` 쌍으로 구성되는데, 키는 메시지가 저장되는 파티션을 결정하는데 사용될 수 있다. 
그러므로 `Producer` 가 키와 함께 발송된 메시지는 동일한 `Topic Partition` 에 기록되고, 
`Consumer` 는 각 `Partition` 의 메시지는 순서대로 읽어들이기 때문에 메시지 순서가 유지 될 수 있다.  

본 포스팅과 관련된 예제는 [kafka-keys-message-ordering](https://github.com/windowforsun/kafka-keys-message-ordering-demo)
에서 전체 코드를 확인해 볼 수 있다.

### Topic Partitions
하나의 `Topic` 은 하나 이상의 `Partition` 을 가지게 된다. 
즉 이는 `Topic` 에 저장되는 메시지들은 `N` 개의 `Partition` 에 나눠 저장된다고 말할 수 있다. 
`Producer` 가 `Topic` 에 메시지를 발송하면, 해당 메시지는 `Topic Partition` 중 하나에 작성된다. 
그리고 `Topic` 을 구독하는 `Consumer Group` 의 단일 `Consumer` 중 하나가 특정 `Topic Partition` 의 메시지를 소비하게 되는 것이다. 
아래 그림은 이를 도식화해 표현한 것이다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-keys-message-ordering-1.drawio.png)

위 그림을 보면 메시지는 카 없이 발송되고 있으므로, 
어느 `Topic Partition` 에 기록될지는 보장할 수 없다. 
각 `Topic Partition` 에는 순서 보장이 필요로하는 메시지들의 순서와 그룹별 색상과 `m` 라는 문자열 변호로 표현이 돼있다. 
예를 들어 메시지 그룹 중 붉은색 `m1` 은 0번과 2번 파티션에 기록이 된 것을 볼 수 있다. 
즉 해당 `Topic` 의 모든 `Partition` 을 구독하는 `Consumer` 는 기대하는 메시지의 순서를 전혀 보장 받을 수 없게 된다. 
붉은색 `m1` 메시지 유형의 가장 첫번째 메시지는 0번 파티션에 있지만, 
2번 파티셭에 있는 3번째 메시지가 먼저 소비될 수 있다는 의미이다.  


### Scalability
`Kafka` 에서 `Topic` 을 구성하는 `Partition` 은 높은 확장성을 제공하는 중요한 요소이다. 
특정 `Topic` 메시지 처리의 처리량을 증가시키고 싶다면 `Topic` 의 `Parition` 수와 `Consumer Group` 을 구성하는 
`Consumer` 수를 늘리는 방법으로 가능하다. 
앞선 그림에서는 3개로 구성된 `Partition` 을 단일 `Consumer` 가 모두 구독하고 있었지만, 
아래 그림을 보면 4개의 `Consumer` 가 자신에게 할당된 `Partition` 을 구독하고 있는 것을 확인할 수 있다. 
결과적으로 앞선 그림보다 아래 그림이 처리량 입장에서는 큰 폭으로 증가했다고 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-keys-message-ordering-2.drawio.png)


언급한 것처럼 처리량은 큰폭으로 증가했지만, 
메시지 순서는 더욱 지켜지기 어렵게 된 상태이다. 
단일 `Consumer` 의 경우에는 동일 인스턴스에서 서로 다른 `Partition` 에 저장된 메시지로 
붉은색 `m1` 메시지의 소비되는 순서에만 고민이 했었다. 
하지만 여러 `Consumer` 를 사용하는 현상태에서는 각 `Parition` 마다 
할당된 `Consumer` 가 다르므로 붉은색 `m1` 메시지가 각 다른 `Consumer` 에게 전달되므로 
메시지 처리의 순서는 더욱 예측이 어려워졌다.  

그림을 보면 `Partition` 의 수는 3개이지만, `Consumer` 의 수는 4개이다. 
이런 상태에서는 `Consumer` 하나는 어떤 `Partition` 도 할당받지 못한 상태로 남게된다. 
불필요한 자원 낭비일 수도 있으나, 유사시 특정 `Consumer` 가 가용 불가 상태가 된다면 해당 `Consumer` 가 
리밸런스 동작을 통해 바로 `Parititon` 을 이어 처리하는 식으로 장애대응용으로 사용할 수 있다.  

위와 처럼 `Partition` 수와 `Consumer` 수가 맞지않는 경우 `Partition` 수를 늘리는 것도 고민해 볼 수 있다. 
`Partition` 수를 늘리는 것에는 사전에 신중한 검토와 고려후 진행돼야하는 작업이다. 
`Partition` 수는 늘린 이후에는 줄일 수 없고, 
`Partition` 의 수를 늘렸을 때 메시지 키를 사용하는 경우에도 `Partition` 할당이 달라져 기존 순서와 다르게 처리 될 수 있다.  
그리고 키 유형 종류에 따라 사용되지 않는 `Partition` 이 생길 수 있기 때문에 유의해야 한다.  


### Message Keys
앞서 언급한 것처럼 메시지 키를 사용하면 동일한 키를 가진 메시지들은 동일 `Partition` 에 기록되므로, 
메시지 순서를 보장하기 위한 용도로 사용될 수 있다. 
물론 `Producer` 에서 메시지를 전송 할때 `Partition` 을 직접 지정하는 것도 가능하지만, 
메시지 키를 사용하는 방법이 일반적이다.  

메시지가 `key-value` 로 구성될 때 값과 마찬가지로 카에도 `String`, `Integer,` Json` 그리고 `Avro` 와 같이 
다양한 포멧을 지정해서 사용할 수 있다. 
작성된 키는 `Producer` 에서 해시되고 해당 해시를 통해 메시지가 기록될 `Partition` 이 결정된다. 
해시값이 없거나 키가 없는 상황이라면 `Kafka` 는 `Round-Robin` 이나 사용하는 `Partitioning` 로직을 바탕으로 
최대한 고르게 분산시킨다.  



### Message Ordering
아래 그림은 가장 첫번째 예시 그림에서 각 메시지 유혈벌로 키를 추가한 상황을 그려본 것이다. 
4개의 메시지 키 유형이 있고, 이 유형들이 3개의 `Partition` 에 기록되는 상황이다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-keys-message-ordering-3.drawio.png)


동일한 메시지 키들은 같은 `Partition` 에 저장되는 것을 확인 할 수 있다. 
하나의 `Partition` 에 하나의 메시지 키만 사용되는 것이 아니라 동일한 해시값이라면, 
여러 메시지 키가 함께 기록 될 수 있다. 
하지만 `Consumer` 는 `Partition` 기준으로 메시지를 구독하고 소비하기 때문에, 
메시지를 처리하는 `Consumer` 입장에서는 메시지 키를 기준으로 봤을 때 메시지 순서를 보장 받을 수 있다.  


### Concurrent Consumers
`Spring Kafka` 에서 소개하는 `ConcurrentKafkaListenerContainerFacory` 를 사용하면 
`Kafka Application` 에서 메시지 처리의 효율을 높이고, 
높잡한 처리 로직을 간소화 할 수 있다. 
`Kafka Message Listener` 의 관리의 목적으로 사용되는데, 
`KafkaListenerContainerFactory` 인터페이스의 구현체로 `Kafka` 메시지를 구독하는 `ListenerContainer` 를
생성하는 기능을 제공한다. 
그리고 이 팩토리 클래스를 통해 다수의 `ConcurrentMessageListenerContainer` 인스턴스를 생성하고 관리할 수 있다. 
이런 `ListenerContainer` 들은 여러 스레드나 태스크에서 동시에 메시지를 처리하도록 지원한다.  

예제 코드에서는 동시성을 3으로 지정한 상태이다. 
이는 하나의 `Spring Boot Application` 에서 3개의 `Consumer` 인스턴스를 사용하는 것임을 기억하자. 

```java
final ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
factory.setConsumerFactory(consumerFactory);
factory.setConcurrency(3);
```

그리고 예제에서 사용하는 `Topic` 의 `Partition` 은 총 10개로 구성돼 있다. 
`Topic` 의 `Partition` 구성 정보는 `Kafka CLI Tool` 을 통해 아래와 같이 확인 가능하다.  


```
$ kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group demo-consumer-group

GROUP               TOPIC              PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                         HOST            CLIENT-ID
demo-consumer-group demo-inbound-topic 0          0               0               0               consumer-demo-consumer-group-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     consumer-demo-consumer-group-1
demo-consumer-group demo-inbound-topic 1          0               0               0               consumer-demo-consumer-group-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     consumer-demo-consumer-group-1
demo-consumer-group demo-inbound-topic 2          0               0               0               consumer-demo-consumer-group-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     consumer-demo-consumer-group-1
demo-consumer-group demo-inbound-topic 3          0               0               0               consumer-demo-consumer-group-1-85eebec6-36c5-4a94-a336-de3a837f1a8b /172.23.0.3     consumer-demo-consumer-group-1
demo-consumer-group demo-inbound-topic 4          0               0               0               consumer-demo-consumer-group-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     consumer-demo-consumer-group-2
demo-consumer-group demo-inbound-topic 5          0               0               0               consumer-demo-consumer-group-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     consumer-demo-consumer-group-2
demo-consumer-group demo-inbound-topic 6          0               0               0               consumer-demo-consumer-group-2-11eeb7ba-7ff7-45d6-a576-8f123bdae678 /172.23.0.3     consumer-demo-consumer-group-2
demo-consumer-group demo-inbound-topic 7          0               0               0               consumer-demo-consumer-group-3-d61c357b-eeb8-4600-9815-ee767ca6fe8a /172.23.0.3     consumer-demo-consumer-group-3
demo-consumer-group demo-inbound-topic 8          0               0               0               consumer-demo-consumer-group-3-d61c357b-eeb8-4600-9815-ee767ca6fe8a /172.23.0.3     consumer-demo-consumer-group-3
demo-consumer-group demo-inbound-topic 9          0               0               0               consumer-demo-consumer-group-3-d61c357b-eeb8-4600-9815-ee767ca6fe8a /172.23.0.3     consumer-demo-consumer-group-3
```


예제에서 `Producer` 가 메시지 키만 사용한다면 각 메시지는 키값의 해시에 매핑되는 `Partition` 에 기록이 되고, 
`Consumer` 는 자신에게 할당된 `Partition` 만 소비하므로 순서 보장이 가능하다. 
예제의 `Spring Boot Application` 에서  `Consumer Group` 내 `Consumer` 수는 3개 이므로, 
각 `Consumer` 마다 3~4개의 `Partition` 을 할당받게 되는 식이 된다.  


### Producer Retry
`Producer` 실행 중 네트워크 실패와 같은 일시적인 오류로 메시지 발송을 재시도 할 수 있다. 
이런 `Producer Retry` 를 제어할 수 있는 몇가지 옵션들이 있는데, 
이런 옵션의 설정에 따라 메시지 순서에 영향을 줄 수 있음을 기억하자. 
아래는 `Producer Retry` 와 관련된 주요 옵션들이다. 
관련해서 좀 더 자세한 내용과 설명은 [Idempotence Producer]({{site.baseurl}}{% link _posts/kafka/2024-02-10-kafka-practice-kafka-idempotent-producer.md %})
에서 확인 가능하다. 


properties|desc|default
---|---|---
retries|최대 재시도 수|2147483647
enable.idempotence|true 인 경우 메시지가 한 번만 작성되도록 한다.|true
max.in.flight.requests.per.connection|`ack` 확인 없이 전송 할 수 있는 최대 요청 수|5

만약 아래와 같은 옵션인 상태에서 `Producer Retry` 동작이 발생한다면 메시지 순서를 보장 받을 수 없다.  

```
retries > 0
enable.idempotence = false
max.in.flight.requests.per.connection > 1
```  

`enable.idempotence=false` 인 상태에서 일시적 오류로 재시도가 발생하면 아래와 같은 상황을 가정할 수 있다. 

1. 첫 번째 메시지를 전송한다. 
2. 네트워크 지연 또는 일시적인 문제로 `Kafka Broker` 로 전송이 실패한다. 
3. 첫 번째 메시지의 재시도를 수행한다. 
4. 두 번째 메시지를 전송한다.(첫 번째 메시지 재시도를 하는 동안)
5. 두 번째 메시지는 성공적으로 `Topic Partition` 에 기록된다. 
6. 첫 번째 메시지 재시도가 성공해 `Topic Partition` 에 기록된다. 


```
retries > 0
enable.idempotence = true
max.in.flight.requests.per.connection > 5
```  

먼저 `enable.idempotence=true` 인 경우 각 메시지 마다 `PID`, 메시지 번호가 부여되므로 
중복 쓰기와 순서 보장에 도움이 된다. 
하지만 `max.in.flight.requests.per.connection > 5` 이기 때문에 순서 보장에 실패하게 된다. 
순서 보장을 받을 수 있는 설정은 아래와 같은데 관련 자세한 설명은 [Idempotence Producer]({{site.baseurl}}{% link _posts/kafka/2024-02-10-kafka-practice-kafka-idempotent-producer.md %})
에서 확인 할 수 있다.  

```
retries > 0
enable.idempotence = true
max.in.flight.requests.per.connection <= 5
```  

