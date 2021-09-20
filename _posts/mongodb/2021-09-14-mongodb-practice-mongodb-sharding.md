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
결론 먼저 말하면 `MongoDB` 는 `Horizontal Scaling` 을 통해 `Sharding` 을 지원하므로, 
알아 볼 것은 `MongoDB Sharded Cluster` 에 대한 내용이다.  

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
`MongoDB` 의 [Sharding Cluaster](https://docs.mongodb.com/manual/reference/glossary/#std-term-sharded-cluster)
는 아래와 같은 3가지 컴포넌트로 구성된다.  

컴포넌트|설명
---|---
shard|각 `shard` 는 하나의 컬렉션에 대한 부분집합으로 구성된다. 즉 하나의 컬렉션의 데이터가 `N` 개의 `shard` 에 분리돼서 저장된다. 그리고 `shard` 의 구성은 `Replica Set` 으로 다시 구성될 수 있다.
mongos|`mongos` 는 클라이언트 애플리케이션과 `sharded cluster` 간의 인터페이스를 제공하는 쿼리 라우터이다. 즉 `shard cluster` 사용을 위해서는 `mongos` 를 통해 쿼리를 수행해야 한다.
config server|`config server` 는 전체 `MongoDB Sharded Cluster` 를 구성하는 서버의 메타데이터와 설정 정보를 저장한다.

위 3가지 컴포넌트의 구성을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-1.svg)

`MongoDB` 의 `Sharding` 은 `Collection` 레벨에서 수행된다. 
이는 하나의 `Collection` 을 이루는 데이터가 `Sharded Cluaster` 를 구성하는 
각 샤드에 적절하게 분배돼서 저장되는 것을 의미한다.  


## Shard Keys
`Shard Keys` 는 `Sharded Cluster` 에서 특정 데이터를 어느 샤드에 저장할지를 결정하는 중요한 요소이다. 
이는 하나 혹은 그 이상의 `Document` 의 필드로 이뤄질 수 있다. 
더 자세한 내용은 [여기](https://docs.mongodb.com/manual/core/sharding-shard-key/)
에서 확인 할 수 있다.  

버전|특이사항
---|---
4.4 및 이후|`Shard key` 에 존재하지 않는 필드를 설정할 수 있다. 더 자세한 내용은 [여기](https://docs.mongodb.com/manual/core/sharding-shard-key/#std-label-shard-key-missing)에서 확인 가능하다. 
4.2 및 이전|`Shard key` 로 설정하는 필드는 반드시 값이 존재하는 필드로 구성돼야 한다. 

`Shard key` 를 설정하는 방법은 [여기](https://docs.mongodb.com/manual/core/sharding-shard-a-collection/#std-label-sharding-shard-key-creation)
에서 확인 할 수 있다. 
버전마다 아래와 특징을 가지고 있다. 

버전|특이사항
---|---
5.0 및 이후|데이터 분포 및 성능에 따라 샤드 키 구성후 변경이 가능 하다. [자세히](https://docs.mongodb.com/manual/core/sharding-reshard-a-collection/#std-label-sharding-resharding)
4.4 및 이후|구성된 샤드 키에 `suffix` 및 필드를 새로 추가할 수 있다. [자세히](https://docs.mongodb.com/manual/core/sharding-refine-a-shard-key/#std-label-shard-key-refine)
4.2 및 이전|한 번 구성한 샤드 키의 변경은 불가능 하다. 

앞서 언급한 것처럼 `Shard key` 는 `Sharded Cluster` 의 분배를 결정하는 값인데 버전마다 아래와 같은 특이 사항이 있다. 

버전|특이사항
---|---
4.2 및 이후|샤드 키로 지정된 값을 `_id` 와 같은 변경 불가능한 필드로 지정되지 않은 이상 변경할 수 있다. [자세히](https://docs.mongodb.com/manual/core/sharding-change-shard-key-value/#std-label-update-shard-key)
4.0 및 이전|샤드 키로 지정된 필드는 변경 불가능하다. 


### Shard Key Index
이미 데이터가 존재하는 `Collection` 을 `Sharding` 하기 위해서는 `Shard Key` 로 시작하는 인덱스가 존재해야 한다. 
빈 `Collection` 을 `Sharding` 할때 적절한 `Shard Key Index` 가 없는 경우 `MongoDB` 는 자동으로 명시적인 인덱스를 생성한다. 
`Shard Key Index` 와 관련된 더 자세한 내용은 [여기](https://docs.mongodb.com/manual/core/sharding-shard-key/#std-label-sharding-shard-key-indexes)
에서 확인 가능하다. 

## Chunks 
`MongoDB` 의 파티션은 여러개의 [청크](https://docs.mongodb.com/manual/reference/glossary/#std-term-chunk)
로 구성된다. 
그리고 각 청크는 특정 [샤드 키](https://docs.mongodb.com/manual/reference/glossary/#std-term-shard-key)
의 범위에 대한 데이터를 포함한다. 


## Balancer and Even Chunk Distribution
`Sharded Cluster` 를 구성하는 모든 샤드에 청크를 균등하게 분산하기 위해 [Balancer](https://docs.mongodb.com/manual/core/sharding-balancer-administration/)
가 백그라운드에서 실행되어 청크를 [샤드](https://docs.mongodb.com/manual/reference/glossary/#std-term-shard)
간에 마이그레이션을 수행한다. 

## Advantage of Sharding
### Read/Write
`Shared Cluster` 를 구성한 경우 `Read/Write` 에 대한 작업을 구성된 샤드에 분산시켜 각 샤드가 클러스터의 작업 하나를 나눠 처리하기 때문에, 
전체적인 가용량이 증가하게 된다. 
그리고 이후에도 샤드를 추가해서 클러스터 전체에 대해서 `Horizontal` 확장이 가능하다.  

그리고 샤키 키에 대해서 접두사가 포함되는 쿼리는 `mongos` 가 특정 샤드 또는 샤드 세트를 대상으로 지정할 수 있다. 
이러한 방법을 사용하면 클러스터 전체 샤드에 대해서 쿼리를 수행하는 것보다 더 효율적이다.  

`4.4` 버전 이후 부터는 [hedged reads](https://docs.mongodb.com/manual/core/sharded-cluster-query-router/#std-label-mongos-hedged-reads)
라는 기능을 통해 읽기 동작을 `Primary` 가 아닌 `Secondary` 를 통해 수행하도록 할 수 있어, 
읽기 동작에 대해서 레이턴시를 최소화 할 수 있다.  

### Storage Capacity
`Sharded Cluster` 는 여러 샤드의 집합이고 하나의 `Collection` 을 구성하는 데이터를 각 샤드에 분산 시켜 저장하기 때문에, 
데이터가 증가한다면 샤드를 추가하는 방법으로 전체 스토리지 용량을 증가 시킬 수 있다.  


### High Availability
`MongoDB` 의 `Shared Cluster` 는 여러 샤드에 라우팅 역할을 해주는 `mongos` 와 실제 데이터가 저장되는 `shard` 로 분리 돼있다. 
이러한 특징으로 `MongoDB` 에 대해서 고가용성을 보장할 수 있다.  
각 샤드가 `ReplicaSet` 구성으로 `Primary`, `Secondary` 로 구성된 경우에는 특정 노드의 장애가 발생 하더라도, 
하나의 `Secondary` 만 생존해 있다면 읽기/쓰기 연산을 계속해서 수행할 수 있다. 
그리고 특정 샤드를 구성하는 노드 전체에 장애가 발생하더라도, 해당 샤드 데이터의 읽기/쓰기 연산은 불가능하지만 
그 외 샤드에 대한 읽기/쓰기 동작은 가능하므로 특정 노드의 장애가 `MongoDB` 전체 장애를 발생시키지 않는다.  


### Considerations Before Sharding



### Shard Key Strategy
.. 아래 쪽에 그림 있는거랑 같이 설명 ..



그렇다면 먼저 `Sharding` 을 하지 않고, 단일 노드에 매우 큰 데이터를 저장하고 높은 처리량을 요구한다고 가정해보자. 
데이터가 매우 크고, 이후에도 지속적으로 데이터의 크기가 커진다면 해당 노드에 계속해서 





---
## Reference
[MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)  








