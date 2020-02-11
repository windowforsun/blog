--- 
layout: single
classes: wide
title: "[Docker 실습] Docker Swarm 개념 및 구성"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Swarm 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Swarm
---  

## Docker Swarm
- Docker Swarm 은 여러 대의 Docker Host 를 하나인 것처럼 다룰 수 있게 해주는 Clustering 툴이고, `Orchestration` 도구 이다.
- 원래 별개의 프로젝트였지만, Docker 1.12 버전 부터 Docker Engine 에 결합되어 함께 기본 기능으로 제공된다.
- Docker Engine 을 Swarm Cluster 에 참여 시키는 상태를 `Docker Swarm mode` 라고 한다.
- `Docker Swarm mode` 인 상태에서는 Swarm 을 사용해서 동작수행 및 Container 통제가 가능하다.

### Swarm 기능
- 여러 Host 를 엮어 하나의 Host 처럼 다를 수 있도록 해주는 사용자 인테페이스 역할을 수행한다.
- Swarm(Cluster) 에 속한 Host 중 장애가 발생하더라도 서비스의 연속성을 최대한 확보할 수 있도록 Cluster 가용성 관리를 제공한다.
- Swarm 에서 구동의 단위를 Service 등 논리적인 단위를 사용해서 생명주기 및 상호관계를 정의 및 관리하는 Orchestrator 역할을 수행한다.
- [Swarm mode overview](https://docs.docker.com/engine/swarm/) 공식 문서를 참조하면 아래와 같은 키워드가 있다.
	- Cluster management integrated with Docker Engine
	- Decentralized design
	- Declarative service model
	- Desired state reconciliation
	- Multi-host networking
	- Service discovery
	- Load balancing
	- Secure by default
	- Rolling updates

### Swarm 의 구성 요소 및 용어
- `Container` : 사용자가 실행하고자 하는 프로그램의 독립적 실행을 보장하기 위한 격리된 실행 공간 또는 환경을 뜻한다.
- `Task` : Swarm 이 관리하는 작업의 최소 단위로 Container 와 1:1 관계이다.
- `Service` : Swarm 이 수행하는 단위 업무의 논리적인 단위로, Swarm 을 통해 여러 Task 로 분할되어 처리 된다.
- `Stack` : Swarm 수행하는 단위 업무의 최상위 계층으로, 하나의 Stack 은 하나 이상의 Service 로 구성된다.
- `Node` : Swarm 에 속한 Docker Engine 이 설치된 Host 또는 Docker Host 의 단위이다.
- `Manager Node` : Swarm 의 Node 들 중 Swarm Cluster 를 관리하는 Node 를 뜻한다.
	- Manager Node 는 Worker Node 가 될 수 있다.
- `Worker Node` : Swarm 의 Node 들 중 Manager node 의 관리를 통해 Container 를 실행하여 실제로 동작을 수행하는 Node 를 뜻한다.
	- Node 에 대한 별도의 설정을 하지 않았다면 기본적으로 Worker Node 가 된다.
- Swarm 구성 요소를 계층으로 표현하면 아래와 같다.
	- Stack > Service > Task(Container)
- Node 하나에 Stack 이 구성될 수도 있고, 여러 Node 에 걸쳐 Stack 구성될 수도 있다.
	
### Swarm 의 흐름
- Docker Engine 을 통해 Swarm 이 동작하는 것과 같이, Swarm 의 명령어는 Docker Engine 즉 Docker Client 에서 처리하게 된다.
1. Manager Node 에서 구성된 Service 를 설정된 Task 만큼 분할하고 각 복사본의 배치 계획을 세운다.
1. Manager Node 는 세워진 배치 계획에 따라 Task 를 Worker Node 에 배치명령을 내린다.
1. Worker Node 는 Task 를 받아 Container 를 생성하고 실행 시킨다.
- Worker Node 에 Task 가 생성되면 해당 Task 는 다른 Node 로 이전 불가하고, 할당된 Node 에서 생성, 삭제, 장애의 생명주기를 마친다.

### Swarm 구성하기
- Docker Swarm 은 복잡한 절차, 별도의 도구 없이 하나 이상의 Docker Engine 이 있다면 구성 할 수 있다.
- 하나 이상의 Docker Engine 을 위해 [Vagrant 로 Docker Swarm 테스트 환경 구성하기]({{site.baseurl}}{% link _posts/2020-02-06-docker-practice-docker swarm.md %})
을 사용한다.
- `vagrant up` 을 통해 구성된 환경을 구동시키고, `vagrant ssh manager` 명령어로 Manager Node 에 접속한다.
	
	```
	$ vagrant up
	Bringing machine 'manager' up with 'virtualbox' provider...
Bringing machine 'worker1' up with 'virtualbox' provider...
Bringing machine 'worker2' up with 'virtualbox' provider...
==> manager: Box 'ubuntu/xenial64' could not be found. Attempting to find and install...
    manager: Box Provider: virtualbox
    manager: Box Version: >= 0
==> manager: Loading metadata for box 'ubuntu/xenial64'
	
	.. 생략 ..
	
    manager: Setting up containerd.io (1.2.10-3) ...
    manager: Setting up docker-ce-cli (5:19.03.5~3-0~ubuntu-xenial) ...
    manager: Setting up docker-ce (5:19.03.5~3-0~ubuntu-xenial) ...
    manager: Setting up libltdl7:amd64 (2.4.6-0.1) ...
    manager: Processing triggers for libc-bin (2.23-0ubuntu11) ...
    manager: Processing triggers for ureadahead (0.100.0-19.1) ...
    manager: Processing triggers for systemd (229-4ubuntu21.23) ...
==> worker1: Box 'ubuntu/xenial64' could not be found. Attempting to find and install...
    worker1: Box Provider: virtualbox
    worker1: Box Version: >= 0
==> worker1: Loading metadata for box 'ubuntu/xenial64'
	
	.. 생략 ..

    worker1: Setting up containerd.io (1.2.10-3) ...
    worker1: Setting up docker-ce-cli (5:19.03.5~3-0~ubuntu-xenial) ...
    worker1: Setting up docker-ce (5:19.03.5~3-0~ubuntu-xenial) ...
    worker1: Setting up libltdl7:amd64 (2.4.6-0.1) ...
    worker1: Processing triggers for libc-bin (2.23-0ubuntu11) ...
    worker1: Processing triggers for ureadahead (0.100.0-19.1) ...
    worker1: Processing triggers for systemd (229-4ubuntu21.23) ...
==> worker2: Box 'ubuntu/xenial64' could not be found. Attempting to find and install...
    worker2: Box Provider: virtualbox
    worker2: Box Version: >= 0
==> worker2: Loading metadata for box 'ubuntu/xenial64'

	.. 생략 ..

    worker2: Setting up containerd.io (1.2.10-3) ...
    worker2: Setting up docker-ce-cli (5:19.03.5~3-0~ubuntu-xenial) ...
    worker2: Setting up docker-ce (5:19.03.5~3-0~ubuntu-xenial) ...
    worker2: Setting up libltdl7:amd64 (2.4.6-0.1) ...
    worker2: Processing triggers for libc-bin (2.23-0ubuntu11) ...
    worker2: Processing triggers for ureadahead (0.100.0-19.1) ...
    worker2: Processing triggers for systemd (229-4ubuntu21.23) ...
	```  
	
	- Manager Node 로 사용할 1개의 호스트와 Worker Node 로 사용할 2개의 호스트로 구성했다.

### Swarm Cluster 초기화
- 구성된 여러 개의 Docker Engine 을 Swarm Cluster 로 구성하기 위해서는 Manager Node 에서 `docker swarm init` 명령어를 수행해 주면된다.
- Manager Node 에 접속하고, `docker swarm init` 명령어를 수행한다.

	```
	$ vagrant ssh manager
	.. 생략 ..
	vagrant@manager:~$ sudo docker swarm init --advertise 192.168.100.10
	unknown flag: --advertise
	See 'docker swarm init --help'.
	vagrant@manager:~$ sudo docker swarm init --advertise-addr 192.168.100.10
	Swarm initialized: current node (yenkumtb6whjug68c32babba9) is now a manager.
	
	To add a worker to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642g6edu6j2eqrgdvr9 192.
	168.100.10:2377
	
	To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
	```  
	
	- `--advertise-addr` 옵션을 통해 Swarm 에서 Node 연결로 사용할 주소를 명시한다.
	- 새로운 Node 를 추가할 때 `docker swarm init` 명령어의 출력 결과로 나온 `docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642g6edu6j2eqrgdvr9 192.168.100.10:2377` 을 사용하면 된다.
- `docker info` 를 통해 현재 Docker Engine 의 상태를 확인 할 수 있는데, Manager Node 에서 결과는 아래와 같다.
	
	```
	vagrant@manager:~$ sudo docker info
	
	.. 생략 ..
	
	 Swarm: active
	  NodeID: yenkumtb6whjug68c32babba9
	  Is Manager: true
	  ClusterID: to9ltuqve7beyueasd1qolw5m
	  Managers: 1
	  Nodes: 1
	  Default Address Pool: 10.0.0.0/8
	  SubnetSize: 24
	  Data Path Port: 4789
	  Orchestration:
	   Task History Retention Limit: 5
	  Raft:
	   Snapshot Interval: 10000
	   Number of Old Snapshots to Retain: 0
	   Heartbeat Tick: 1
	   Election Tick: 10
	  Dispatcher:
	   Heartbeat Period: 5 seconds
	  CA Configuration:
	   Expiry Duration: 3 months
	   Force Rotate: 0
	  Autolock Managers: false
	  Root Rotation In Progress: false
	  Node Address: 192.168.100.10
	  Manager Addresses:
	   192.168.100.10:2377
	   
	.. 생략 ..
	```  
- `docker node ls` 를 통해 현재 Swarm Cluster 로 구성된 Node 를 조회하면 아래와 같다.

	```
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VER	SION
	yenkumtb6whjug68c32babba9 *   manager             Ready               Active              Leader              19.03.5
	```  
	
	- 지금은 아직 추가된 Node 가 없기 때문에, Manager Node 한개만 조회 된다.
	
### Swarm Cluster 에 Node 추가하기
- 구성된 Swarm Cluster 에 참여하기 위해서는 Token 값이 필요한데, `docker swarm init` 명령어의 결과로 출력되지만 잊었을 경우에는 아래 명령어로 다시 조회 할 수 있다.

	```
	Worker Node 를 추가할 때 사용하는 token 조회
	vagrant@manager:~$ sudo docker swarm join-token worker
	To add a worker to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642g6edu6j2eqrgdvr9 192.168.100.10:2377
	
	Manager Node 추가할 때 사용하는 token 조회
	vagrant@manager:~$ sudo docker swarm join-token manager
	To add a manager to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8wxl64or7q4s5hykgs8b3eakd 192.168.100.10:2377
	
	-q 옵션을 사용하면 token 자체만 조회 할 수 있다.
	vagrant@manager:~$ sudo docker swarm join-token worker -q
	SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642g6edu6j2eqrgdvr
	```  
	
- Worker Node 로 생성했던 `worker1`, `worker2` 를 구성된 Swarm Cluster 에 참여 시켜본다.

	```
	Worker Node 1
	$ vagrant ssh worker1
	.. 생략 ..
	vagrant@worker1:~$ sudo docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642
	g6edu6j2eqrgdvr9 192.168.100.10:2377
	        This node joined a swarm as a worker.
	vagrant@worker1:~$ exit
	logout
	Connection to 127.0.0.1 closed.
	
	Worker Node 2
	$ vagrant ssh worker2
	.. 생략 ..
	vagrant@worker2:~$ sudo docker swarm join --token SWMTKN-1-16qr38qqzlk834h642ejskmlc4mrx9uw8h6dqbpydsev6wry3j-8f9n1f642
	g6edu6j2eqrgdvr9 192.168.100.10:2377
	        This node joined a swarm as a worker.
	```  
	
- 각 Node 에서 `docker info` 를 통해 Docker Engine 의 상태를 확인하면 아래와 같다.

	```
	Worker Node 1
	vagrant@worker1r:~$ sudo docker info
	
	.. 생략 ..
	
	 Swarm: active
	  NodeID: no6ck7n6btli6vzcbs5csmoj8
	  Is Manager: false
	  Node Address: 192.168.100.11
	  Manager Addresses:
	   192.168.100.10:2377
	   
	.. 생략 ..
	
	Worker Node 2
	vagrant@worker2:~$ sudo docker info
	
	.. 생략 .. 
	
	 Swarm: active
	  NodeID: pysqwwd8wf59n56lxi10vr55m
	  Is Manager: false
	  Node Address: 192.168.100.12
	  Manager Addresses:
	   192.168.100.10:2377
	   
	.. 생략 ..
	```  
	
- Manager Node 에서 `docker node ls` 를 통해 현재 Swarm Cluster 의 구성을 조회하면 아래와 같다.

	```
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VER	SION
	yenkumtb6whjug68c32babba9 *   manager             Ready               Active              Leader              19.03.5
	no6ck7n6btli6vzcbs5csmoj8     worker1             Ready               Active                                  19.03.5
	pysqwwd8wf59n56lxi10vr55m     worker2             Ready               Active                                  19.03.5
	```  
	
	- 새로 추가한 `worker1`, `worker2` Node 가 추가 된 것을 확인 할 수 있다.

### Swarm Cluster 떠나기
- 구성된 Swarm Cluster 를 떠나는 방법은 `docker swarm leave` 명령어로 가능하다.

	```
	Worker Node 1
	vagrant@worker1:~$ sudo docker swarm leave
	Node left the swarm.
	vagrant@worker1:~$ sudo docker info
	
	.. 생략 ..
	
	 Swarm: inactive
	 
	.. 생략 ..
	
	Worker Node 2
	vagrant@worker2:~$ sudo docker swarm leave
	Node left the swarm.
	vagrant@worker2:~$ sudo docker info
	
	.. 생략 ..
	
	 Swarm: inactive
	 
	.. 생략 ..
	
	Manager Node
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VER	SION
	yenkumtb6whjug68c32babba9 *   manager             Ready               Active              Leader              19.03.5
	no6ck7n6btli6vzcbs5csmoj8     worker1             Down                Active                                  19.03.5
	pysqwwd8wf59n56lxi10vr55m     worker2             Down                Active                                  19.03.5
	vagrant@manager:~$ sudo docker swarm leave -f
	Node left the swarm.
	
	.. 생략 ..
	
	 Swarm: inactive
	 
	.. 생략 ..
	```  
	
	- Manager Node 의 경우 `-f` 옵션으로 강제로 Swarm Cluster 를 떠나도록 했다.


---
## Reference
[How nodes work](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/)
[Swarm mode key concepts](https://docs.docker.com/engine/swarm/key-concepts/)
[Docker Swarm을 이용한 쉽고 빠른 분산 서버 관리](https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html)

	