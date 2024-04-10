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


### Stateless Retry
`Java Kafka Client` 는 `Stateless Retry` 를 지원하는데 관련 내용은 아래와 같다.  

- 재시도 동작은 `Batch` 처리의 `poll` 동작에서 발생한다. 
- `Consumer` 의 `poll` 동작은 `poll.timeout` 이내 수행 완료돼야 한다. 이는 `batch` 처리에서 포함되는 모든 재시도, 총 처리 시간(REST API, DB, .. 호출), 재시도의 `delay`, `backoff` 시간 등이 포함된다.  
- `poll.timeout` 의 기본 값은 5분 이고, `batch` 처리 레코드 수의 기본 값은 500개 이다. 이를 단 위 레코드 당 처리 소요시간으로 계산하면 600ms 이내 하나의 메시지 처리를 완료해야 한다는 의미이다. 
- 만약 `poll.timeout` 안에 메시지 처리가 완료되지 못하면, 메시지는 중복 될 수 있다. 

아래 도식화한 플로우를 보면 `Consumer` 가 `Inbound Topic` 로 부터 메시지를 소비한 후 결과에 대한 이벤트를 `Outbound Topic` 으로 보내기전, 
`Third party` 서비스에 `REST API` 를 호출하고 있다.  

.. 그림 ..

1-1. 첫 번쨰 `Consumer` 는 `Inbound Topic` 으로부터 메시지를 소비한다. 
1-2. 첫 번째 `Consumer` 는 `Third party` 서비스에게 `REST API` 호출을 수행하지만 `503` 에러로 호출은 실패한다. 
1-3. 첫 번째 `Consumer` 는 재시도 설정에 따라 `Thrid party` 서비스에세 `REST API` 호출을 재시도 한다.

첫 번째 `Consumer` 가 재시도를 수행하는 과정에서 `poll.timeout` 이 발생 했고, 
`Kafka Broker` 는 `Rebalancing` 을 트리거해 첫 번째 `Consumer` 를 `Consumer Group` 에서 제거하고 
두 번째 `Consumer` 를 동일한 `Partition` 에 할당한다.  

2-1. 두 번째 `Consumer` 는 동일한 메시지를 `Inbound Topic` 으로부터 소비한다. 
2-2. 두 번째 `Consumer` 가 `Third party` 에 `REST API` 요청을 보내는 시점 부터는 `200` 으로 성공한다. 
1-4. 첫 번째 `Consumer` 도 `Third party` 요청에 성공하고, 처리 결과를 `Outbound Topic` 에 게시한다. 
2-3. 두 번째 `Consumer` 또한 처리 결과를 `Outbound Topic` 에 게시한다. 

`Outbound Topic` 을 사용해서 이후 `downstream` 서비스는 결과에 따른 처리를 이어서 수행한다. 
위와 같은 처리 흐름으로 `Outbound Topic` 에서 결과 이벤트가 구분될 수 있더라도, 
`downstream` 에서 이런 중복 수행에 대한 내용을 처리하는 것은 권장되지 않는다.  

`Third party` 서비스가 몇 시간에 걸쳐 가용 불가능 상태에 빠진 상황을 가정해보자. 
재시도 횟수 값 등을 올려 몇 시간 동안 재시도를 수행 할 수는 있다. 
하지만 `Consumer` 에 설정된 `poll.timeout` 이 발생 할 수 있으므로 무한한 재시도는 사실살 불가능하다. 
그러므로 적절한 수치의 재시도에도 실패하는 경우에는 `Dead letter topic` 과 같은 별도의 토픽으로 메시지를 보내 이후 다시 처리가 될 수 있도록 해야한다.  


#### Stateless Retry Strategy
`Stateless Retry` 를 적용할 떄 충분한 재시도 수행 시간을 확보 할 수 있는 몇가지 전략이 있다. 
이후 설명하는 각 전략은 각 요소마다 장/단점이 있으므로 애플리케이션의 요구사항을 잘 파악하고 신중하게 선택해야 한다.  

- `poll.timeout` 과 배치 처리 옵션 조정

충분한 재시도 시간 확보를 위해 `poll.timeout` 을 늘리고, 배치 처리 옵션의 수치를 줄이는 방법이 있다. 
이러한 방식은 `poll.timeout` 을 늘려 `Consumer` 가 실패상태로 판별되지 않는 재시도 수행 시간을 늘릴 수 있고, 
배치 처리 옵션을 줄이는 것으로 `poll.timeout` 내에 더 많은 재시도를 수행 할 수 있다. 
하지만 이는 `batch.size` 가 줄어듬에 따라 처리량이 줄어들게 되고, 
재시도 시간은 `poll.timeout` 에 크에 의존한다는 점이 있다.  

- `Retry topic` 사용하기

재시도가 필요한 메시지의 경우 재시도 만을 위해 구성한 별도의 `Retry topic` 에 메시지를 전달하는 방식이다. 
이는 기존 `Consumer` 의 처리량과 안정성은 그대로 가져가면서 충분한 재시도 수행 시간을 확보하는 것이다. 
하지만 추가적인 토픽에서 재시도 처리를 할 추가적인 비지니스 구성 등이 필요하므로 복잡성이 늘어나고, 
재시도가 성공하더라도 `upstream` 의 메시지 순서가 지켜지지 않을 수 있다. 

- `Kafka Broker` 의 재시도 사용하기

`Kafka Client` 에서 별도의 재시도 로직을 구성하는 것 대신 `RetryableException` 을 발생시키면, 
예외가 발생한 메시지는 다음 `poll()` 동작에서 다시 소비된다. 
이는 `Kafka Broker` 의 재시도를 사용하는 방법으로 중복 메시지라던가 추가적인 비지니스, 순서 같은 이슈가 발생하지 않는 방법이다. 
하지만 재시도 횟수를 제한 할 수 없어 `Poison pill` 메시지가 발생할 수 있고, 
`back-off` 설정이라던가 재시도에 대한 구체적인 옵션을 지정할 수 없다는 점이 있다.  

