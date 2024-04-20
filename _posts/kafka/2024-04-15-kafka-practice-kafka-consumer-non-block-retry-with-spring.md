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


### Blocking retry
애플리케이션에서 메시지를 `poll` 한 뒤 `downstream` 으로 전달하는 시나리오를 가정했을 때, 
`downstream` 의 일시적인 이슈로인한 메시지 처리 실패와 같은 상황에서는 `Blocking retry` 가 적합하다. 
이유는 동일 `Topic Partition` 에 있는 이후 메시지들도 같은 이유로 계속해서 실패될 가능성이 높기 때문이다. 
즉 위와 같은 상황에서는 `Non-blocking retry` 를 적용하는 것은 큰 이미가 없고 `Blocking retry` 가 적합한 방식이다. 
가장 처음 실패한 메시지를 재시도 수행하던 중 재시도가 성공하면 이후 메시지들도 모두 실패 없이 처리 될 수 있는 상황이기 때문이다.  

.. 그림 ..

위 그림을 보면 `Consumer` 가 `m1` 메시지를 소비하고
처리 과정 중 `Third-party` 서비스에서 `POST` 요청을 보내지만 해당 서비스의 일시적인 이슈로 요청은 실패한다. 
이런 상황에서 이후 메시지가 `Non-blocking` 방식으로 처리된다고 하더라도 모두 동일한 이유로 실패할 가능성이 높기 때문에, 
`Non-blocking retry` 를 적용하기에는 큰 이득이 없는 상황이다. 
그러므로 `Blocking retry` 가 적절하고 `m1` 의 재시도 수행이 성공하면 이어서 처리되는 메시지들도 모두 성공하게 될 것이다.  

이러한 특성을 가진 `Blocking retry` 는 `Topic Partition` 이 `Blocking` 돼어 재시도 수행까지 이후 메시지 소비가 멈춘다는 단점이 있지만, 
메시지의 순서 보장된다는 장점이 있다. 
그러므로 메시지 순서가 보다 중요한 서비스에서는 `Blocking retry` 가 적합한 경우도 있기 때문에 요구사항에 맞는 전략을 선택해야 한다.  


### Non-Blocking retry
`Non-Blocking retry` 는 메시지에 재시도 수행이 있더라도 동일 `Topic Partition` 의 이후 메시지들이 계속해서 처리 될 수 있음을 보장한다. 
이러한 특성을 잘 보여주는 예로 `DB` 에 `INSERT` 와 `UPDATE` 를 수행하는 애플리케이션을 예로 들 수 있다. 
특정 레코드의 `UPDATE` 메시지를 수신 했지만, 아직 `DB` 에 해당 레코드가 없는 경우 메시지 처리는 실패하게 된다. 
이 상황에서 `UPDATE` 메시지는 `Non-Blocking retry` 로 계속해서 재시도를 수행하고, 
메시지는 계속해서 소비하게 될 떄 특정 시점 이후 `INSERT` 메시지가 수신되면 재시도를 수행하던 `UPDATE` 까지 모두 성공적인 처리를 완료 할 수 있다.  

.. 그림 ..

위 시나리오에서 `Non-Blocking retry` 는 기존 `Topic Partition` 에서 소비되는 메시지 중 `UPDATE` 메시지는 계속해서 처리되지만, 
재시도가 필요한 `INSERT` 메시지는 다른 `Topic Partition` 에서 처리된다. 
그러므로 기존 `Topic Partition` 의 메시지 처리는 중단없이 계속해서 처리될 수 있지만, 
재시도 수행이 필요로하는 `INSERT` 메시지는 별도의 `Topic Partition` 에서 수행되므로 기존 `Topic Partition` 의 
메시지 순서는 보장될 수 없음을 기억해야 한다.



### Spring Kafka Retryable Topics
`Spring Kafka` 는 `Non-Blocking retry` 을 위해 앞서 말한 방식과 동일하게 `Retryable topics` 를 사용한다. 
그리고 `Retryable topics` 를 사용하는 `Consumer` 는 기존 `Topic` 를 사용하는 `Consumer` 와 
별도의 인스턴스를 `Spring` 이 실행한다. 
이러한 방식으로 동일한 `Consumer` 가 기존 `Topic` 과 `Retryable topics` 를 소비하지 않도록 한다.  

만약 이벤트가 여려번 혹은 장기간 재시도가 필요한 경우 단일 `Retryable topics` 를 사용하는 것이 아니라, 
여러개의 `Retryable topics` 를 구성해서 사용 할 수 있다. 
다수를 사용하면 그만큼 관리 포인트가 늘어난다는 단점이 있고, 단일을 사용하면 관리 포인트가 줄어든다는 장잠이 있다. 
여러번 혹은 장기간 재시도를 수행하는 상황에서 단일 `Retryable topics` 는 해당 토픽이 중단될 수 있다는 단점이 있다. 
이런 상황에서 `N` 개 이상의 `Retryable topics` 를 사용해서 비지니스에 더욱 맞는 재시도 수행을 중단을 줄이면서 구성할 수 있다. 
그리고 이러한 재시도에도 실패한 메시지의 경우 `Dead letter` 토픽을 구성해서 해당 토픽으로 보내,
실패에 대한 로그를 남기거나 `Consumer` 가 다시 해당 토픽을 사용하도록 해서 추후 처리를 만들 수 있다.  


`Spring Kafka` 에서 `Retryable topics` 는 `@RetryableTopic` 어노케이션을 추가하는 방식으로 
재시도를 수행할 `Topic` 를 구성 할 수 있다. 
해당 어노테이션의 주요 설정 옵션에 대한 설명은 아래와 같다.  

Properties|Desc
---|---
attempts|`Dead letter` 토픽으로 보내지기 전까지 최대 시도 횟수
backoff|각 재시도에 적용할 `backoff` 로, 재시도간 지연시간을 의미
timeout|`Dead letter` 토픽으로 보내지기 전까지 재시도의 최대 시간
fixedDelayTopicStrategy|단일, 다수 재시도 토픽을 사용할지 결정
topicSuffixStrategy|재시도 토픽의 증가 인덱스 및 지연 값을 접미사로 붙일지
dltStrategy|`Dead letter` 토픽 생성 여부와 전송 실패 처리 방안
include/includeNames|재시도를 수행할 예외
exclude/excludeNames|재시도를 수행하지 않을 예외
traversingCauses|포함/제외되는 예외를 찾기 위해 `exception chain` 을 탐색 할지
autoCreateTopics|재시도/`Dead letter` 토픽이 자동/수동 생성 결정
replicationFactor|자동 생성된 토픽에 적용 할 `replicationFactory`
numPartitions|재동 생성된 토픽에 적용 할 `partitions` 수


이후 실제 코드 예제로 알아보겠지만,
`Spring Kafka` 를 사용하면 `Consumer` 애플리케이션에서 `Non-Blocking retry` 를 구현이 간단해진다.
`Listener` 나 `Annotation` 을 추가하는 것만으로 재시도 동작에 대한 활성화와 정의가 가능하기 때문이다.
이러한 동작은 `Spring Kafka` 라이브러리 내부적으로 수행되기 때문에 개발자는 애플리케이션에 맞는 설정 값만 잘 기입해주면 된다.
다시 한번 언급하지만 이러한 `Non-Blocking retry` 는 기존 `upstream` 의 메시지 순서가 보장되지 않는 다는 점을 꼭 기억해야 한다.


