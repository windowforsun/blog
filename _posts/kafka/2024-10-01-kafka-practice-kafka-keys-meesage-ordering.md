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

### Topic Partitions
하나의 `Topic` 은 하나 이상의 `Partition` 을 가지게 된다. 
즉 이는 `Topic` 에 저장되는 메시지들은 `N` 개의 `Partition` 에 나눠 저장된다고 말할 수 있다. 
`Producer` 가 `Topic` 에 메시지를 발송하면, 해당 메시지는 `Topic Partition` 중 하나에 작성된다. 
그리고 `Topic` 을 구독하는 `Consumer Group` 의 단일 `Consumer` 중 하나가 특정 `Topic Partition` 의 메시지를 소비하게 되는 것이다. 
아래 그림은 이를 도식화해 표현한 것이다.  

.. 사진 ..

위 그림을 보면 메시지는 카 없이 발송되고 있으므로, 
어느 `Topic Partition` 에 기록될지는 보장할 수 없다. 
각 `Topic Partition` 에는 순서 보장이 필요로하는 메시지들의 순서와 그룹별 색상과 `g` 라는 문자열 변호로 표현이 돼있다. 
예를 들어 메시지 그룹 중 붉은색 `g1` 은 0번과 2번 파티션에 기록이 된 것을 볼 수 있다. 
즉 해당 `Topic` 의 모든 `Partition` 을 구독하는 `Consumer` 는 기대하는 메시지의 순서를 전혀 보장 받을 수 없게 된다. 
붉은색 `g1` 메시지 유형의 가장 첫번째 메시지는 0번 파티션에 있지만, 
2번 파티셭에 있는 3번째 메시지가 먼저 소비될 수 있다는 의미이다.  


### Scalability
`Kafka` 에서 `Topic` 을 구성하는 `Partition` 은 높은 확장성을 제공하는 중요한 요소이다. 
특정 `Topic` 메시지 처리의 처리량을 증가시키고 싶다면 `Topic` 의 `Parition` 수와 `Consumer Group` 을 구성하는 
`Consumer` 수를 늘리는 방법으로 가능하다. 
앞선 그림에서는 3개로 구성된 `Partition` 을 단일 `Consumer` 가 모두 구독하고 있었지만, 
아래 그림을 보면 4개의 `Consumer` 가 자신에게 할당된 `Partition` 을 구독하고 있는 것을 확인할 수 있다. 
결과적으로 앞선 그림보다 아래 그림이 처리량 입장에서는 큰 폭으로 증가했다고 할 수 있다.  


.. 사진 ..

언급한 것처럼 처리량은 큰폭으로 증가했지만, 
메시지 순서는 더욱 지켜지기 어렵게 된 상태이다. 
단일 `Consumer` 의 경우에는 동일 인스턴스에서 서로 다른 `Partition` 에 저장된 메시지로 
붉은색 `g1` 메시지의 소비되는 순서에만 고민이 했었다. 
하지만 여러 `Consumer` 를 사용하는 현상태에서는 각 `Parition` 마다 
할당된 `Consumer` 가 다르므로 붉은색 `g1` 메시지가 각 다른 `Consumer` 에게 전달되므로 
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

