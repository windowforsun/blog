--- 
layout: single
classes: wide
title: "[MongoDB 개념] "
header:
  overlay_image: /img/mongodb-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - MongoDB
tags:
  - MongoDB
  - Concept
  - MongoDB
  - Sharding
toc: true
use_math: true
---  


## MongoDB Sharding
`MongoDB` 는 `Database` 엔진 자체적으로 `Sharding` 아키텍쳐를 지원한다. 
`MongoDB` 의 `Sharding` 이 어떤 것이고 `Kubernetes` 를 사용해서 어떻게 구성할 수 있는지 알아보고자 한다.  

`Sharding` 은 여러 노드(머신, 컴퓨터)에 데이터를 나눠 저장하는 방법을 의미한다. 
`MongoDB` 는 앞서 언급한것과 같이 `Sharding` 을 지원하기 때문에 매우 큰 데이터를 저장하면서 높은 처리량의 아키텍쳐를 구성할 수 있다.  

데이터 크기가 매우 크고, 높은 처리량이 필요하다면 아래와 같은 요구사항이 발생한다. 
- 데이터의 크기가 매우 크고, 계속해서 커진다면 용량 문제가 발생할 수 있다. 
- 높은 처리량이 요구된다면 CPU, RAM 에 대한 부하가 생긴다. 
- 위 2가지는 모두 `DISK I/O` 성능과 대역폭에 의존하게 된다. 

나열한 문제점을 해결하면서 요구사항을 충족할 수 있는 방법은 `Vertical Scaling` 과 `Horizontal Scaling` 이 있다. 
결론 먼저 말하면 `MongoDB` 는 `Horizontal Scaling` 을 통해 `Sharding` 을 지원한다.  

### Vertical Scaling
`Vertical Scaling` 은 노드에 대해서 `Scale Up` 을 수행하는 방법을 의미한다. 
처리량이 부족하다면 `CPU` 와 `RAM` 을 더 높은 스펙으로 추가하거나 교체 할 것이고, 
저장공간이 부족하다면 용량을 계속해서 추가하는 것이다. 
한개 혹은 몇개의 노드를 두고 계속해서 노드 자체의 덩치를 키워나가는 방법으로 설명할 수 있다. 
하지만 `Vertical Scaling` 은 스케일링이 가능한 절대적인 최대 값이 존재하기 마련이다. 
여타 `Cloud Service` 를 사용하거나 물리 머신을 사용한다고 하더라도 단일 노드에 대해서 `Scale Up` 을 할 수 있는 최대 `Limit` 는 항상 존재하기 때문이다.  

### Horizontal Scaling
`Horizontal Scaling` 은 새로운 노드를 추가하는 방법으로 `Scale Out` 을 수행하는 방법을 의미한다. 
단일 노드의 스펙은 높지 않을 수 있지만, 이러한 여러 노드로 하나의 거대한 시스템을 만들어 높은 처리량과 매우 큰 용량을 제공할 수 있다. 
만약 필요하다면 노드를 추가해서 계속해서 전체 스펙을 키워나갈 수 있고, 불필요한 경우에는 특정 노드를 제거해서(`Migration` 필요) 스펙을 낮출 수도 있다. 
`Vertical Scaling` 에 비해 비용측면에서 이점이 있을 수 있고, 스펙을 올릴 수 있는 최대 값 또한 비교적 매우 크다라고(수천 대의 노드) 할 수 있다. (시스템마다 다르겠지만 추가 가능한 최대 노드의 수는 존재한다.)
하지만 유지보수, 복잡성에 대한 단점이 존재한다.  


## Sharded Cluster
`MongoDB` 에서 







그렇다면 먼저 `Sharding` 을 하지 않고, 단일 노드에 매우 큰 데이터를 저장하고 높은 처리량을 요구한다고 가정해보자. 
데이터가 매우 크고, 이후에도 지속적으로 데이터의 크기가 커진다면 해당 노드에 계속해서 





---
## Reference
[MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)  








