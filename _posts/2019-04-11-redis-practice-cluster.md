--- 
layout: single
classes: wide
title: "Redis Cluster 설정하기"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 에서 Cluster 설정을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Cluster
---  


## 환경
- CentOS 6
- Redis 5.0

## Cluster 구성하기
- 7001, 7002, 7003 포트를 사용하여 3개로 Redis Cluster 를 구성한다.
- Redis Cluster 는 최소 3개가 필요하다.
- Redis 설정 파일을 위치를 파악한다.

```
[root@windowforsun ~]# ll /etc/redis.conf
-rw-r-----. 1 redis root 62229 Apr 11 04:28 /etc/redis.conf
```  

- redis.conf 파일을 아래와 같은 이름으로 복사한다.
	- redis-7001.conf
	- redis-7002.conf
	- redis-7003.conf

```
[root@windowforsun ~]# cp /etc/redis.conf /etc/redis-7001.conf
[root@windowforsun ~]# cp /etc/redis.conf /etc/redis-7002.conf
[root@windowforsun ~]# cp /etc/redis.conf /etc/redis-7003.conf
```  

- 7001~7003 설정 파일의 아래와 같은 부분들을 아래와 같이 변경한다.

```
port <port>
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
pidfile /var/run/redis_<port>.pid
dbfilename dump-<port>.rdb
cluster-config-file nodes-<port>.conf
```  

- appendonly 설정의 경우 Cluster 설정과 직접적인 연관이 있는 설정은 아니지만, 다운되었던 Master 노드 재 시작시 appendonly 파일에 가장 최근까지 데이터가 있으므로, Cluster 운영시에는 yes 를 권장한다.

## Redis Instance 실행

```
[root@windowforsun ~]# redis-server /etc/redis-7001.conf &
[1] 23584
[root@windowforsun ~]# redis-server /etc/redis-7002.conf &
[2] 23588
[root@windowforsun ~]# redis-server /etc/redis-7003.conf &
[3] 23592
```  

```
[root@windowforsun ~]# redis-cli -c -p 7001
127.0.0.1:7001> set 1 1
(error) CLUSTERDOWN Hash slot not served
```  

- Cluster 가 아직 실행되지 않은 모습을 확인할 수 있다.

## Cluster 시작하기

```
[root@windowforsun ~]# redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```  

- Redis 5.0 이상 부터는 `redis-cli --cluster` 명령어를 통해 Cluster 를 시작할 수 있다.
- Redis 4.0 버전까지는 `redis-trib.rb` 를 사용해야 한다. 
- 실행된 Cluster 정보를 확인한다.

```
[root@windowforsun ~]# redis-cli -c -p 7001
127.0.0.1:7001> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:3
cluster_my_epoch:1
cluster_stats_messages_ping_sent:207
cluster_stats_messages_pong_sent:186
cluster_stats_messages_sent:393
cluster_stats_messages_ping_received:184
cluster_stats_messages_pong_received:207
cluster_stats_messages_meet_received:2
cluster_stats_messages_received:393
```  

- Cluster 의 노드 리스트와 각 노드 상태정보를 확인한다.

```
127.0.0.1:7001> cluster nodes
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555065315455 3 connected 10923-16383
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 master - 0 1555065313450 2 connected 5461-10922
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 myself,master - 0 1555065312000 1 connected 0-5460
```  






gem install redis

yum install ruby-devel ruby-irb ruby-rdoc ruby-ri

출처: https://binshuuuu.tistory.com/30 [지식저장소]




redis config file paht is /etc/redis.conf

redis local cluster 설정관련
https://binshuuuu.tistory.com/30
https://blog.leocat.kr/notes/2017/11/07/redis-simple-cluster
https://brunch.co.kr/@daniellim/31
http://redisgate.jp/redis/cluster/cluster_start.php



## 설치 안해도 될듯 redis-cli --cluster 로 가능한듯 내껀 5버전이라서
ruby 2.3 설치
https://zetawiki.com/wiki/CentOS6_ruby-2.3_%EC%84%A4%EC%B9%98

gem update

gem update --system

gem install redis


---
## Reference
[Redis CLUSTER Start](http://redisgate.kr/redis/cluster/cluster_start.php)  
[Redis 설치 및 Cluster 구성](https://brunch.co.kr/@daniellim/31)  
[[Redis] Redis cluster 구성하기](https://blog.leocat.kr/notes/2017/11/07/redis-simple-cluster)  
[CentOS (6.8ver) - redis cluster 구성 (master-slave & cluster)](https://binshuuuu.tistory.com/30)  
[[Redis] 클러스터 생성 및 운영](https://bstar36.tistory.com/361)  