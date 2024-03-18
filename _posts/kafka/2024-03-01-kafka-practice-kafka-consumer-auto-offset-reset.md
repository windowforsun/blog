--- 
layout: single
classes: wide
title: "[Kafka] Kafka Consumer Auto offset Reset"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Consumer 가 토픽을 구독 할때 읽기를 시작할 위치를 컨트롤 할 수 있는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Consumer
    - Offset
    - Auto Offset Reset
toc: true
use_math: true
---  

## Consumer Auto Offset Rest
`Kafka Consumer` 에서 `auto.offset.reset` 설정은 
`Consumer` 가 `Topic` 의 `Partition` 를 구독 시작하는 시점인 초기 오프셋이 없는 경우에 대한 동작방식을 정의한다. 
해당 설정을 통해 `Consumer Group` 을 구성하는 `Consumer` 들이 구독하는 `Topic` 의 `Partition` 을 시작 부분부터 읽을지, 
끝 부분부터 읽어야 할지 알 수 있다.  

### How Consumer consuming messages
모든 `Kafka Consumer` 는 `Consumer Group` 에 속하고 
어떤 `Consumer Group` 에 속하는지는 `Consumer` 의 설정 중 `group.id` 를 통해 결정된다. 
즉 모든 `Consumer` 는 `Consumer Group` 에 속하고 `Consumer Group` 에는 최소 하나 이상의 `Consumer` 가 포함된다. 
그리고 `Consumer Group` 을 구성하는 `Consumer` 가 메시지를 소비하기 위해서는 `Kafka Broker` 의 `Topic Partition` 할당을 받아야 한다. 
`Consumer Group` 을 기준으로 각 `Partition` 은 하나의 `Consumer` 만 할당되지만 `Consumer` 는 하나 이상의(혹은 모든) `Parition` 을 할당 받을 수 있다. 

`Consumer Group` 과 그 멤버인 `Consumer` 들이 생성되고 각 `Consumer` 들이 메시지를 소비하기 위해 `Partition` 을 할당 받게 된다. 
그리고 `Consumer` 가 메시지 폴링 시작전 `Partition` 의 어느 위치에서 부터 폴링을 할지 초기 시작 지점을 결정해야 한다. 
`Consumer` 에게 특정 위치의 `offset` 부터 시작해라는 설정이 없는 한 `Partition` 의 시작 부분부터 메시지를 읽는 것과 
`Consumer` 가 `Partition` 을 구독한 시점부터 메시지를 읽는 2가지 옵션이 있다.  

### How to config
우리는 위에서 설명한 `Conumer` 가 구독한 이후 메시지 폴링을 `Partition` 의 어느 곳부터 할지에 대해서는 
`auto.offset.reset` 에 아래와 같은 설정값을 지정해서 가능하다.  

value|desc
---|---
earliest|`offset` 을 가장 앞선 `offset` 인 `Partition` 의 시작 부분부터 사용하도록 재설정한다. 
latest|`offset` 을 가장 최신 `offset` 인 `Partition` 의 끝 부분부터 사용하도록 재설정한다. (기본값)
none|`Consumer Group` 에 대한 `offset` 이 없다면 예외가 발생한다. 

`Consumer Group` 의 `offset` 이 이미 주어진 있는 상황에서 위 설정 값들은 사용되지 않는다. 
위 설정 값들은 `Consumer Group` 중 특정 `Consumer` 가 멈추고 재시작 됐을 때, 
해당 `Consumer` 가 어느 부분부터 메시지를 소비해야할지 결정할 때 사용된다.  

### Earliest

![그림 1]({{site.baseurl}}/img/kafka/consumer-auto-offset-reset-1.drawio.png)


`auto.offset.resest: earliest` 로 설정한 경우 새롭게 시작된 `Consumer` 는 
자신에게 할당된 `Partition` 에 있는 모든 메시지를 처음부터 차례대로 소비하게 된다. 
위 그림을 보면 `Consumer` 가  `Partition` 의 시작부분에 있는 `message 1` 부터 소비하는 것을 알 수 있다.  

만약 `Partition` 에 수백만 개의 메시지가 있다면 전체 시스템에 큰 부하를 초래할 수 있으므로 `Partition` 의 볼륨을 잘 이해하고 해당 값을 설정해야 한다. 
`Partition` 의 데이터는 보존 용량/기간에 따라 오래된 메시지도 보관이 돼있을 수 있어 시스템을 특정 시점으로 되돌리거나 하는 상황에서 유용하게 사용될 수 있다. 
하지만 `retention.ms : -1` 철럼 설정된 경우에는 시스템 시작 시점부터 생성된 모든 메시지가 새로운 `Consumer` 가 구독을 시작 할때마다 소비 될 수 있음을 의미한다.  


### Latest

![그림 1]({{site.baseurl}}/img/kafka/consumer-auto-offset-reset-2.drawio.png)


`auto.offset.reset: latest` 로 설정 됐다면 새로운 `Consumer` 가 `Partition` 을 구독 한 이후 메시지부터 소비한다. 
위 그름을 보면 `offset` 이 3인 시점 부터 새로운 `Consumer` 가 `Partition` 구독을 시작 했으므로 `offset` 이 1과 2의 메시지는 무시하고, 
`offset 3` 부터 메시지를 소비하게 된다.  

즉 `latest` 는 기존 메시지는 소비하지 않고 스킵하는 설정인데 이는 애플리케이션의 요구사항을 잘 고려한 후 설정해야 한다.  

### Data loss
새로운 `Consumer` 가 `Consumer Group` 에 참여하는 과정에서 기존 `Consumer` 가 메시지 처리 완료전 종료되어 `offset` 커밋이 되지 않은 상황을 가정해보자. 
위와 같은 상황에서 만약 `auto.offset.reset: latest` 설저이라면 기존 `Consumer` 가 처리 중에 종료된 메시지는 새로운 `Consumer` 에게 다시 전덜되지 않고 
메시지 자체가 유실될 수 있다. 

1. `auto.offset.reset: latest` 설정으로 `Consumer Group` 은 `A`, `B` 라는 `Consumer` 로 구성돼 있다. 
2. `A Consumer` 가 `Partition` 을 할당 받고 메시지를 `poll()` 한다. 
3. `A Consumer` 는 `poll()` 한 메시지를 완전히 처리하기 전에 예외 발생으로 종료된다. (`offset` 커밋도 수행되지 않았다.)
4. `Rebalancing` 이 수행되며 `B Consumer` 가 `A Consumer` 가 할당 받았던 `Partition` 을 할당 받는다. 
5. `B Consumer` 의 `auto.offset.reset` 은 `latest` 이므로 할당 받은 이후 메시지부터 사용 할 수 있으므로 `A Consumer` 에서 비정상적으로 종료된 메시지는 받지 못하고 유실된다. 

정리하면 `A Consumer` 가 `Partition` 을 부터 메시지를 읽었지만, 해당 메시지에 대한 `offset commit` 이 이뤄지지 않았기 때문에 
오류 시나리오에서 새로운 `B Consumer` 가 동일한 `Partition` 을 할당 받았을 때 처리가 완료되지 못한 메시지가 전달될 것 같지만 실제로는 그렇지 않은 상황이 발생하는 것이다.  


### Checking offset
`Topic partition` 을 구독하는 모든 `Consumer Group` 은 자신이 사용하는 `Topic partition` 의 `offset` 을 `Kafka Broker` 의 `__consumer_offsets` 라는 `Topic` 에 저장한다. 
`Kafka` 가 기본으로 제공하는 여러 관리 스크립트 중 `kafka-consumer-groups` 스크립트를 사용하면 데이터 손실 상황이라던가 `Kafka` 운영 중 데이터관련 이슈가 있을 때 `offset` 상태를 확인함으로써 
현재 상황을 좀 더 이해할 수 있다. 
아래 명령어와 같이 `--group` 을 통해 확인하고자 하는 `Consumer Group` 을 지정하고, `--describe` 옵션을 추가하면 해당 `Consumer Group` 에 대한 정보가 출력된다. 
`test-topic` 에는 `message 1`, `message 2` 와 같은 2개의 메시지가 포함된 상태이다.  

```bash
$ kafka-topics --bootstrap-server localhost:9092 --topic test-topic --create --partitions 1
Created topic test-topic.


$ kafka-console-producer --bootstrap-server localhost:9092 --topic test-topic 
>message 1
>message 2


$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --group test-consumer-group


$ kafka-consumer-groups.sh --botstrap-server localhost:9092 --group test-consumer-group --describe

GROUP               TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                         HOST            CLIENT-ID
test-consumer-group test-topic      0          -               2               -               consumer-test-consumer-group-1-925d2211-0d76-404b-bd8e-adfd954abfd5 /               consumer-test-consumer-group-1
```  

`LOG-END-OFFSETS` 은 현재 파티션의 `offset` 수인 메시지 수를 의미한다. 
그리고 `CURRENT-OFFSET` 은 현재 해당 `Consumer Group` 에서 `Consumer` 가 소비한 `offset` 을 의미한다. 
지금 현재는 `aotu.offset.reset : latest` 로 돼 있으므로 `0` 값을 조회되는 것이다. 
`LAG` 해당 `Consumer` 가 `Partition` 의 맨끝 `offset` 에서 얼만큼 뒤쳐져 있는지를 의미한다. 
위 처럼 현재 소비한 `offset` 이 없는 경우에도 `LAG` 은 발생하지 않은 것으로 나온다.   

위에서 살펴본 `Data Loss` 시나리오를 `CURRENT-OFFSET` 이 설정되지 않은 상태에서 가정한다면 새로운 메시지가 `Partition` 에 추가돼서 `Consumer` 가 처리를 수행하지만 에러가 발생했을 때, 
`LOG-END-OFFSET` 은 3으로 이동하고 `CURRENT-OFFSET` 은 설정되지 않은 상태 그대로 유지된다. 
그 이후 새로운 `Consumer` 가 동일한 `Partition` 을 할당 받더라도 `auto.offset.reset: latest` 옵션 값으로 실패한 새 메시지는 여전히 소비되지 않고, 
그 다음 메시지를 대기하게 된다.  

`Conumser Group` 의 `CURRENT-OFFSET` 은 각 `Consumer` 가 최초 메시지 하나를 성공적으로 소비 완료하면 실제 값이 설정된다. 
만약 `CURRENT-OFFSET` 이 정상적으로 설정된 `Consumer` 가 재시작된다면 `auto.offset.reset` 의 설정 값과는 관게 없이 소비하던 다음 `offset` 을 이어서 소비하게 된다. 
위 출력 결과에서 `Consumer` 가 1개의 메시지를 소비하면 `CURRENT-OFFSET` 은 1이 되고 `LAG` 은 1이 될 것이다. 
그리고 이어서 다음 메시지까지 소비하면 `CURRENT-OFFSET` 은 2가 되고 `LAG` 은 0이 된다.  

```bash
$ kafka-consumer-groups.sh --botstrap-server localhost:9092 --group test-consumer-group --describe

GROUP               TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                         HOST            CLIENT-ID
test-consumer-group test-topic      0          1               2               1               consumer-test-consumer-group-1-925d2211-0d76-404b-bd8e-adfd954abfd5 /               consumer-test-consumer-group-1

$ kafka-consumer-groups.sh --botstrap-server localhost:9092 --group test-consumer-group --describe

GROUP               TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                         HOST            CLIENT-ID
test-consumer-group test-topic      0          2               2               0               consumer-test-consumer-group-1-925d2211-0d76-404b-bd8e-adfd954abfd5 /               consumer-test-consumer-group-1
```  

`CURRENT-OFFSET` 이 1인 상황에서 2번째 메시지를 소비하던 중 해당 `Consumer` 가 비정성 종료되고, 
새로운 `Consumer` 가 동일한 `Partition` 을 할당 받는다면 `CURRENT-OFFSET` 이 있이므로 2번째 메시지는 다시 소비되어 진다.  


---  
## Reference
[Kafka Consumer Auto Offset Reset](https://www.lydtechconsulting.com/blog-kafka-auto-offset-reset.html)  
