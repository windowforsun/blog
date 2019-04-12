--- 
layout: single
classes: wide
title: "Redis 의 Cluster 란"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 의 Cluster 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Cluster
---  
# Background
## 성능 향상 방법
### Scale Up
- 단일 머신(Machine)에 CPU, 디스크 등을 추가해서 성능을 향상 시킨다.

### Scale Out
- 적절한 성능의 머신(Machine)을 추가해서 전체적인 성능을 향상시키는 방법
	- Software 가 Scale Out 을 지원해야 가능하다.
	
![scale up and scale out]({{site.baseurl}}/img/server/improveperformance-scaleupscaleout.jpg)


## 대란 데이터 처리, 저장 방법
### Data Partitioning
- 대량의 데이터를 처리하기 위해 DBMS 안에서 분할하는 방식이다.
- 한대의 DBMS 에서 처리가 가능하다.
- 주로 하나의 큰 테이블을 여러개의 테이블로 분할하는데에 사용하는 방식이다.

![data partitioning]({{site.baseurl}}/img/redis/concept-cluster-datapartitioning.jpg)

- Scale Up -> 1 Machine -> 1 DBMS -> Data Partitioning

## Data Sharding
- 대량의 데이터를 처리하기 위해 여러 개의 DBMS 에 분할하는 기술이다.
- DBMS 안에서 데이터를 나누는 것이 아니라, DBMS 밖에서 데이터를 나누는 방식이다.
- Shard 수에 따라 여러 대의 DBMS 가 필요하다.

![data sharding]({{site.baseurl}}/img/redis/concept-cluster-datasharding.jpg)

- Scale Out -> n Machaines/VMs -> n DBMSs -> Data Sharding

## Network Topology
- Topology 의 종류
	
	![network topology]({{site.baseurl}}/img/redis/concept-cluster-topology.png)

### Single point of failure(SPOF) 단일 장애점이 있는 Topology
- Start
- Tree

#### Greemplum Database Architecture
- Active Master는 1개만 필요하고, 백업용 Standby Master 를 두고 Active Master 다운 시 Standby Master 가 Master 의 역할을 수행한다.
- Master 는 데이터가 어느 세그먼트 노드에 있는지와 세션(connection) 정보를 관리한다.
- 클라이언트는 항상 마스터를 통해서만 쿼리 수행이 가능하다.

![]()

#### HBase Architecture
- HBase 는 HDFS(Hadoop Distributed File System) 을 기반으로 분다.
- Master 는 Region 정보를 가지고 있고, HDFFS 의 name node 에 위치한다.
- 클라이언트는 Master 로 부터 Region 정보를 가져와서, 연결은 Master 를 거치지 않고 Region 서버에 직접한다.
- Region 정보가 복제된 여러개의 Master 를 둘 수 있다.
- HBase 는 단일 장애점(SPOF)을 가지는 구조는 아니지만, 소수의 Master 가 다운될 경우 HBase 를 사용할 수 없는 구조이다.
- Multiple Points of Failure(MPOF) 이다.

![]()

### Single point of failure 가 없는 Topology
- Mesh

#### Redis Cluster Architecture
- Clone 노드를 포함한 모든 노드가 서로를 확인하고 정보를 주고 받는다. -> Full Connected Messh Topology
- Greenplum 이나 HBase 와 같이 별도의 Master 노드를 두지 않는 구조이다.
- 모든 노드가 Cluster 구성 정보(슬롯 할당 정보)를 가지고 있다.
- 클라이언트는 어느 노드든지 접속해서 Cluster 구성 정보(슬롯 할당 정보) 를 가져와서 보유하고, 입력되는 키에 따라 해당 노드에 접속해서 처리한다.
- 일부 노드가 다운되어도 다른 노드에 영향을 주지 않는다.
- 총 노드 수의 과반수 이상의 노드가 다운되면 Redis Cluster 는 멈추게 된다.
- 데이터를 처리하는 Master 노드는 1개 이상의 복제 노드를 가질 수 있다.

![]()

## Key-Node 할당 방식
### Range Assignment
- Key 의 범위로 노드를 할당한다.
- Key 의 알파벳 첫 글자로 노드를 할당할 경우, 노드가 26개이면 A, B, C, D 순으로 노드를 할당하면 된다.
	- 영단어의 비율을 고려한다면 P = 11%, Y = 0.5% 로 각 노드의 데이터 크기가 매우 차이날 수 있다.
- Key 를 한국의 지역 순으로 할당한다면 각 도시의 인구 분포도에 따라 노드의 데이터 크기가 차이 날 수있다.
- List 할당 방식은 Range 할당 방식의 한 종류이다.

### Hash Assignment
- Key 에 Hash 함수를 적용해서 노드를 할당한다.
- 노드 개수와 무관하고 모든 노드에 일정하게 데이터가 할당 될 수 있다.
- Redis Cluster-Hash Sharding By Key



# Redis Cluster
## Redis Cluster 의 목표
- 1000 대의 노드 까지 확장할 수 있도록 설계되었다.
- 노드 추가, 삭제 시 Redis Cluster 전체를 중지할 필요 없고, 키 이동 시에만 해당 키에 대해서만 잠시 멈출 수 있다.

## Redis Cluster Key-Node 할당 방법
- Redis 서버가 10대 있을 경우 입력된 키를 한 서버로 들어가게 하기 위해서, 입력된 키에 Hash function 을 적용해 1~100 까지의 숫자를 만든다.
	- 1~10은 1번 서버에 넣고, 11~20은 2번 서버에 .. 넣는 방식으로 할당 한다.
	- 1~100 까지의 숫자를 Slot(슬롯) 이라고 한다.
	- 모든 서버는 자신이 보유하고 있는 Slot 정보를 가지고 있으며, cluster nodes 명령으로 각 서버에 할당된 Slot 정보를 얻을 수 있다.
- Redis Cluster 는 16384개의 Slot 을 사용한다.
	- Slot 번호는 0~16383이고, 이 Slot 을 노드에 할당한다.
	- Redis 노드가 3개 일 경우
		- 1번 노드 0-5460
		- 2번 노드 5461-10922
		- 3번 노드 10923-16383
	- Slot 을 할당하는 것은 Redis Cluster 관리자가 하게 된다.
- Hash function 은 CRC16 function 을 사용한다.

```
HASH_SLOT = CRC16(key) mode 16384
```  

## Redis Server 의 역할
- Redis 서버는 Master 또는 Slave 가 될 수 있다.
- 한 Master 가 다운되면 다른 Master 들이 장애조치(failover) 를 진행한다.
- Redis Cluster 에서는 별도의 센티널이 필요하지 않다.

## Redis Cluster 제한 사항
- 기본적으로 멀티 키 명령은 수행할 수 없다.
	- MSET key1 value1 key2 value2
	- SUNION key1 key2
	- SORT
- hash tag 를 사용하여 위 명렁어의 key 를 다음과 같이 사용하면 가능하다.
    - {user001}.following 과 {user001}.followers 는 같이 슬롯에 저장된다.
- Data Merge(멀티 키 명령)를 허용하지 않는 이유
	- 노드간 데이터(key, value) 를 전송하는 것은 병목 현상이 발생할 수 있어 성능이 우선인 Redis 에게는 치명적이다.
	- merge 가 필요한데 key 에 hash tag 를 사용하지 않았다면 어려운 상황이 발생할 수 있다.
- Cluster 모드에서는 DB 0번만 사용할 수 있다.
- Redis 3.0 버전 이상 부터 Cluster 가 가능하다.

---
## Reference
[Redis CLUSTER Introduction](http://redisgate.kr/redis/cluster/cluster_introduction.php)  