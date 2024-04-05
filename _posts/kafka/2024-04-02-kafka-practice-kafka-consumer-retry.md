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

## Consumer Retry
`Consumer` 가 메시지를 소비한 후 처리하는 과정에서 일시적인 네트워크 오류 등은 재시도 동작을 통해 충분히 해결 가능하다. 
이는 메시지 처리 과정에서 재시도 가능한 예외가 발생하면 재시도가 수행될 수 있도록 구성이 필요하다. 
위와 같이 `Consumer` 의 재시도 동작을 구현할 때 고려할만한 몇가지 요소에 대해 알아 본다.  

재시도가 가능한 예외에 대한 케이스로는 아래와 같은 것들이 있을 수 있다. 

- `Third party` 서비스의 요청이 `502` 와 같은 에러 응답으로 내려오는 경우(해당 서비스 복구시 정상화 될 수 있다.)
- `DB` 에 `INSERT`, `UPDATE` 를 수행 할때 `lock` 관련 에러 
- 일시적인 `DB` 커넥션 에러
- 재시도시 해결 될 수 있는 `Kafka` 예외 

이러한 재시도로 해결가능한 예외로 인해 애플리케이션 자체가 실패하게 되면, 
이후 `downstream` 에 대한 구성이 불안해 질 수 있다. 
복구를 위해서는 실패한 이벤트에 대해 다시 재생해 메세지 처리를 수행한다 거나 하는 식으로 별도의 처리가 필요할 수 있다.  

재시도에 대한 동작을 구현 할때 아래와 같은 고려사항이 있다. 

- `Consumer` 가 재시도를 수행하는 동안 사용하는 `Topic Partition` 의 추가적인 메시지 소비는 수행되지 않는다. 
- 재시도 동작을 위해서는 추가적인 소요시간이 필요할 수 있다. 
- `Poison pill` 메시지 위험(성공할 수 없는 메시지)
- `Consumer` 가 재시도를 수행하는 과정에서 `Kafka Broker` 는 `Consumer` 가 죽었다고 판단하여 중복 메시지가 소비될 수 있다. 
