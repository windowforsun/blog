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
  - Kubernetes
toc: true
use_math: true
---  

## Consumer Lag
컨슈머 랙(`Consumer Lag`)이란 프로듀서가 데이터를 넣는 속도가 컨슈머가 가져가는 속도보다 빠른 경우, 
프로듀서가 넣은 데이터 오프셋과 컨슈머가 가져간 데이터의 오프셋 차이 또는 토픽의 가장 최신 오프셋과 컨슈머 오프셋의 차이를 의미한다.  

프로듀서가 데이터를 토픽내의 파티션에 추가하면 각 데이터에는 오프셋이라는 숫자가 증가하며 붙게 된다. 
그리고 컨슈머는 파티션의 데이터를 하나씩 읽어올 때, 데이터를 어디까지 읽었는지 표시하기 위해 자신이 마지막에 읽은 데이터의 오프셋을 저장해둔다. 
즉 `Consumer Lag` 이란 위서 설명한 것처럼 프로듀서는 빠르게 많은 데이터를 파티션에 넣어 오프셋값이 크게 증가 했지만, 
컨슈머는 느리게 읽고 있어서 마지막으로 읽은 오프셋의 값이 작고 읽어야 하는 데이터가 너무 많이 남아 있다는 의미이다. 
이는 다르게 말하면 데이터 지연이 발생한다 던가 컨슈머가 정상적으로 동작을 해주지 못하고 있는 상황이라고 추측 할 수 있다.  

프로듀서와 컨슈머의 오프셋은 파티션을 기준으로 한 값이기 때문에 
토픽 하나에 여러 파티션이 존재하는 상황이라면 `Lag` 은 여러개가 존재하 수 있다. 
예를 들어 파티션이 2개이고 컨슈머 그룹이 1개
토픽에 여러 파티션이 존재하는 경우 `Lag` 은 여러개 존재 할 수 있다. 
예를들어 파티션이 2개이고 하나의 컨슈머 그룹이 있는 상태라면 2개의 `Lag` 이 발생할 수 있다. 
그리고 이중에서 가장 높은 `Lag` 숫자를 갖는 것을 `records-lag-max` 이라고 한다.  


### Consumer Lag 증상
`Consumer Lag` 은 `Kafka` 를 운영할때 가장 중요한 지표중 하나라고 할 수 있다. 
실질적인 `Lag` 의 크기는 작거나 클 수 있는데 이를 통해 해당하는 토픽의 프로듀서, 컨슈머의 상태를 파악하고 분석하는데 중요한 정보가 될 수 있다.  

`Consumer Lag` 이 증가하고 있다면 아래와 같은 상황을 고려해 볼 수 있다.  
- 컨슈머의 성능에 비해 프로듀서가 순간적으로 많은 메시지를 보내는 경우
- 컨슈머의 `polling` 속도가 느려지는 경우, `polling` 후 메시지 처리 로직
- 네트워크 지연 이슈
- `Kafka broker` 내부 이슈

## Burrow 

[Burrow](https://github.com/linkedin/Burrow) 는 `Linkedin` 에서 만든 `Consumer Lag` 모니터링 오픈소스 툴이다. 
각 컨슈머는 특정 토픽에 대해 고유의 `groupId` 를 가지고 





---
## Reference
[Burrow - Kafka Consumer Lag Checking](https://github.com/linkedin/Burrow)  
[Consumer Lag Evaluation Rules](https://github.com/linkedin/Burrow/wiki/Consumer-Lag-Evaluation-Rules)  
[Setting Up Burrow & Burrow Dashboard For Kafka Lag Monitoring](https://www.lionbloggertech.com/setting-up-burrow-dashboard-for-kafka-lag-monitoring/)  
