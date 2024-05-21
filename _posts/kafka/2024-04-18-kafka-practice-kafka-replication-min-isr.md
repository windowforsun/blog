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


## Kafka Replication
`Kafka` 는 분산 메시징 시스템으로 고가용성과 내구성을 제공한다. 
이러한 `Kafka` 의 특성은 `Broker Cluster` 간의 `Replication` 이 있기 때문에 보장될 수 있다. 
특정 `Broker` 가 실패하더라도 사용 중인 `Topic Partition` 의 데이터는 유실되지 않고, 
복제된 `Broker` 의 `Topic Partition` 의 데이터로 지속적인 서비스가 가능하다. 
`Replication` 동작은 말그대로 복제이기 때문에 약간의 지연시간이 증가할 수 있다. 
하지만 복제 수준 설정이 가능하기 때문에 서비스 니즈에 맞는 설정해주어야 한다. 
이러한 설정은 `Producer` 가 `acks` 를 수신할 최소 `Replication` 개수인 `Min In-Sync Replica(Min ISR)` 이라고 한다.  


### Min In-Sync Replica
`Kafka` 가 `Producer` 로 부터 메시지를 받고 `Topic Partition` 에 메시지를 작성 할떄, 
`Topic Partition` 의 `Leader` 에게 메시지를 작성한다. 
여기서 `Leader Replica` `Topic Partition` 의 `Replica` 중 `ISR` 에 의해 `Leader` 로 선출된 `Replica` 를 의미한다. 
`Leader Replica` 에 작성된 메시지는 이후 `Follower Replica` 들에게 복제된다. 
이런 `Topic Partition` 의 복제 수는 `replication.factor` 에 의해 정해진다. 
만약 `replication.factor` 이 3이라면 데이터가 `Leader Replica` 에 써진 뒤 2개의 `Follower Replica` 에 데이터가 복제되어, 
총 3개의 `Replica` 에 데이터가 저장됨을 의미한다.  

`Replica` 에 데이터가 복제 될 때, 
`min.insync.replicas` 설정 만큼 데이터가 복제 된 후 `Producer` 가 `acks` 를 받을 수 있다. 
즉 이는 전송한 데이터가 성공적으로 작성을 보장하는 최소한의 `Replicas` 수를 결정한다. 
해당 값이 `all` 로 설정되면 모든 `Replica` 에 복제된 후에야 `Producer` 는 `acks` 를 수신해 이후 메시지 처리가 가능하다. 

### ISR
`ISR(In-Sync Replicas)` 는 `Partition Replica` 중 `Leader Replica` 와 동기화 된 `Replica` 를 의미한다. 
`Follower Replica` 들은 동기화를 위해 `Leader Replica` 에게 최신 데이터를 얻고, 
각 `Partition` 의 `Leader Replica` 는 `Follower Replica` 의 `Lag` 상태를 추적한다. 
그리고 `Leader Replica` 가 `replica.lag.time.max.ms` 내 `Follower Replica` 에게 최신 데이터 `Fetch` 요청을 받지 못하거나, 
해딩 시간내에 `Leader Replica` 의 가장 최신 데이터까지 동기화를 못한 경우에는 동기화 되지 않은 걸로 간주해 `ISR` 에서 제거된다.  
