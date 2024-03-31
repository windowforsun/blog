--- 
layout: single
classes: wide
title: "[Kafka] Kafka Producer Acks"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Producer 의 성능과 지속성에 영향을 미치는 설정인 Akcs 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka
    - Kafka Producer
    - Producer
    - Acks
    - Producer Acks
toc: true
use_math: true
---  

## Producer Acks
`Producer` 를 사용할 떄 `Producer acks` 설정은 `Performance` 와 `Durability` 에 영향을 미치는 설정이므로 구성에 따른 동작과 특징을 잘 알아둘 필요가 있다. 
이는 `Producer` 가 메시지를 `Kafka Broker` 의 `Topic Partition` 에 작성할 때 `Partition Replica` 에 쓰기 확인 동작을 결정하는 설정 값이다.  


### Config
`akcs` 설정은 `0`, `1`, `all` 과 같이 3가지 설정이 가능하다.  

`acks = 0` 은 `Producer` 가 메시지를 `Partition` 에 전송 후 쓰기 완료 응답인 `ack` 를 받기 전에 
`offset` 을 `commit` 하고 다음 메시지 처리를 수행한다. 
그러므로 `Partition` 에 발송한 메시지 쓰기의 성공/실패 여부는 알 수 없고 상관하지 않는 설정이다.  

`acks = 1` 은 `Producer` 가 `Partition Leader` 의 `ack` 만 대기 한 후 성공하면 다음 메시지 처리를 이어서 진행한다. 
하지만 그외 `Partition Follower` 의 메시지 쓰기 성공/실패는 알 수 없다.  

`acks = all` 은 메시지를 발송한 후 `min.insync.replicas` 설정 값으로 설정된 만큼 `Partition` 의 쓰기 성공/실패 결과를 대기한다. 
즉 `min.insync.prelicas` 옵션을 통해 얼만큼의 `Partition` 에 대한 메시지 쓰기 성공/실패를 결과를 받고 다음 메시지 처리를 수행 할지 설정 가능하다. 
만약 해당 `Topic Partition` 의 실제 `Replica` 수가 설정값 보다 적은 경우 오류가 발생하고, `Producer` 는 쓰기 재시도를 수행하게 된다.  

이러한 `acks` 는 성능과 내구성을 결정 할 수 있는 설정값이다. 
`Producer` 가 발송한 메시지가 `Partition` 의 얼만큼의 복제본까지 성공/실패를 확인할지에 대한 것으로, 
그 수가 적을 수록 성능은 향상될 수 있지만 메시지 유실 위험도는 더 커진다.  

`acks` 옵션에 대한 내용을 다시 정리하면 아래와 같다.  

value| Performance & Durability  |Desc
---|---------------------------|---
0| Performance > Durability  |발송한 메시지의 성공/실패를 어떠한 확인하지 않고 다음 메시지를 처리한다. 메시지 유실 위험이 가장 크다. 
1| Performance >= Durability |발송한 메시지에 대해서 `Partition Leader` 에 대한 성공/실패만 확인한다. 메시지 유실이 발생 할 수 있다. 
all| Performance < Durability  |발송한 메시지에 대해서 `Partition Follower` 까지 성공/실패를 확인한다. 메시지 유실 위험이 가장 적다. 


### Behaviour
`acks` 설정에 따른 동작을 좀 더 자세히 알아보기 위해서 아래와 같은 시나리오를 사용한다. 
`Service` 는 `Inblound Topic` 으로 부터 메시지는 소비하는 `Consumer` 이자 `Outbound Topic` 으로 결과 메시지를 전송하는 `Producer` 이다. 
그리고 `Outbound Topic` 에 성공적으로 메시가 발송됐으면 `Inbound Topic` 의 `offset` 을 `commit` 한다. 
이를 도식화하면 아래와 같다. 


![그림 1]({{site.baseurl}}/img/kafka/producer-acks-1.drawio.png)

가장 먼저 알아볼 옵션은 `acks = 0` 이다. 
해당 옵션을 `Partition` 의 쓰기 결과 여부를 대기하지 않기 때문에 가장 좋은 성능을 보여주지만 메시지 유실 위험은 가장 높다. 
아래 플로우를 보면 `Partition Leader` 에 메시지가 일시적인 이슈로 `Replication event` 발송을 실패했다. 
하지만 `Producer` 는 메시지 발송 후 곧 바로 `Inbound Topic` 의 `offset` 을 `commit` 하기 때문에 
해당 메시지는 유실된다.  

![그림 1]({{site.baseurl}}/img/kafka/producer-acks-2.png)

동일한 시나리오에서 `acks = 1` 의 동작은 `Partition Leader` 동작만 확인 한후 `ack` 를 받는 식으로 최소한의 성능 비용으로 메시지 손실 가능성을 줄인다. 
여기서 발생할 수 있는 메시지 손실 위험은 `Partition Leader` 에서 다른 복제본으로의 복제 수행전에 중단될 수 있다는 점이다. 
이럴 경우 `Producer` 는 손실된 메시지를 다시 발송하지 않기 때문에 복제본에서 해당 메시지의 손실은 발생한다.  

![그림 1]({{site.baseurl}}/img/kafka/producer-acks-3.png)

`ack = all` 인 설정에서 동일 시나리오를 살펴본다면, 
`Partiton Leader` 는 `Producer` 로 부터 메시지를 받은 뒤 `ISR(In-Sync Replica)` 인 `Partition Follower` 를 대상으로 복제 동작을 수행하기 때문에 바로 `ack` 를 전송하지 않는다. 
`ISR` 인 `Partition Follower` 들의 모든 복제가 완료된 이후 `ack` 를 보내 다음 메시지 처리가 수행될 수 있도록 한다. 
하지만 여기서 일시적인 이슈로 복제 동작이 실패하거나 오래 걸린다면, 
`Producer` 측에서는 타임아웃이 발생하고 `ack` 수신에 실패한 메시지를 다시 처리 후 전송하게 되므로 메시지 손실은 발생하지 않는다.  

![그림 1]({{site.baseurl}}/img/kafka/producer-acks-4.png)

아래와 같이 `ack = all` 일 때 모든 `ISR` 에 대한 복제까지 타임아웃내 완료하고 `ack` 를 `Producer` 에게 보닌다면, 
`Producer` 는 `offset` 을 `commit` 하고 이후 메시지 처리르 재개한다. 
이 시점 이후 부터는 `Partition Leader` 일시적으로 종료되는 현상이 있더라도 복제까지 완료한 메시지에 대해서는 손실되지 않는다.  

![그림 1]({{site.baseurl}}/img/kafka/producer-acks-5.png)


각 설정에 대한 동작을 살펴보았을 떄, 
`ack = all` 는 메시지 손실을 방지할 수 있는 설정이다. 
`all` 인 경우 `ISR` 까지 모두 복제 후 `ack` 를 받을 수 있어 `Producer` 입장에서 약간의 성능 저하가 발생 할 수 있다. 
하지만 이런 성능저하는 무시할 수 있는 수준으로 알려져 있다.  

---  
## Reference
[Kafka Producer Acks](https://www.lydtechconsulting.com/blog-kafka-producer-acks.html)  
[Kafka Producer Acks](https://www.popit.kr/kafka-%EC%9A%B4%EC%98%81%EC%9E%90%EA%B0%80-%EB%A7%90%ED%95%98%EB%8A%94-producer-acks/)  
[Producer Config acks](https://kafka.apache.org/documentation/#producerconfigs_acks)  
