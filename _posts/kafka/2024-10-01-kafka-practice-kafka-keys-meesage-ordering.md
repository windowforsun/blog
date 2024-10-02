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
