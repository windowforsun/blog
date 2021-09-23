--- 
layout: single
classes: wide
title: "[MongoDB 개념] MongoDB Sharding (Shared Cluster)"
header:
  overlay_image: /img/mongodb-bg.png
excerpt: 'MongoDB 에서 Sharding 과 Shared Cluster 의 특징과 주의할 점에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - MongoDB
tags:
  - MongoDB
  - Concept
  - MongoDB
  - Sharding
  - Shared Cluster
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


## Considerations Before Sharding(주의 사항)
`Sharded Cluster` 는 단일 샤드만 사용하는 환경보다는 더욱 큰 용량과 많은 요청을 처리할 수 있는 구조인것은 분명하다. 
하지만 추가 인프라적인 요구사항과 그에 따른 복잡성이 수반되기 때문에, 
신중한 계획에 따른 수행 및 지속적인 유지보수를 필요로한다.  

그리고 하나의 `Collection` 이 샤딩되면, 
`MongoDB` 에서는 샤딩된 컬렉션을 다시 해제하는 방법은 지원하지 않는다.  

`Shard Key` 관련해서는 키의 분포가 좋지 못할때 `reshard` 동작을 수행할 수 있지만, 
이는 확장성과 성능 문제와 관련있으므로 초기부터 키 선택을 신중하게 고려해야 한다. 
관련해서 `Shard Key` 선택에 대한 문서는 [여기](https://docs.mongodb.com/manual/core/sharding-choose-a-shard-key/#std-label-sharding-shard-key-selection)
에서 확인 가능하다.  

`Mongos` 에서 수행되는 쿼리의 `Shard Key` 가 접두사를 포함하지 않는 경우, 
`Mongos` 는 구성된 클러스터 전체에 쿼리 수행을 요청하게 된다. 
그러므로 통계/수집과 같이 오랜 시간 소요될 수 있는 쿼리에는 주의가 필요하다.  


## Sharded and Non-Sharded Collections

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-2.svg)

`MongoDB` 데이터베이스에는 샤딩된 컬렉션과 샤딩되지 않은 컬렉션이 함께 공존할 수 있다. 
샤드 컬렉션은 [partition](https://docs.mongodb.com/manual/reference/glossary/#std-term-data-partition)
단위로 분할되어 클러스터의 각 샤드에 분산되어 저장된다. 
그리고 샤드되지 않은 컬렉션은 [primary shard]()(기본 샤드)
에만 저장된다. 
모든 `MongoDB` 데이터베이스에는 기본 샤드가 기본적으로 존재한다.  


## Connection to a Sharded Cluster

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-3.svg)

하나의 컬렉션이 여러 조각으로 나눠진 `Sharded Cluaster` 에서 정상적인 데이터 연산을 수행하기 위해서는 라우터 역할을 하는 [mongos](https://docs.mongodb.com/manual/reference/glossary/#std-term-mongos)
에 연결해서 연산을 수행해야 한다. 
`mongos` 는 샤드 컬렉션, 샤드되지 않는 컬렉션 모두 데이터 연산을 수행할 수 있다. 
만약 데이터 연산을 위해 특정 단일 샤드에 연결한 경우 해당 샤드에 있는 데이터만 연산 대상이 되기 때문에 주의가 필요하다.  

기존 [mongod](https://docs.mongodb.com/manual/reference/program/mongod/#mongodb-binary-bin.mongod)
에 연결하는 방법과 동일하게 
[mongosh](https://docs.mongodb.com/mongodb-shell/#mongodb-binary-bin.mongosh), 
[MongoDB driver](https://docs.mongodb.com/drivers/?jump=docs)
를 사용해서 `mongos` 에 연결 할 수 있다.  

mongosh 또는 MongoDB 드라이버를 사용하여 mongod에 연결하는 것과 동일한 방식으로 mongos에 연결할 수 있습니다.

## Sharding Strategy
샤드 키의 선택은 `Shared Cluster` 의 성능, 효율, 확장성 등과 같이 많은 부분에 영향을 미치는 중요한 포인트이다. 
최상의 하드웨어 및 인프라 자원을 갖춘 상태라도 샤드키의 선택에 따라 병목현상이 발생할 수 있음을 주의 해야 한다. 
그리고 샤드 키의 선택은 `Sharding Strategy` 에 따라 그 효율이 달라질 수 있으므로 샤딩 전략에 알맞는 샤드 키 선택이 필요하다.  

`MongoDB` 는 `Sharded Cluster` 에 데이터를 분산 시킬 수 있는 두가지 샤딩 전략을 제공한다.  

### Hashed Sharding

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-4.svg)

`Hashed Sharding` 은 지정된 샤드 키 필드 값을 바탕으로 해시 계산된 해시값이 포함되는 `chunk` 에 할당된다. 
여기서 언급된 해시함수 계산은 `MongoDB` 에서 샤딩을 위해 자동으로 수행하는 동작이기 때문에 애플리케이션에서 별도로 해시관련 연산을 수행할 필요는 없다.  

해시 샤딩을 사용할 때 샤드 키의 값이 인접해 있더라도 해당 값들이 동일한 청크에 있을 가능성은 거의 없다. 
그러므로 해시 값을 바탕으로 데이터 분산은 샤드 키가 [monotonically](https://docs.mongodb.com/manual/core/sharding-choose-a-shard-key/#std-label-shard-key-monotonic)(단순하게)
변하는 데이터 세트에서도 더욱 고른 분포의 데이터 분산을 보여줄 수 있다.  

해시 함수를 사용하게 되면 특정 데이터 셋이 있을때 해당 데이터가 하나의 샤드에만 포함될 확률은 현저하게 낮다는 것을 의미한다. 
그러므로 하나의 데이터 세트 연산 동작을 위해서는 전체 클러스터에 대해서 [broadcast operation](https://docs.mongodb.com/manual/core/sharded-cluster-query-router/#std-label-sharding-mongos-broadcast)
이 불가피 하다는 단점이 있다.  

`Hashed Sharding` 관련 더 자세한 내용은 [여기](https://docs.mongodb.com/manual/core/hashed-sharding/)
에서 확인 가능하다.  

### Ranged Sharding

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-5.svg)

`Ranged Sharding` 은 샤드 키의 값 기준으로 데이터의 범위를 나누는 것을 의미한다. 
그리고 샤드 키는 범위에 해당하는 `chunk` 에 할당된다.  

범위 샤딩을 사용할때 샤드 키의 값이 인접해 있다면 해당 값들은 동일한 청크에 있을 가능성이 매우 높다. 
이러한 특징으로 연산이 필요한 데이터가 인접해 있을 경우 특정 샤드에만 요청해서 작업을 수행하는 방법을 사용할 수 있다.  

위와 같은 특징으로 `Ranged Sharding` 의 효율은 샤드 키 선택에 좌우 된다. 
잘 고려되지 않는 샤드 키는 고르지 않은 데이터 분포를 가져올 수 있고, 
이는 샤딩의 장점 일부가 상실되거나 성능적으로 병목 지점이 될 수 있다. 
샤드 키 선택 관련해서는 [여기](https://docs.mongodb.com/manual/core/ranged-sharding/#std-label-sharding-ranged-shard-key)
에서 더 자세한 내용을 확인 할 수 있다.  

`Ranged Sharding` 과 관련해서 더 자세한 내용은 [여기](https://docs.mongodb.com/manual/core/ranged-sharding/)
에서 확인 가능하다.  

## Zones in Sharded Clusters

![그림 1]({{site.baseurl}}/img/mongodb/concept-mongodb-sharding-6.svg)

[Zones](https://docs.mongodb.com/manual/reference/glossary/#std-term-zone) 
은 여러 데이터 센터로 구성된 `Shared Cluster` 에서 데이터의 인접성을 개선하는데 도움이 될 수 있다. 
즉 하나의 `Shared Cluster` 에서 샤드 키를 기반으로 샤드에 대해서 사용자가 정의한 `Zone`을 만들 수 있다. (사용자 정의 `Sub Cluster` 라고 할 수 있음)
각 `Zone` 은 클러스터에서 하나 이상의 샤드를 포함하게 된다. 
그리고 각 샤드도 하나 이상의 `Zone`에 포함될 수 있는 `N:N` 의 관계를 갖는다.  

그리고 균형이 잡힌 `Sharded Cluster` 에서 `MongoDB` 는 `Zone` 에 포함된 샤드에 대해서만 `chunk` 마이그레이션을 수행한다. 
각 `Zone` 은 하나 이상의 샤드 키 값 범위가 있어야 한다. 
`Zone` 의 샤드 키 범위는 하한 범위는 포함하지만 상한 범위는 제외한다. (`start <= zone range < end`)  

새로운 `Zone` 을 생성 할때는 샤드 키에 포함되는 필드를 사용해서 범위를 정의해야 한다. 
복합 샤드 키를 사용하는 경우라면 `Zone` 범위에는 샤드 키의 범두사가 포함되야 한다. 
`Zone` 범위 정의 관련 더 자세한 내요은 [여기](https://docs.mongodb.com/manual/core/zone-sharding/#std-label-zone-sharding-shard-key)
에서 확인 가능하다.  

## Collections in Sharding

## Change Streams
`MongoDB 3.6` 부터 `Replica Set` , `Shared Cluster` 에 대해서 [Change Stream](https://docs.mongodb.com/manual/changeStreams/)
을 사용할 수 있다. 
`Change Streams` 을 사용하면 애플리케이션에서 `oplog` 를 `tailing` 해서 변경사항을 감지하는 복잡하고 안정적이지 않는 방법 대신, 
실시간으로 데이터의 변경사항에 액세스 할 수 있다. 
애플리케이션은 `Change Stream` 을 사용해서 데이터 수집, 컬렉션의 데이터 변경사항 구독 등의 동작을 수행할 수 있다.  

## Transactions
`MongoDB 4.2` 부터 [distributed transactions](https://docs.mongodb.com/manual/core/transactions/)
이 도입 돼어 `Shared Cluster` 에서도 `Multi-Document` 에 대해 트랙잭션을 사용할 수 있다.  

트랜잭션이 커밋되기 전까지는 트랜잭션에서 수행된 데이터 변경은 트랜잭션 외부에서 변경사항을 확인 할 수 없다. 
트랜잭션이 실제로 커밋된 이후에 다른 트랜잭션에서 실제 변경사항이 반영된 결과를 조회할 수 있다는 의미이다.  

트랜잭션이 여러 샤드에 반영(커밋)될 때 모든 외부의 다른 읽기 작업들이 해당 트랜잭션이 모든 샤드에 커밋 될때까지 기다릴 필요는 없다. 
만약 트랜잭션이 A, B 샤드에 2라는 값으로 커밋될 때 A샤드만 커밋이 완료된 상태라면, B 샤드에서 읽기 동작을 수행하는 외부 읽기 동작에서는 여전이 1이 조회 될 수 있다.  

트랜잭션 관련 더 자세한 링크는 아래와 같다.  
- [Production Considerations](https://docs.mongodb.com/manual/core/transactions-production-consideration/)
- [Production Considerations (Sharded Clusters)](https://docs.mongodb.com/manual/core/transactions-sharded-clusters/)



---
## Reference
[MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)  








