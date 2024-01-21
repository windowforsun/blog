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


## Rebalancing Protocol
`Kafka 2.4` 이전 버전의 `Kafka` 에서는 기존에 `Eager Rebalancing Protocol` 을 기본 
`Rebalancing Protocol` 로 사용했다. 
`Eager Rebalancing` 의 단점을 좀 더 보완한 `Incremental Rebalancing` 이 `Kafka 2.4` 버전 부터 나오게 되었고, 
기본은 동일하게 `Eager Rebalancing` 이지만 필요에 따라 `Incremental Rebalancing` 을 설정해 사용할 수 있다. 
본 포스팅에서는 두가지 `Reblaancing Protocol` 의 동작 방식와 어떠한 차이점들이 있는지 확인해 본다.  


> #### Group Coordinator
> - `Kafka Cluster` 의 브로커 중 하나이다. 
> - 특정 `Consumer` 에 대한 메타 데이터를 관리하고, `Rebalancing` 프로세스를 조정한다. 
> - `Consumer Group` 생성 또는 그룹 가입때 `Group Coordinator` 와 연결을 맺게된다. 
> - `Group Coordinator` 는 `Consumer` 가입/탈퇴를 처리하고, 필요시 `Rebalancing` 을 시작한다. 
> 
> #### Group Leader
> - `Consumer Group` 내 특정 `Consumer` 가 `Group Leader` 로 선정 될 수 있다. 
> - 해당 `Consumer` 는 그룹 내 다른 `Consumer` 를 대표해서 `Group Coordinator` 와 통신을 담당한다. 
> - `Rebalancing` 과정에서 `Group Leader` 는 새로운 `Partition Assignment` 계획을 수립하고 이를 다른 `Consumer` 들에게 전파한다.
