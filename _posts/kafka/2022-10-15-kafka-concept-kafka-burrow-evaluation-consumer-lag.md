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
  - Kafka
  - Concept
  - Kafka
  - Burrow
  - Consumer Lag
toc: true
use_math: true
---  

## Consumer Lag Evaluation Rules
`Burrow` 는 `Consumer Group` 의 상태를 사용하는 파티션의 오프셋에 대해 정해진 규칙을 바탕으로 판별한다. 
그룹이 사용하는 모든 파티션에 대해 평가를 수행하는 방식으로 `Consumer Group` 에 포함된 `Consumer` 전체가 정상 상태임을 판별 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/concept-burrow-evaluation-consumer-lag-2.png)

위 사진에서 파티션과 `Consumer Group` 의 관계를 살펴보자.
1개의 토픽에는 여러 파티션이 존재할 수 있고, `Consumer Group` 은 1개 이상의 파티션과 관계를 맺을 수 있는 구성을 띄고 있다.
그리고 `Consumer Group` 은 여러개의 `Consumer` 로 구성될 수 있다.
이러한 구성에서 우리가 `Consumer Lag` 를 평가할 때 중짐적으로 봐야하는 곳은 하나의 `Consumer Group` 이 하나의 토픽을 소비할때 얼마나 정상적으로 동작하고 있느냐라고 할 수 있다.  


### Evaluation Window
`Burrow` 는 `in-memory` 에 `Consumer Lag` 평가에 필요한 데이터들을 저장하는데,
이는 `Storage` 설정값 중 `intervals`(기본값 10) 에 따라 해당 평가 간격이 결정된다.
해당 값은 각 파티션에 대해 평가 용도로 저장할 오프셋 수를 의미한다.
그리고 `Consumer Lag` 평가에 대한 기준은 `Consumer Setting` 에 있는 `offset commit interval` 값과 결합돼 최종적으로 결정된다.
만약 `offset commit interval` 이 60초 이고, `intervals` 값이 10 이라면 10분 동안 `offset` 값들을 바탕으로 `Consumer Group` 상태를 평가한다.  

실제 평가는 슬라이딩 윈도우 방식으로 이뤄진다.
즉 새로운 `offset commit` 이 들어오게 되면 가장 오래된 `offset commit` 이 삭제되는 방식을 의미한다.

![그림 1]({{site.baseurl}}/img/kafka/concept-burrow-evaluation-consumer-lag-3.png)

### Evaluation Rules
`Burrow` 는 아래와 같은 규칙을 바탕으로 지정된 파티션에 대한 `Consumer Group` 의 상태를 평가한다.  
이러한 상태 값들은 모두 슬라이딩 윈도우에 존재하는 값들을 기반으로 산출된다.

consumer 상태|partition 상태|상황|설명
---|---|---|---
OK|OK|`lag` 가 0인 경우, `lag` 가 -1 인 경우(아직 해당 `Burrow` 시작등으로 `broker offset` 을 수신하지 못한 상태)|모두 정상 상태
ERROR|STALLED|`consumer offset` 은 변경되지 않고, `lag` 이 고정 혹은 증가하는 경우|`consumer` 가 커밋을 수행 중인 상태
WARNING|OK|`consumer offset` 은 증가하고, `lag` 은 고정 혹은 중가하는 경우|`consumer` 가 느려서 뒤쳐지고 있는 상태
ERROR|STOPPED|`consumer offset` 의 가장 최근시간과 현재시간의 차이가 가장 최근 `offset` 과 가장 오래된 `offset` 의 차이보다 클 경우, 만약 `commit offset` 과 `broker offset` 이 동일하다면 파티션은 정상|`consumer` 가 최근 아무런 `offset commit` 을 수행하지 않은 상태


### 평가 예시
아래 예제를 바탕으로 실제로 어떤 윈도우를 바탕으로 평가라 이뤄지는지 살펴본다. 
`W1`, `W2`, `W3`, .. 은 윈도우에 저장된 `offset` 정보를 의미한다. 
그리고 `T`, `T+60`, `T+120` 은 최초 `T` 시간에서 `60초` 만큼 지난 시간을 `T+60` 으로 간주한다.  

> `offset commit interval` 은 60초, `Strage` 의 `intervals` 는 10 인 설정 값으로 예제를 진행한다. 

#### Example 1) Consumer OK, Partition OK

| |W1|W2|W3|W4|W5|W6|W7|W8|W9|W10
|---|---|---|---|---|---|---|---|---|---|---
|Offset|10|20|30|40|50|60|70|80|90|100
|Lag|0|0|0|0|0|0|1|3|5|5
|Timestamp|T|T+60|T+120|T+180|T+240|T+300|T+360|T+420|T+480|T+540

위 윈도우는 `offset` 도 계속해서 증가하고 있고, `lag` 도 0으로 유지되다가 이후 시점 부터 약간 증가를 보이고 있지민, 
아직은 모두 `OK` 인 상태이다. 
하지만 이후 6번 `offset commit` 동안 `lag` 값이 5로 유지되거나 증가한다면 `consumer` 는 `WARNING` 상태로 변경될 수 있다.  


### Example 2) Consumer ERROR, Partition STALLED

|	|W1	|W2	|W3	|W4	|W5	|W6	|W7	|W8	|W9	|W10
|---|---|---|---|---|---|---|---|---|---|---
|Offset	|10	|10	|10	|10	|10	|10	|10	|10	|10	|10
|Lag|	1|	1|	1|	1|	1|	2|	2|	2|	3|	3
|Timestamp|	T|	T+60|	T+120|	T+180|	T+240|	T+300|	T+360|	T+420|	T+480|	T+540

위 윈도우는 `offset` 의 값이 증가 하고 있지 않기 때문에 `partition` 의 상태는 `STALLED` 이다. 
그리고 `lag` 또한 점차 증가하고 있기 때문에 `consumer` 의 상태는 `ERROR` 이다. 
`consumer` 는 실행 중이고 `offset commit` 을 수행하곤 있지만, 정상적으로 `parsition` 의 값을 사용중이지 못하므로 지연도 늘어나고 있는 상태이다.  


### Example 3) Consumer WARNING, Partition OK


|	|W1	|W2	|W3	|W4	|W5	|W6	|W7	|W8	|W9	|W10
|---|---|---|---|---|---|---|---|---|---|---
|Offset|	10|	20|	30|	40|	50|	60|	70|	80|	90|	100
|Lag|	1|	1|	1|	1|	1|	2|	2|	2|	3|	3
|Timestamp|	T|	T+60|	T+120|	T+180|	T+240|	T+300|	T+360|	T+420|	T+480|	T+540


### Example 4) 

---
## Reference
[Consumer Lag Evaluation Rules](https://github.com/linkedin/Burrow/wiki/Consumer-Lag-Evaluation-Rules)  
