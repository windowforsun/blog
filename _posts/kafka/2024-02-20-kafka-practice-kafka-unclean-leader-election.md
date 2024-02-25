--- 
layout: single
classes: wide
title: "[Kafka] Kafka Unclean Leader Election"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Cluster 에서 약간의 메시지 손실로 내구성을 줄이고 가용성을 높이는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Cluster
    - Kafka Broker
    - Availability
    - Durability
toc: true
use_math: true
---  

## Unclean Leader Election
`unclean.leader.election.enable` 은 `Kafka Cluster` 의 구성원인 `Leader` 와 `Follower` 가 있을 떄, 
`Leader` 의 복제본이 아직 모두 동기화되지 않은 `Follower` 가 오류 시나리오에서 `Leader` 가 될 수 있는지를 결정하는 옵션이다. 
만약 해당 옵션을 활성화 시킨다면 `Leader` 로 선정된 `Follower` 에는 모든 복제본의 데이터가 존재하지 않을 수 있으므로, 
메시지의 손실이 발생 할 수 있다.  

해당 옵션을 설정여부에 따라 `Kafka Cluster` 가 보장하는 범위나 장단점에 확실한 차이가 있으므로 
옵셜 설정에 따른 결과를 이해하는 것이 중요하다.  
