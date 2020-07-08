--- 
layout: single
classes: wide
title: "[Redis 실습] 한 머신에서 Cluster 설정하기"
header:
  overlay_image: /img/redis-bg.png
excerpt: '한 머신에서 Redis Cluster 설정을 해보자'
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
	- 출력예시 명령어는 Slave 없이 3개의 Master 노드만으로 Cluster 를 구성하는 명령어이다.
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

- 각 Node 의 Slot 은 아래와 같이 할당 되이져 있는 것을 확인할 수 있다.
	- 7001 : 0-5460
	- 7002 : 5461-10922
	- 7003 : 10923-16383

![cluster 디자인]({{site.baseurl}}/img/redis/practice-onemachinecluster-clusterdesign.png)

- 현재 구성된 Cluster 는 위의 그림과 같이 3개의 다른 포트를 갖는 Master 노드들로 구성되어 있다.

```
[root@windowforsun ~]# redis-cli -c -p 7001
127.0.0.1:7001> set key hello
-> Redirected to slot [12539] located at 127.0.0.1:7003
OK
127.0.0.1:7003> set key2 world
-> Redirected to slot [4998] located at 127.0.0.1:7001
OK
127.0.0.1:7001> get key
-> Redirected to slot [12539] located at 127.0.0.1:7003
"hello"
127.0.0.1:7003> get key2
-> Redirected to slot [4998] located at 127.0.0.1:7001
"world"
```  

- `-c` 옵션을 주고 접속했을 때 키를 입력하면 포트가 변경되는 것을 확인 할 수 있다. 이를 스마트 클라이언트라고 한다.
- 스마트 클라이언트란
	- Cluster 로 구성된 Redis Server 들은 각 할당된 Slot 에 해당하는 key 에대한 명령어 처리만 가능하다.
	- 스마크 클라이언트는 현재 7001 Server 라도 명렁어 처리에 사용하는 key 가 7003 Server 에 해당하는 Slot 이라면 7003 Server 로 접속해서 처리를 수행한다.


## Cluster 관련 장애 복구 Test
### Node 다운
- 7001 Server 를 다운 시킨다.
	- `debug segfault` 는 debug 용도로 Server 를 crash 시키는 명령어이다.
	
```
[root@windowforsun ~]# redis-cli -p 7001
127.0.0.1:7001> debug segfault
Could not connect to Redis at 127.0.0.1:7001: Connection refused
(2.37s)
not connected> quit
```  

- Cluster 정보를 확인한다.

```
127.0.0.1:7002> cluster info
cluster_state:fail      // 현재 Cluster 가 비정상인 상태
cluster_slots_assigned:16384
cluster_slots_ok:10923  // 현재 Cluster 에서 사용가능한 Slot 수, 하지만 현재 Cluster 는 사용 불가능함
cluster_slots_pfail:0
cluster_slots_fail:5461 // 다운된 Slot 수
cluster_known_nodes:3   // Cluster 에 참가하고 있는 Redis Server 수
cluster_size:3  // Cluster 에서 Slot 할당된 Master 노드 수
cluster_current_epoch:3
cluster_my_epoch:2
cluster_stats_messages_ping_sent:207692
cluster_stats_messages_pong_sent:207139
cluster_stats_messages_meet_sent:2
cluster_stats_messages_sent:414833
cluster_stats_messages_ping_received:207138
cluster_stats_messages_pong_received:206264
cluster_stats_messages_meet_received:1
cluster_stats_messages_fail_received:1
cluster_stats_messages_received:413404
127.0.0.1:7002> cluster node
(error) ERR Unknown subcommand or wrong number of arguments for 'node'. Try CLUSTER HELP.
```  

```
127.0.0.1:7002> cluster nodes
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 master,fail - 1555254927265 1555254924757 1 disconnected 0-5460
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 myself,master - 0 1555255108000 2 connected 5461-10922
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555255108577 3 connected 10923-16383
```  

### 노드 재시작 및 Cluster 복구
- 다운 시켰던 7001 Server 를 시작 시킨다.

```
[root@windowforsun ~]# redis-server /etc/redis-7001.conf &
[1] 32657
```  

- 다시 Cluster 정보를 확인해 보면 정상으로 돌아온 것을 확인 할 수 있다.

```
[root@windowforsun ~]# redis-cli -p 7002
127.0.0.1:7002> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:3
cluster_my_epoch:2
cluster_stats_messages_ping_sent:211570
cluster_stats_messages_pong_sent:207510
cluster_stats_messages_meet_sent:2
cluster_stats_messages_sent:419082
cluster_stats_messages_ping_received:207509
cluster_stats_messages_pong_received:206628
cluster_stats_messages_meet_received:1
cluster_stats_messages_fail_received:1
cluster_stats_messages_received:414139
```  

```
127.0.0.1:7002> cluster nodes
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 master - 0 1555255454513 1 connected 0-5460
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 myself,master - 0 1555255453000 2 connected 5461-10922
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555255455015 3 connected 10923-16383
```  

## 기존 Cluster 에 Master 노드 추가하기
- 7004 포트 Redis Server 설정 파일을 준비한다.
	- redis-7001.conf 를 redis-7004.conf 로 복사
	
```
[root@windowforsun ~]# cp /etc/redis-7001.conf /etc/redis-7004.conf
```  

- redis-7004.conf 파일에서 포트 등 수정해야 할 부분들을 수정해 준다.

```
port <port>
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
pidfile /var/run/redis_<port>.pid
dbfilename dump-<port>.rdb
cluster-config-file nodes-<port>.conf
```  

- 7004 포트 Redis Server 를 실행 시킨다.

```
[root@windowforsun ~]# redis-server /etc/redis-7004.conf &
[2] 402
```  

- 현재 구성된 Cluster 에 7004 Redis Server 를 추가시킨다.
	- `redis-cli --cluster add-node <추가IP>:<PORT> <구성된ClusterIP>:<PORT>`
	- 구성된 Cluster IP PORT 중 하나를 입력한다.
	
```
[root@windowforsun ~]# redis-cli --cluster add-node 127.0.0.1:7004 127.0.0.1:7001
>>> Adding node 127.0.0.1:7004 to cluster 127.0.0.1:7001
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:7004 to make it join the cluster.
[OK] New node added correctly.
```  

- 추가만 된 상태이고, 아직 추가한 7004 Redis Server 에 Slot 이 할당된 상태는 아니므로 Cluster 정보를 확인해본다.

```
[root@windowforsun ~]# redis-cli -p 7001
127.0.0.1:7001> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:4   // 노드가 추가됨
cluster_size:3          // 아직 Slot 이 할당되지 않아 size 는 그대로
cluster_current_epoch:3
cluster_my_epoch:1
cluster_stats_messages_ping_sent:50823
cluster_stats_messages_pong_sent:43607
cluster_stats_messages_update_sent:1
cluster_stats_messages_sent:94431
cluster_stats_messages_ping_received:43606
cluster_stats_messages_pong_received:50823
cluster_stats_messages_meet_received:1
cluster_stats_messages_received:94430
127.0.0.1:7001> cluster nodes
7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004@17004 master - 0 1555311299507 0 connected
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 master - 0 1555311299001 2 connected 5461-10922
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555311299000 3 connected 10923-16383
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 myself,master - 0 1555311298000 1 connected 0-5460
```  

- 7004 Redis Server 에 Slot 할당을 위해 resharding 을 한다.

```
[root@windowforsun ~]# redis-cli --cluster reshard 127.0.0.1:7001
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: 7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004
   slots: (0 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 10        // 몇개의 Slot 을 옮길지
What is the receiving node ID? 7943a7038646b9ca8c73f8547f227f46e9068ad0     // 받을 노드 ID 를 입력한다.
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 995c0743f7e499b2b6af8e808c148d200f40bb6e        // 할당 해줄 노드 ID 를 입력한다. 모든 노드에서 할당 받을 경우 all 을 입력
Source node #2: done    // 소드 노드 ID 입력이 끝났으면 done 을 입력

Ready to move 10 slots.
  Source nodes:
    M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
       slots:[0-5460] (5461 slots) master
  Destination node:
    M: 7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004
       slots: (0 slots) master
  Resharding plan:
    Moving slot 0 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 1 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 2 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 3 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 4 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 5 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 6 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 7 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 8 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
    Moving slot 9 from 995c0743f7e499b2b6af8e808c148d200f40bb6e
Do you want to proceed with the proposed reshard plan (yes/no)? yes
Moving slot 0 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 1 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 2 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 3 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 4 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 5 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 6 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 7 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 8 from 127.0.0.1:7001 to 127.0.0.1:7004:
Moving slot 9 from 127.0.0.1:7001 to 127.0.0.1:7004:

```  

- 7004 Redis Server 에 Slot 이 할당 된 것을 확인 한다.

```
127.0.0.1:7001> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:4
cluster_size:4      // Slot 할당 후 Cluster 사이즈 변경됨
cluster_current_epoch:4
cluster_my_epoch:1
cluster_stats_messages_ping_sent:55150
cluster_stats_messages_pong_sent:45977
cluster_stats_messages_fail_sent:4
cluster_stats_messages_update_sent:1
cluster_stats_messages_sent:101132
cluster_stats_messages_ping_received:45973
cluster_stats_messages_pong_received:53638
cluster_stats_messages_meet_received:4
cluster_stats_messages_fail_received:2
cluster_stats_messages_update_received:4
cluster_stats_messages_received:99621
127.0.0.1:7001> cluster nodes
7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004@17004 master - 0 1555311587354 4 connected 0-9      // Slot 이 할당 됨
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 master - 0 1555311586352 2 connected 5461-10922
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555311586051 3 connected 10923-16383
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 myself,master - 0 1555311586000 1 connected 10-5460
```  

![새로 추가된 Cluster 구성]()

## 기존 Cluster 에 Master 노드 제거하기
- 7004 노드 제거를 위해 7004 노드에 있는 Slot 을 7001 로 Resharding 한다.

```
[root@windowforsun ~]# redis-cli --cluster reshard 127.0.0.1:7001
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[10-5460] (5451 slots) master
M: 7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004
   slots:[0-9] (10 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 10        // Resharding 할 Slot 의 수
What is the receiving node ID? 995c0743f7e499b2b6af8e808c148d200f40bb6e     // Slot 을 받을 노드
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 7943a7038646b9ca8c73f8547f227f46e9068ad0    // Slot 을 보낼 노드
Source node #2: done

Ready to move 10 slots.
  Source nodes:
    M: 7943a7038646b9ca8c73f8547f227f46e9068ad0 127.0.0.1:7004
       slots:[0-9] (10 slots) master
  Destination node:
    M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
       slots:[10-5460] (5451 slots) master
  Resharding plan:
    Moving slot 0 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 1 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 2 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 3 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 4 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 5 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 6 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 7 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 8 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
    Moving slot 9 from 7943a7038646b9ca8c73f8547f227f46e9068ad0
Do you want to proceed with the proposed reshard plan (yes/no)? yes
Moving slot 0 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 1 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 2 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 3 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 4 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 5 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 6 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 7 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 8 from 127.0.0.1:7004 to 127.0.0.1:7001:
Moving slot 9 from 127.0.0.1:7004 to 127.0.0.1:7001:
```  

- cluster info 를 보면 cluster_known_nodes:4 이고 cluster_size:3 인것을 확인 할 수 있다.

```
[root@windowforsun ~]# redis-cli -p 7001
127.0.0.1:7001> cluster info
cluster_state:ok

생략 ..

cluster_known_nodes:4
cluster_size:3

생략 ..
```  

- Cluster 에서 7004 노드를 제거한다.
	- `redis-cli --cluster del-node <기존ClusterIP>:<PORT> <제거할노드ID>

```
[root@windowforsun ~]# redis-cli --cluster del-node 127.0.0.1:7001 7943a7038646b9ca8c73f8547f227f46e9068ad0
>>> Removing node 7943a7038646b9ca8c73f8547f227f46e9068ad0 from cluster 127.0.0.1:7001
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
[1]+  Done                    redis-server /etc/redis-7004.conf
```  

- Cluster 에서 노드가 제거되고 shutdown 되었다.

```
[root@windowforsun ~]# redis-cli -p 7001
127.0.0.1:7001> cluster info
cluster_state:ok

생략 ...

cluster_known_nodes:3
cluster_size:3

생략 ...
```  

- cluster info 에서 cluster_known_nodes 까지 변경된 것을 확인할 수 있다.


## 에러 및 이슈 관련
- Cluster 노드를 추가하거나 Reshard 를 하다보면 아래와 같은 WARNING 메시지를 볼 수 있다.

```
[root@windowforsun ~]# redis-cli --cluster add-node 127.0.0.1:7004 127.0.0.1:7001
>>> Adding node 127.0.0.1:7004 to cluster 127.0.0.1:7001
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: 433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
[WARNING] Node 127.0.0.1:7001 has slots in importing state 12539.
[WARNING] The following slots are open: 12539.
>>> Check slots coverage...
[OK] All 16384 slots covered.
```  

- cluster nodes 를 확인하보면 아래 처럼 7001 포트에 해당되는 Slot 이 아닌, Slot 번호가 보인다.

```
127.0.0.1:7001> cluster nodes
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 master - 0 1555256681791 2 connected 5461-10922
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 myself,master - 0 1555256680000 1 connected 0-5460 [12539-<-f2bcff12612ab068b68f48bc46de575895b947e5]
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555256680788 3 connected 10923-16383
```  

- 아래 명령어를 통해 해당 Slot 을 Stable 시켜주면 

```
[root@dontrise windowforsun_com]# redis-cli -p 7001 cluster setslot 12539 STABLE
OK
```  

- 아래와 같이 정상으로 Slot 들이 각 node 에 잘 분배된 것을 확인 할 수 있다.

```
127.0.0.1:7001> cluster nodes
433e9fbd5a51a829bfbf87a8d1755c78f852f0d2 127.0.0.1:7002@17002 master - 0 1555303895862 2 connected 5461-10922
f2bcff12612ab068b68f48bc46de575895b947e5 127.0.0.1:7003@17003 master - 0 1555303895000 3 connected 10923-16383
995c0743f7e499b2b6af8e808c148d200f40bb6e 127.0.0.1:7001@17001 myself,master - 0 1555303894000 1 connected 0-5460
```  

---
## Reference
[Redis CLUSTER Start](http://redisgate.kr/redis/cluster/cluster_start.php)  
[Redis 설치 및 Cluster 구성](https://brunch.co.kr/@daniellim/31)  
[[Redis] Redis cluster 구성하기](https://blog.leocat.kr/notes/2017/11/07/redis-simple-cluster)  
[CentOS (6.8ver) - redis cluster 구성 (master-slave & cluster)](https://binshuuuu.tistory.com/30)  
[[Redis] 클러스터 생성 및 운영](https://bstar36.tistory.com/361)  