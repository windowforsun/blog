--- 
layout: single
classes: wide
title: "[Docker 실습] Docker Swarm Cluster"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Swarm Cluster 에 대해 알아보고, 장애 상황을 실습해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Swarm
  - Cluster
---  

## Docker Swarm Cluster
- 클러스터를 사용할 때 아래 2가지 관점에 대한 고려가 필요하다.
	- `서비스의 가용성` : 클러스터를 이루고 있는 구성 중 일부분에 장애가 발생했을 때, 전체 서비스에 영향을 주지 않아야 한다.
	- `제어의 가용성` : 일부분에 장애가 발생하더라도 구성된 클러스터는 제어 가능한 생태여야 한다.
- Swarm Cluster 의 경우 위 2가지를 아래와 같은 방식으로 보장하고 있다.
	- `서비스의 가용성` : Manager Node 에서 하나의 Service 를 Task 로 분할하고, 각 Task 에 Slot 을 할당해 빈 Slot 이 없도록 관리하는 Scheduling 기능에 의해 보장한다.
	- `제어의 가용성` : 복수의 Manager Node 로 구성된 Manager Pool 을 사용해서, 일부 Manager Node 가 죽더라도 Cluster 기능의 정상을 현상황 정보 공유를 통해 보장한다.
- 클러스터는 아래와 같은 두 가지로 분류 된다.
	- `부하 분산 / 병렬 클러스터` : 같은 일이나 일련의 일을 병렬로 처리하기 위한 클러스터
	- `HA 클러스터` : 장애상황을 유연하게 대처하기 위한 가용성 클러스터

### HA(High Availability) 클러스터
 - 가용성 클러스터는 클러스터의 구성 중 일부가 불능 상태가 되더라도 수행 하는 동작은 정상적으로 수행하며 최소한의 영향만 미치도록 하는 것이다.
 - 이런 가용성 클러스터에는 아래와 같은 특징 및 기능이 필요하다.
 * 이중화 : 구성의 모든 자원이 둘 이상인 상태에서, 그 중 일부에 문제가 발생했을 때도 다른 부분으로 업무 수행이 가능해야 한다. 서버 장비 뿐만 아니라, 네트워크, 저장소 등 전체 시스템 구성에 모두 이뤄져야 한다.
 * 동기화 : 업무 수행에 필요한 데이터, 클러스터 관련 정보등 모든 데이터가 특정 자원에 대해 Share Storage, 실시간 동기화등 을 통해 독립성을 보장해야한다.  
 * 지능화 : 장애가 발생했을 때 장애 지점을 파악하거나, 복구 방법 등을 파악하고 처리할 수 있어야 한다.
 * 자동화 : 이중화, 동기화, 지능화가 모두 만족 된상황에서 관련 처리가 관리자의 개입 없이 가능하고, 장애 시간을 줄일 수 있어야 한다.
 
### Swarm Cluster
- Swarm Cluster 는  Worker Node 에서 하나의 Service 를 Task 로 분할하여 부하분산을 수행한다.
- Manager Node 는 이런 구성에서 안전적인 작업 수행을 위해 아래와 같은 구현 사항을 가지고 있다.
* 이중화 : 하나 이상의 Manager Node 를 가질 수 있는 구조를 통해, Swarm Cluster 에 구성돼 있는 모든 Node 들은 상황에 따라 Manager Node 로 쉽게 전환하여 Manager Node 의 작업을 수행 할 수 있다.
* 동기화 : 하나 이상의 Manager Node 인 Manager Pool 안에서 각 Manager Node 들은 상호간 상태 정보를 별도의 Storage 없이 실시간으로 동기화 한다. 
* 지능화 : 장애 대한 처리를 위해 [Raft Concensus Algorithm](http://thesecretlivesofdata.com/raft/) 를 사용한다. Manager Pool 에서 과반수 이상이 유효한 경우, 서비스를 멈추지 않고 제공이 가능하고 이러한 특징으로 Manager Pool 의 수는 홀수개를 권장한다.
* 자동화 : Task Scheduler 를 통해 장애 상황에서 클러스터의 안정성 유지와 업무의 연속성을 보장한다.
 
### Swarm Cluster 구성요소
- Swarm Cluster 의 특징 중 하나는 아주 간단한 Manager Node, Worker Node 의 구조를 가지고 있다는 점이다.
- Manager Node 는 Swarm Cluster 의 관리 기능을 수행하고, Worker Node 에서는 실제로 업무를 수행하는 Container 가 구동 된다.
- 한 호스트에서 Swarm Cluster 를 처음 생성 했다면, 해당 호스트는 바로 Manager Node 의 역할을 수행한다.
- Manager Node 는 아래와 같은 역할을 수행한다.
	- Swarm Cluster 와 사용자간의 인터페이스 역할을 수행하며, 사용자로 부터 Swarm Cluster 관련 명령을 받아 이를 Docker Engine 을 통해 수행한다.
	- Swarm Cluster 를 구성하고 있는 Node 를 가입, 탈퇴 등을 처리하는 Node Manager 역할을 수행한다.
	- 사용자가 구성한 Service 에 맞게 Worker Node 들에게 Task 를 분배해주는 Task Scheduler 역할과 Service 의 생명주기를 관리한다.
	- 다수의 Manager Node 들과 클러스터 정보를 동기화하며 클러스터의 가용성을 관리하는 Swarm Cluster Manager 역할을 수행한다.
	
### Swarm Cluster 구성 방법
> #### Split Brain
> - [Split Brain](https://en.wikipedia.org/wiki/Split-brain) 이란 클러스터 환경에서 장애가(네트워크 단절) 발생했을 때, 모든 노드들이 자신이 primary 라고 인식하는 상황을 뜻한다.
> - Swarm Cluster 에서는 짝수개의 Manager Node 가 있는 생태에서 절반의 Manager Node 에 장애가 발생했을 때, 현재 상황이 Manager Pool 이 과반수 인지 아닌지 판별하기 여려운 상황이다.

#### Manager Node 와 Worker Node 가 분리된 구성

![그림 1]({{site.baseurl}}/img/docker/practice-dockerswarmcluster-1.png)

- 위와 같은 구성은 작은 규모보다는 큰 규모에 어울리는 구성이다.
- Manager Pool 은 Swarm Cluster 를 관리하는 역할만 수행하고, Worker Pool 은 업무를 수행하는 역할만 수행한다.
- 이렇게 Manager Node 와 Worker Node 를 분리하는 구성은 `제어의 가용성` 확보에 유리하지만, Swarm Cluster 특성상 홀수개의 Manager Pool 을 유지 해야 한다는 점을 유의해야 한다.

#### Manager Node 와 Worker Node 가 혼합된 구성

![그림 1]({{site.baseurl}}/img/docker/practice-dockerswarmcluster-2.png)

- 적은 규모에 적합한 구성이다.
- Manager Node 가 Worker Node 의 역할까지 수행하기 때문에 비용적 측면이나, 보다 유연한 클러스터 구성이 가능하다.
  
## Manager Node 가 짝수개 일때 Swarm Cluster
- Swarm Cluster 의 경우 Manager Node 의 개수가 짝수개일 때, `Split Brain` 상황으로 인해 가용성 기능 제공이 불가할 수 있다고 언급했었다.
- 실제로 Swarm Cluster 를 2개의 Manager Node 로 구성해서 이를 테스트 해본다.
- 현재 구성된 Swarm Cluster 의 Node 는 아래와 같이, Manager Node 1, Worker Node 1 로 구성돼 있다.
	
	```
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VER
	SION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active                                  19.03.5
	```  
	
### Worker Node, Manager Node 로 승격 시키기
- 테스트 전 먼저 Worker Node 를 Manager Node 로 승격(promote) 시켜야 하는데, `docker node promote` 명령어를 통해 가능하다.

	```
	vagrant@manager:~$ sudo docker node promote worker1
	Node worker1 promoted to a manager in the swarm.
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VER
	SION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active              Reachable           19.03.5
	```  
	
	- Manager Node 를 Worker Node 로 만드는 방법은 `docker node demote <NODE>` 를 통해 가능하다.
	
	- `status` 가 공백에서 `Reachable` 로 변경된 것을 확인 할 수 있다.
- `docker node inspect` 명령어로 두 Node 의 상제 정보를 확인하면 아래와 같다.

	```
	vagrant@manager:~$ sudo docker node inspect --pretty manager
	ID:                     z2h43qxvvr5u0poie80zjc5kg
	Hostname:               manager
	
	.. 생략 ..
	
	Manager Status:
	 Address:               192.168.100.10:2377
	 Raft Status:           Reachable
	 Leader:                Yes
	 
	.. 생략 ..
	 
	vagrant@manager:~$ sudo docker node inspect --pretty worker1
	ID:                     5x09cav0mappy087xxjm4rcb1
	Hostname:               worker1
	
	.. 생략 ..
	
	Manager Status:
	 Address:               192.168.100.11:2377
	 Raft Status:           Reachable
	 Leader:                No
	 
	.. 생략 ..	 
	```  
	
	- Node 정보에 Manager 관련 정보가 있는 것을 확인 할 수 있다.
- `docker info` 를 통해 현재 Docker Engine 에서의 상태를 확인하면 아래와 같다.

	```
	vagrant@manager:~$ sudo docker info
	
	.. 생략 ..	 
	
	 Swarm: active
	  NodeID: z2h43qxvvr5u0poie80zjc5kg
	  Is Manager: true
	  ClusterID: cma0zqa900pmd2mddabi5uu7e
	  Managers: 2
	  Nodes: 2
	  Default Address Pool: 10.0.0.0/8
	  SubnetSize: 24
	  Data Path Port: 4789
	  Orchestration:
	   Task History Retention Limit: 3
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
	   192.168.100.11:2377	   
	   
	.. 생략 ..	 
	```  
	
	- `Manager Addresses` 에서 Manager Node 주소가 `manager` 호스트, `worker1` 호스트 의 주소 2개로 나오는 것을 확인 할 수 있다.
	
### Leader Down 테스트
- 테스트를 위해 간단한 Nginx 서비스를 아래 명령어를 통해 실행 시킨다.

	```
	vagrant@manager:~$ sudo docker service create --name swarmcluster-nginx --replicas 2 -p 80:80 nginx
	dftmc4851pmcdtek8m2wmxiic
	overall progress: 2 out of 2 tasks
	1/2: running   [==================================================>]
	2/2: running   [==================================================>]
	verify: Service converged
	vagrant@manager:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE	        ERROR               PORTS
	p7c7hywop4zu        swarmcluster-nginx.1   nginx:latest        worker1             Running             Running 40 seconds ago
	mhte19fi539v        swarmcluster-nginx.2   nginx:latest        manager             Running             Running 40 seconds ago
	```  
	
	- Service 의 2개의 Task 가 각 2개의 Node 에서 실행 중인 상태이다.
- Manager Node 중 가장 중심이 되는 Leader 인 `manager` 호스트를 먼저 강제로 죽여 테스트를 해본다.

	```
	$ vagrant halt manager
	==> manager: Attempting graceful shutdown of VM...
	```  
	
	- Vagrant 관련 명령은 VM 이아닌, 호스트에서 Git Bash, Windows 명령 프롬프트 등에서 실행한다.
- Manager Node 중 하나인 `worker1` 호스트에 접속해서 현재 Swarm Cluster 에 구성된 Node 의 상태를 확인 해본다.
	
	```
	$ vagrant ssh worker1
	.. 생략 ..
	vagrant@worker1:~$ sudo docker node ls
	Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.
	```  
	
	- 리더가 죽었다는 말과 구성된 Manager 의 수가 적고, 절반 이상의 Manager 들이 활성화 되야 한다는 메시지와 함께 Swarm Cluster 의 기능은 가용하지 못한 상태가 되었다.
- 복구를 위해 다시 `manager` 호스트를 올리고 접속해서 Swarm Cluster 의 Node 상태를 확인 해본다.

	```
	$ vagrant up manager
	Bringing machine 'manager' up with 'virtualbox' provider...
	==> manager: Checking if box 'ubuntu/xenial64' version '20200129.0.0' is up to date...
	==> manager: A newer version of the box 'ubuntu/xenial64' for provider 'virtualbox' is
	
	.. 생략 ..
	
	==> manager: Machine already provisioned. Run `vagrant provision` or use the `--provision`
	==> manager: flag to force provisioning. Provisioners marked to run always will still run.
	$ vagrant ssh manager
	.. 생략 ..
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Reachable           19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active              Leader              19.03.5
	```  
	
	- 원래 Leader 였던 `manager` 호스트를 내렸다가 올리자, `worker1` 이 Leader 가 된 것을 확인 할 수 있다.
	
### Manager Down 테스트
- 이어서 바로 시작하기 때문에 `manager` 호스트는 일반 Manager Node 이고, `worker1` Manager Node 는 Leader 역할인 상황이다.
- Manager Node 중 Leader 가 아닌, 일반 Manager Node 가 다운 됐을 때 상황을 테스트 해본다. `manager` 호스트를 다운 시키고, `worker1` 호스트에 접속해 상태를 확인해 본다.

	```
	$ vagrant halt manager
	==> manager: Attempting graceful shutdown of VM...
	$ vagrant ssh worker1
	.. 생략 ..
	vagrant@worker1:~$ sudo docker node ls
	Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.
	vagrant@worker1:~$ sudo docker ps
	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
	c5e1436d9fa3        nginx:latest        "nginx -g 'daemon of…"   25 minutes ago      Up 25 minutes       80/tcp              swarmcluster-nginx.1.p7c7hywop4zuwj6r7eoo101x4
	```  
	
	- 일반 Manager Node 가 죽더라도, 동일한 에러 메시지가 나오며, Swarm Cluster 는 가용 불가 상태이다.
- `manager` 호스트를 다시 구동 시키고, Node 와 Service 의 상태를 확인해본다.

	```
	$ vagrant ssh worker1
	.. 생략 ..
	vagrant@worker1:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg     manager             Ready               Active              Reachable           19.03.5
	5x09cav0mappy087xxjm4rcb1 *   worker1             Ready               Active              Leader              19.03.5
	vagrant@worker1:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
	p7c7hywop4zu        swarmcluster-nginx.1       nginx:latest        worker1             Running             Running 15 minutes ago     
	e5rd3857bb6u        swarmcluster-nginx.2       nginx:latest        manager             Running             Running about a minute ago 
	h753vjxunqng         \_ swarmcluster-nginx.2   nginx:latest        manager             Shutdown            Complete 2 minutes ago     
	mhte19fi539v         \_ swarmcluster-nginx.2   nginx:latest        manager             Shutdown            Complete 15 minutes ago    
	```  
	
	- Swarm Cluster 및 Service 모두 정상 상태로 돌아 온것을 확인 할 수 있다.
	- `docker service ps` 이력을 보면 `manager` 호스트가 다운되며 내려갔던 Container 들의 이력과 복구 된 Container 를 확인 할 수 있다.
	
## Manager Node 가 홀수개 일때 Swarm Cluster
- Docker 에서 권장하는 홀수개의 Manager Node 를 구성해서 비슷한 상황을 테스트 해본다.
- 새로운 호스트 `worker2` 를 Swarm Cluster 에 Manager Node 로 바로 추가한다.

	```
	vagrant@worker1:~$ sudo docker swarm join-token manager
	To add a manager to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-3afbwbonaogdup7cgb2rdwak40h6o07evu1nkbeos2wczsmclw-2fu7jobta4cp5jeelolnvfnzh 192.168.100.11:2377
	
	vagrant@worker1:~$ exit
	logout
	Connection to 127.0.0.1 closed.
	$ vagrant ssh worker2
	.. 생략 ..
	vagrant@worker2:~$ sudo docker swarm join --token SWMTKN-1-3afbwbonaogdup7cgb2rdwak40h6o07evu1nkbeos2wczsmclw-2fu7jobta4cp5jeelolnvfnzh 192.168.100.11:2377
	This node joined a swarm as a manager.
	vagrant@worker2:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg     manager             Ready               Active              Reachable           19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active              Leader              19.03.5
	gnrqiccn1sexhomztqc3ic76p *   worker2             Ready               Active              Reachable           19.03.5
	```  

	- 구성된 Swarm Cluster 에 `worker2` 노드가 Manager Node 로 추가 됐다.
	- 현재 상황은 `worker1` 노드가 Leader Manager Node 이고, `manager`, `worker2` 는 일반 Manager Node 이다.
- 깔끔한 테스트를 위해 기존 `swarmcluster-nginx` Service 는 지우고, 아래 명령어로 다시 실행 시킨다.

	```
	vagrant@worker1:~$ sudo docker service rm swarmcluster-nginx
	swarmcluster-nginx
	vagrant@worker1:~$ sudo docker service create --name swarmcluster-nginx --replicas 3 -p 80:80 nginx
	g69w9ymookk5s40blrhqw5ull
	overall progress: 3 out of 3 tasks
	1/3: running   [==================================================>]
	2/3: running   [==================================================>]
	3/3: running   [==================================================>]
	verify: Service converged
	vagrant@worker1:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
	e9h6t2gfa8wq        swarmcluster-nginx.1   nginx:latest        manager             Running             Running 28 seconds ago         
	oea47jjygiin        swarmcluster-nginx.2   nginx:latest        worker2             Running             Running 26 seconds ago         
	tljna3icc0yr        swarmcluster-nginx.3   nginx:latest        worker1             Running             Running 26 seconds ago         
	```  
	
	- 3개의 Manager Node 에 각각 Service Task 가 하나씩 실행 중이다.
	
### Leader Down 테스트
- Leader 인 `worker1` 호스트를 다운 시키고, `manager` 호스트에서 Swarm Cluster 의 Node 상태를 확인하면 

	```
	$ vagrant halt worker1
	==> worker1: Attempting graceful shutdown of VM...
	$ vagrant ssh manager
	.. 생략 ..
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Down                Active              Unreachable         19.03.5
	gnrqiccn1sexhomztqc3ic76p     worker2             Ready               Active              Reachable           19.03.5
	```  
	
	- Manager Node 가 짝수개 일 때와 달리, Swarm Cluster 가 가용상태이다.
	- 다운된 `worker1` Node 의 상태가 변경됐고, Leader 가 `manager` Node 로 변경 되었다.
- 실행 중이였던 Service 의 상태를 조회하면 아래와 같다.

	```
	vagrant@manager:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
	e9h6t2gfa8wq        swarmcluster-nginx.1       nginx:latest        manager             Running             Running 16 minutes ago     
	oea47jjygiin        swarmcluster-nginx.2       nginx:latest        worker2             Running             Running 16 minutes ago     
	rg4ska0myc15        swarmcluster-nginx.3       nginx:latest        worker2             Running             Running 15 minutes ago     
	tljna3icc0yr         \_ swarmcluster-nginx.3   nginx:latest        worker1             Shutdown            Running 16 minutes ago     
	```  
	
	- `worker1` 에서 실행 중이였던 Task 가 `worker2` 로 이동한 것과 구성된 Service 는 정상적인 상태인것을 확인 할 수 있다.
- 다운 시켰던 `worker1` 호스트를 다시 올리고, Swarm Cluster 의 Node 상태와 Service 상태를 확인하면 아래와 같다.

	```
	$ vagrant up worker1
	Bringing machine 'worker1' up with 'virtualbox' provider...
	==> worker1: Checking if box 'ubuntu/xenial64' version '20200129.0.0' is up to date...
	
	.. 생략 ..
	
	==> worker1: Machine already provisioned. Run `vagrant provision` or use the `--provision`
	==> worker1: flag to force provisioning. Provisioners marked to run always will still run.
	$ vagrant ssh manager
	.. 생략 ..
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active              Reachable           19.03.5
	gnrqiccn1sexhomztqc3ic76p     worker2             Ready               Active              Reachable           19.03.5
	vagrant@manager:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
	e9h6t2gfa8wq        swarmcluster-nginx.1       nginx:latest        manager             Running             Running 19 minutes ago     
	oea47jjygiin        swarmcluster-nginx.2       nginx:latest        worker2             Running             Running 19 minutes ago     
	rg4ska0myc15        swarmcluster-nginx.3       nginx:latest        worker2             Running             Running 19 minutes ago     
	tljna3icc0yr         \_ swarmcluster-nginx.3   nginx:latest        worker1             Shutdown            Shutdown 41 seconds ago    
	```  
	
	- `worker1` 노드가 정상적으로 Swarm Cluster 에 참여한 것을 확인 할 수 있다.
	- `worker1` 노드가 Swarm Cluster 에 참여 했지만, 기존에 `worker2` 로 옮겨졌던 Task 는 다시 `worker1` 으로 돌아오지 않는 것을 확인 할 수 있다.
- 다시 구성된 Swarm Cluster Node 들에 Service 의 균형을 맞추기 위해서는 다시 서비스를 생성하는 방법이 있고, 강제로 업데이트를 하는 방법이 있다.
- 서비스를 새로 생성하는 방법은 실제 서비스 상황이라면 서비스가 중단 될 수 있기 때문에 `docker service update --force` 를 사용해서 강제로 업데이트를 해서 균형을 맞춘다.

	```
	vagrant@manager:~$ sudo docker service update --force swarmcluster-nginx
	swarmcluster-nginx
	overall progress: 3 out of 3 tasks
	1/3: running   [==================================================>]
	2/3: running   [==================================================>]
	3/3: running   [==================================================>]
	verify: Service converged
	vagrant@manager:~$ sudo docker service ps swarmcluster-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
	y6wh13sy2fcd        swarmcluster-nginx.1       nginx:latest        manager             Running             Running 17 seconds ago     
	e9h6t2gfa8wq         \_ swarmcluster-nginx.1   nginx:latest        manager             Shutdown            Shutdown 18 seconds ago    
	rf0dw2ny5uvv        swarmcluster-nginx.2       nginx:latest        worker1             Running             Running 28 seconds ago     
	oea47jjygiin         \_ swarmcluster-nginx.2   nginx:latest        worker2             Shutdown            Shutdown 28 seconds ago    
	zmdgcic2ayn7        swarmcluster-nginx.3       nginx:latest        worker2             Running             Running 22 seconds ago     
	rg4ska0myc15         \_ swarmcluster-nginx.3   nginx:latest        worker2             Shutdown            Shutdown 23 seconds ago    
	tljna3icc0yr         \_ swarmcluster-nginx.3   nginx:latest        worker1             Shutdown            Shutdown 9 minutes ago     
	```  
	
	- 모든 Node 하나의 Task 씩 균형 맞게 할당 된 것을 확인 할 수 있다.

---
## Reference
[Administer and maintain a swarm of Docker Engines](https://docs.docker.com/engine/swarm/admin_guide/)  
[How nodes work](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/)  
	