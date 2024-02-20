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

### Publish duplication message
중복 메시지는 아래와 같은 시나리오에서 발생 할 수 있다. 

1. `Producer` 가 `Kafka Broker` 에 메시지를 전송하고 `Kafka Broker` 는 메시지는 해당하는 `Partition` 에 쓴다. 
2. 이후 `Kafka Broker` 는 `Producer` 에게 완료 신호인 `ack` 를 전송해야 하는데, 일시적인 오류로 해당 `ack` 가 유실 됐다. 
3. `Producer` 는 1번에서 전송한 메시지가 쓰여졌는지 아닌지 알 수 없으므로 다시 재전송을 시도한다. 
4. 해당 `Producer` 는 `Idempotent` 가 아니므로 `Partition` 에 메시지는 중복된다. 

![그림 1]({{site.baseurl}}/img/kafka/idempotent-producer-1.drawio.png)

`Idempotent Producer` 로 구성하게 되면, 각 `Producer` 들은 고유한 `ID`(PID) 가 할당된다. 
그리고 메시지들에는 순차적으로 증가하는 번호가 부여되게 되고, 
`Kafka Broker` 는 `ID + 메시지 번호` 와 같이 조합해서 `Duplication Message` 를 판별하고 거부할 수 있다.   

![그림 1]({{site.baseurl}}/img/kafka/idempotent-producer-2.drawio.png)


### How to make Idempotent Producer
`Kafka Producer` 의 옵션 중 하나인 `enable.idempotent` 는 중복 메시지 가능성이 있는 오류가 발생했을 때,
재시도된 메시지를 `Topic` 의 `Partition` 에 쓸 수 있는지 설정하는 옵션이다. 
여기서 일시적인 오류라는 것은 `Kafka Leader` 가 가용 불가 상태거나, `Partition` 의 복제본이 충분하지 않다거나 등이 있다. 
`true/false` 로 설정 할 수 있고, 기본 값은 `false` 이다.  

그리고 `retires` 의 옵션은 반드시 `0` 보다 큰 값으로 설정해야 한다.  (Kafka 2.0 이하 인 경우 0이 기본값)

`Idempotent` 의 성질을 지니기 위해서는 `ack` 옵션을 `all` 로 해야 한다. 
`Kafka Leader` 는 수신한 메시지에 대한 승인 응답을 하기 전에, 
최소 개수의 동기화 복제본 `Partition` 이 메시지 승인을 승인할 때까지 대기해야 한다. 
여기서 최소 개수에 대한 구성은 `min.insync.replicas` 옵션으로 가능하다.  

`ack` 옵션을 `all` 로 하게되면 성능보다는 내구성에 초점을 맞춘 `Producer` 가 된다. 
즉 메시지 중복은 피할 수 있지만 기본보다 성능저하는 피할 수 없음을 기억해야 한다.  

만약 `retries` dhqtus rkqtdl `0` 으로 설정 돼 있다면, 
`Producer` 는 메시지를 재전송 하지 않고 `dead-letter` 메시지로 전송하게 된다. (애플리케이션의 에러처리 로직에 따라 다를 순 있음)
하지만 이러한 처리는 다른 방안으로 해결 가능한 문제를 해결 불가능하도록 하고, 별도로 `dead-letter` 메시지를 처리해야 하므로 권장되지 않는 방식이다. 
위 상황에서 `Producer` 가 처음 전송한 메시지는 `Topic Partition` 에 쓰여 졌지만, `Kafka Broker` 의 `ack` 만 유실 된 상활 일 수도 있다.  

[Idempotent Consumer]({{site.baseurl}}{% link _posts/kafka/2024-01-13-kafka-practice-kafka-duplication-patterns.md %})
를 구현할 때는 코드 레벨적인 부분이 많았다면, 
`Idempotent Producer` 는 구현 관련해서 코드 레벨적인 내용은없고, 옵션과 설정 값만 잘 구성해 주면 된다.  
