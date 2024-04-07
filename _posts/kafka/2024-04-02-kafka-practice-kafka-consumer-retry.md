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

### Retry period
재시도 동작에서 중요한 것은 재시도를 수행할 시간을 결정하는 것이다. 
재시도를 수행하는 시간이 너무 짧으면 해결하는데 충분하지 않을 수 있고, 
너무 길면 이후 메시지 처리에 지연이 발생할 수 있다.

`Kafka` 는 메시지의 순서 보장을 위해 `Consumer` 에서 재시도 과정에서 재시도로 인해 사용하는 `Topic Partition` 이 차단된다. 
여기서 차단된다는 것은 재시도 수행으로 인해 `Consumer` 는 더 이상 메시지를 소비하지 않는 다는 의미로 이후 메시지에 대해서는 처리 지연이 발생하게 되는 것이다. 
`Topic Partition` 의 차단으로 인한 메시지 지연을 개선하기 위해 메시지를 빠르게 실패하도록 하는 방식도 좋은 방법은 아니다. 
그 이유는 다음 메시지도 동일한 증상으로 실패할 가능성이 높기 때문이다.  

다른 한 가지 방법은 메시지가 성공할 떄까지 재시도를 계속해서 수행하는 것이다. 
히자민 위와 같이 시간, 횟수에 제한 없이 재시도를 수행하게 되면 `Poison pill` 메시지 발생 위험도가 높아진다. 
만약 `Consumer` 재시도 성공에 필요한 `Third party` 서비스가 수 시간동안 정상으로 돌아오지 않는다면, `Consumer` 는 수 시간동안 재시도를 수행하고, 
해당되는 `Topic Partition` 은 수 시간동안 차단되는 것이다. 
이런 상황에서 위험한 점은 `Topic` 의 보존 기한을 넘어가게 되면 메시지 손실로 까지 위험이 커질 수 있다. 
그러므로 재시도의 최대 수행 기간은 고려해서 설정하는 것이 좋다.  

재시도 기간 고려에서 추가적으로 한 가지 더 고민이 필요한 건 `poll.timeout` 에 대한 내용이다. 
`Consumer` 는 `Kafka Broker` 로 부터 메시지를 `poll` 하고 `poll.timeout` 전까지 처리가 완료되지 못하면, 
`Kafka Broker` 는 해당 `Consumer` 가 실패한 것으로 판단하고 `Consumer Group` 에서 제거 후 새로운 `Consumer` 를 할당하기 위해 
`Rebalancing` 을 트리거한다. 
이러면 2번째 `Consumer` 또한 동일한 메시지를 `poll` 하게 되는데, 
이 때 기존 `Consumer` 메시지 결과를 정상적으로 처리한 상태라면 메시지의 중복 처리가 발생하게 되는 것이다. 
계속적인 `poll.timeout` 으로 2번째 `Consumer` 도 `Consumer Group` 에서 제외되면 현상은 계속 반복 될 수 있다.  

