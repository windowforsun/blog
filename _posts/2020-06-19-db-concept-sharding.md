--- 
layout: single
classes: wide
title: ""
header:
  overlay_image: /img/db-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - DB
tags:
  - DB
  - Database
  - Sharding
toc: true
use_math: true
---  

## DB Sharding
- `Sharding` 이란 단일 논리적인 데이터 집합을 여러 데이터베이스에 저장하는 것을 의미한다.
- 다수개의 시스템에 데이터를 분산시킴으로써 데이터베이스 시스템은 하나의 큰 클러스터를 구성하게 되고,
이를 통해 더 크고 많은 데이터 집합을 저장하고 더 많은 요청을 처리할 수 있게 된다.
- 단일 데이터베이스에 저장하기에 데이터 집합이 너무 크다면 `Sharding` 은 하나의 해결책이 될 수 있다.
- `Sharding` 에는 다양한 전략들이 존재하고 이를 통해 필요에 따라 시스템을 추가할 수 있다.
- 데이터 및 트래픽 증가에 따라 데이터베이스의 구성을 유연하게 대응할 수 있다.
- `Sharding` 은 하나의 테이블을 수평 분할(`Horizontal Partitioning`) 해서 여러 개의 작은 단위로 나눠 물리적으로 다른 위치에 분산해서 저장하는 것을 의미한다.
- 여기서 간단한 `Horizontal Partitioning` 과 `Vertical Partitioning` 의 차이는 아래와 같다.
	
	
	![그림 1]({{site.baseurl}}/img/db/concept-sharding-1.png)


## Sharding 의 장단점
### 장점
- `High Availability` : 샤딩은 운영 중단의  영향을 완화시킬 수 있기 때문에, 애플리케이션의 안전성을 높이는데 도움이 될 수 있다. 
만약 샤딩이 돼있지 않다면 사용중인 데이터베이스의 중단은 서비스 전체 애플리케이션을 사용할 수 없게 될 수 있다.
하지만 샤딩이 된경우라면, 일부 노드에 포함되는 데이터베이스의 중단은 해당되는 사용자 혹은 일부 서비스에만 영향이 미치게 된다.
- `Faster queries response` : 샤딩은 하나의 큰 테이블을 하나 이상의 테이블로 나누기 때문에 쿼리 응답 시간을 단축하는데 도움이 될 수 있다. 
모놀리식 데이터베이스일 경우 쿼리를 수행하기 위해서 방대한 `row` 의 모든 행을 검색해야 할 수 있고, 이는 쿼리 응답시간에 큰 영향을 미친다. 
하지만 한 테이블을 어러 개의 테이블로 분할할 경우, 쿼리의 행 수가 줄어들고 쿼리 응답시간 또한 줄어들게 된다. 
- `More write bandwidth` : 샤딩은 수평적(`Horizontal`) 확장을 통해 부하를 분산시키고, 더 많은 트래픽과 더 빠른 처리를 가능하게 한다. 
이는 단일 데이터베이스에서 처리하던 트래픽이 여러 노드의 데이터베이스로 분산되기 때문에, 그 만큼 동시에 처리할 수 있는 양은 증가할 수 있다.
- `Scaling out` : 샤딩은 수평적 확장(`Horizontal scaling`)을 통해 부하를 분산시키고, 더 많은 트래픽과 더 빠른 처리를 가능하게 된다. 
확장을 위해서 시스템을 추가하는 방식으로 진행하기 때문에, 수직 스케일업(`Vertical scaling up`) 과는 대조된다. 
이는 램 혹은 CPU 를 계속 추가하는 방식으로 모놀리식 시스템에 하드웨어를 업그레이드 하는 방식이다.

### 단점
- `Add complexity in the system` : 샤딩을 적용하는 과정에서 올바른 샤딩 아키텍쳐 구현의 복잡성과 기존 애플리케이션의 복잡성이 올라가는 부분은 불가피한 부분이다.
잘못된 샤딩 프로세스로 인해 데이터가 손실되거나 테이블이 손상되는 심각한 위험이 발생할 수 있다. 
하지만 옳바른 샤딩 프로세스로 진행된다 하더라도 팀 워크플로우에 영향을 끼칠 수 있다. 
- `Rebalancing data` : 샤딩이 된 아키텍쳐에서 발생할 수 있는 문제점은 각 샤드간 밸런드에 대한 문제이다. 
한 노드의 샤드에 데이터가 몰릴 경우 해당 샤드를 `핫스팟` 이라고 한다. 
이런 `핫스팟`이 누적될 경우 전체 애플리케이션에 영향을 끼칠 수 있고, 결국에는 균일한 분배를 위해 데이터베이스를 다시 샤딩하고 복구하는 큰 작업을 치뤄야 한다.
- `Joining data from multiple shards` : 샤딩 아키텍쳐로 구성된 상태에서 여러개의 샤드로 분리된 테이블에 대한 조인 작업은 어려울 수 있다. 
일반적으로 테이블에 대한 조인은 쿼리를 통해 데이터베이스에서 수행하도록 하지만, 
다른 노드에 위치하는 샤드와의 테이블 조인은 해당되는 샤드에 쿼리요청을 수행한 결과를 가지고 합치는 동작이 필요하다. 
- `No Native Support` : 샤딩에 대한 동작은 모든 데이터베이스 엔진에서 기본적으로 지원하지 않는다. 
특정 범위의 조직에서 샤딩에 대한 자체적인 방식과 룰을 정립하고 구현하는 것이 필요하게 된다. 
그로인해 샤딩으로 인핸 트러블 슈팅이 어려울 수 있고, 관련 문서나 팁에 대한 레퍼런스 또한 어려울 수 있다.


## Sharding Architecture
- 샤딩에서 `Shard`, `Partition Key`  는 데이터가 분류되는 방법을 결정하는 요소이다.
- `Partition Key` 를 통해 해당하는 데이터베이스로 작업을 라우팅하고 데이터를 효율적으로 검색하고 수정할 수 있다.
- 동일한 `Partition Key` 를 가진 항목이 동일한 노드에 저장되고, 논리적인 `Shard` 는 동일한 `Partition Key` 를 공유하는 데이터 셋을 의미한다.
- 하나의 데이터베이스 노드는 `physical shard` 라고 도하며, 하나 이상의 논리적 `Shard` 를 포함할 수 있다.

### Algorithm Sharding
- 정해진 특정 알고리즘을 통해 샤드를 동적으로 분류하는 방법이다.
- `Key Base Sharding` 혹은 `Hash Sharding` 이라고 불리기도 한다.
- 알고리즘을 샤딩은 데이터베이스 클라이언트의 도움없이 애플리케이션 단에서 주어진 데이터가 해당하는 샤드를 결정할 수 있다.
- 알고리즘 샤딩은 샤딩 함수를 통해 데이터를 찾고, 적용하는 간단한 방법은 $hash(key) mod NUM_DB$ 와 같은 방식이다.
- 알고리즘 샤딩은 `payload` 크기나 공간 효율성은 고려하지 않고, 분류되는 데이터의 수에만 초점이 맞춰져 있다. 
데이터를 균일하게 분배하기 위해서는 각 파티션의 크기가 비슷해야 한다. 
- 새로운 노드나 서버를 추가하려고 할때 각 노드, 서버는 해당 해시값을 알아야 하고 새로운 해시값으로 다시 대응해서 마이그레이션을 해야하기 때문에 까다로울 수 있다.
- 대표적으로 `Memcached` 는 애플리케이션 레벨에서 특정 로직을 통해 클러스터로 묶어진 각 노드에 키를 분배해 관리한다.

### Dynamic Sharding
- 동적 샤딩은 외부 로케이터 서비스를 통해 엔트리의 위치를 결정한다.
- `Range Based Sharding` 라고 불리기도 한다.
- 외부 로케이터 서비스는 데이터가 있는 샤드의 위치를 제공하고 핫스팟을 완화하기 위해 다량의 
































































































---
## Reference
[Understanding Database Sharding](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
[Building Scalable Databases: Pros and Cons of Various Database Sharding Schemes](http://www.25hoursaday.com/weblog/2009/01/16/BuildingScalableDatabasesProsAndConsOfVariousDatabaseShardingSchemes.aspx)
[How Data Sharding Works in a Distributed SQL Database](https://blog.yugabyte.com/how-data-sharding-works-in-a-distributed-sql-database/)
[How Sharding Works](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6)
[Database Sharding](https://medium.com/system-design-blog/database-sharding-69f3f4bd96db)