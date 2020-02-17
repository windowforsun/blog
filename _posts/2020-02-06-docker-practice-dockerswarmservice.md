--- 
layout: single
classes: wide
title: "[Docker 실습] Docker Swarm Service 사용하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Swarm Service 를 올리고 관리해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Swarm
  - Service
---  

## Docker Swarm Service 생성하기
- Docker Swarm 의 논리적 작업 단위인 Service 를 사용하는 법에 대해 알아본다.
- Service 를 구성할때 사용하는 Docker Image 는 Nginx 이미지를 사용한다.


- Service 를 생성하는 방법은 `docker service create <image>` 로 가능하다.

	```bash
	vagrant@manager:~$ sudo docker service create --name swarmservice-nginx -p 80:80 nginx
	5ha1lb0qoph0ip5dx27ljp1l2
	overall progress: 1 out of 1 tasks
	1/1: running   [==================================================>]
	verify: Service converged
	```  
	
	- `--name` 옵션으로 Service 에 `swarmservice-nginx` 라는 이름을 부여 해주었다.
	- `-p` 옵션으로 `swarmservice-nginx` 의 Container 내부 80 포트와 Host 80 포트를 포워딩 시켰다.
- 현재 Swarm 에서 구성된 Service 의 조회는 `docker service ls` 로 가능하다.

	```bash
	vagrant@manager:~$ sudo docker service ls
	ID                  NAME                 MODE                REPLICAS            IMAGE               PORTS
	5ha1lb0qoph0        swarmservice-nginx   replicated          1/1                 nginx:latest
	```  
	
- `docker container ls` 혹은 `docker ps` 를 통해 현재 Host 에서 구동중인 컨테이너를 조회하면 아래와 같다.

	```bash
	vagrant@manager:~$ sudo docker ps
	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS	     NAMES
	56899f20e13b        nginx:latest        "nginx -g 'daemon of…"   32 seconds ago      Up 31 seconds       80/tcp	 swarmservice-nginx.1.drwgvd16xeq9xenzls6tp8ay9
	```  
	
	- Container 의 이름인 `NAMES` 항목은 `<Service이름>.<TaskSlot번호>.<TaskID>` 구성처럼  `swarmservice-nginx.1.drwgvd16xeq9xenzls6tp8ay9` 로 나와있다. 

- Service 하나에서 구동중인 Task 의 확인은 `docker service ps <서비스이름>` 이다.

	```bash
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE	        ERROR               PORTS
	drwgvd16xeq9        swarmservice-nginx.1   nginx:latest        manager             Running             Running 18 minutes ago
	```  
	
- `curl` 명령어를 통해 띄워진 Nginx 서비스에 요청을 보내면 아래처럼 Nginx 기본 페이지 결과를 응답해 준다.

	```bash
	vagrant@manager:~$ curl http://localhost
	<!DOCTYPE html>
	<html>
	<head>
	<title>Welcome to nginx!</title>
	<style>
	    body {
	        width: 35em;
	        margin: 0 auto;
	        font-family: Tahoma, Verdana, Arial, sans-serif;
	    }
	</style>
	</head>
	<body>
	<h1>Welcome to nginx!</h1>
	<p>If you see this page, the nginx web server is successfully installed and
	working. Further configuration is required.</p>
	
	<p>For online documentation and support please refer to
	<a href="http://nginx.org/">nginx.org</a>.<br/>
	Commercial support is available at
	<a href="http://nginx.com/">nginx.com</a>.</p>
	
	<p><em>Thank you for using nginx.</em></p>
	</body>
	</html>
	```  
	
## Docker Swarm Service 삭제하기
- 현재 생성된(구동 중인) Service 를 삭제(중지)하는 방법은 `docker service rm <서비스이름또는아이디>` 로 가능하다.

	```bash
	vagrant@manager:~$ sudo docker service ls
	ID                  NAME                 MODE                REPLICAS            IMAGE               PORTS
	zkesjxt37vo8        swarmservice-nginx   replicated          1/1                 nginx:latest        *:80->80/tcp
	
	# docker service rm zkesj 아이디로도 삭제 가능하다.
	vagrant@manager:~$ sudo docker service rm swarmservice-nginx
	swarmservice-nginx	
	$ sudo docker service ls
	ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
	```  
	
## Docker Swarm Service 사용하기
- 위에서는 아주 기본적으로 Service 를 조작에 대한 명령어만 알아 보았다.
- Service 로 테스트 및 운영을 하며 필요한 몇가지 명령어에 대해 더 다뤄본다.

### Service Inspection
- Docker 관련 구성 요소들에는 대부분 `inspect` 을 통해 구성 요소의 자세한 정보를 볼 수 있는 것처럼, Service 도 동일하다.

	```bash
	vagrant@manager:~$ sudo docker service inspect --pretty swarmservice-nginx
	
	ID:             s2paxecw7ylfvvvoseweohp45
	Name:           swarmservice-nginx
	Service Mode:   Replicated
	 Replicas:      1
	Placement:
	UpdateConfig:
	 Parallelism:   1
	 On failure:    pause
	 Monitoring Period: 5s
	 Max failure ratio: 0
	 Update order:      stop-first
	RollbackConfig:
	 Parallelism:   1
	 On failure:    pause
	 Monitoring Period: 5s
	 Max failure ratio: 0
	 Rollback order:    stop-first
	ContainerSpec:
	 Image:         nginx:latest@sha256:ad5552c786f128e389a0263104ae39f3d3c7895579d45ae716f528185b36bc6f
	 Init:          false
	Resources:
	Endpoint Mode:  vip
	Ports:
	 PublishedPort = 80
	  Protocol = tcp
	  TargetPort = 80
	  PublishMode = ingress
	```  

	- `--pretty` 옵션으로 보다 보기 편하게 출력하도록 했다.
		- 옵션을 주지 않으면 `json` 형식으로 정보가 출력된다.
		
### Service Scale 조절
- 하나의 Service 는 다수의 Task(Container) 로 구성될 수 있는데, 몇개로 구성 될지를 정하는 부분이 `scale` 이다.

	```bash
	.. scale 조정 전 확인 ..
	vagrant@manager:~$ sudo docker service ls
	ID                  NAME                 MODE                REPLICAS            IMAGE               PORTS
	s2paxecw7ylf        swarmservice-nginx   replicated          1/1                 nginx:latest        *:80->80/tcp
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE	        ERROR               PORTS
	drwgvd16xeq9        swarmservice-nginx.1   nginx:latest        manager             Running             Running 20 minutes ago
	
	.. scale 5개로 확장 ..
	vagrant@manager:~$ sudo docker service scale swarmservice-nginx=5
	swarmservice-nginx scaled to 5
	overall progress: 5 out of 5 tasks
	1/5: running   [==================================================>]
	2/5: running   [==================================================>]
	3/5: running   [==================================================>]
	4/5: running   [==================================================>]
	5/5: running   [==================================================>]
	verify: Service converged
	
	.. scale 확장 후 확인 ..
	vagrant@manager:~$ sudo docker service ls
	ID                  NAME                 MODE                REPLICAS            IMAGE               PORTS
	s2paxecw7ylf        swarmservice-nginx   replicated          5/5                 nginx:latest        *:80->80/tcp
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE	        ERROR               PORTS
	drwgvd16xeq9        swarmservice-nginx.1   nginx:latest        manager             Running             Running 21 minutes ago
	xdwf8vkk2n5y        swarmservice-nginx.2   nginx:latest        manager             Running             Running 24 seconds ago
	oufohc0ebegb        swarmservice-nginx.3   nginx:latest        manager             Running             Running 24 seconds ago
	utlh5anyimww        swarmservice-nginx.4   nginx:latest        manager             Running             Running 24 seconds ago
	5sozd0ljs7fg        swarmservice-nginx.5   nginx:latest        manager             Running             Running 24 seconds ago
	```  
	
	- Scale 를 확장 시킬수도 있고, 축소 시킬 수도 있다.
		
### Service Log
- 한 Service 에 구성된 Task(Container) 에서 발생하는 로그를 확인하는 방법은 `logs` 를 사용하면 된다.

	```bash
	vagrant@manager:~$ sudo docker service logs swarmservice-nginx
	swarmservice-nginx.1.drwgvd16xeq9@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:05 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.1.drwgvd16xeq9@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:16 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.2.xdwf8vkk2n5y@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:16 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.2.xdwf8vkk2n5y@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:17 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.5.5sozd0ljs7fg@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:15 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.5.5sozd0ljs7fg@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:18 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.3.oufohc0ebegb@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:04 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.3.oufohc0ebegb@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:17 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.4.utlh5anyimww@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:16 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	swarmservice-nginx.4.utlh5anyimww@manager    | 10.0.0.2 - - [06/Feb/2020:07:44:18 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
	```  
	
### Service Update
- 현재 구동 중인 Service 를 업데이트 하기 위해서는 `docker service update` 를 사용하면 된다.

	```bash
	.. 현재 Service Task 확인 ..
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE        ERROR               PORTS
	dtyilrkv1q69        swarmservice-nginx.1   nginx:latest        manager             Running             Running 38 seconds ago
	qf4zbbu89r3n        swarmservice-nginx.2   nginx:latest        manager             Running             Running 15 seconds ago
	hxpjqiompd5z        swarmservice-nginx.3   nginx:latest        manager             Running             Running 15 seconds ago
	ueuiz1scfwpc        swarmservice-nginx.4   nginx:latest        manager             Running             Running 15 seconds ago
	msyyxjr0jatc        swarmservice-nginx.5   nginx:latest        manager             Running             Running 15 seconds ago
	
	.. Service 이미지 업데이트 ..
	vagrant@manager:~$ sudo docker service update --image nginx:1.16 swarmservice-nginx
	swarmservice-nginx
	overall progress: 5 out of 5 tasks
	1/5: running   [==================================================>]
	2/5: running   [==================================================>]
	3/5: running   [==================================================>]
	4/5: running   [==================================================>]
	5/5: running   [==================================================>]
	verify: Service converged
	
	.. 업데이트 후 Service Task 확인 ..
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE	             ERROR               PORTS
	zoii7pjn8omw        swarmservice-nginx.1       nginx:1.16          manager             Running             Running 29 seconds ago
	dtyilrkv1q69         \_ swarmservice-nginx.1   nginx:latest        manager             Shutdown            Shutdown 30 seconds ago
	cidmpxippq5l        swarmservice-nginx.2       nginx:1.16          manager             Running             Running 15 seconds ago
	qf4zbbu89r3n         \_ swarmservice-nginx.2   nginx:latest        manager             Shutdown            Shutdown 16 seconds ago
	o7hegt4gttyi        swarmservice-nginx.3       nginx:1.16          manager             Running             Running 24 seconds ago
	hxpjqiompd5z         \_ swarmservice-nginx.3   nginx:latest        manager             Shutdown            Shutdown 25 seconds ago
	ktvbxklispwn        swarmservice-nginx.4       nginx:1.16          manager             Running             Running 34 seconds ago
	ueuiz1scfwpc         \_ swarmservice-nginx.4   nginx:latest        manager             Shutdown            Shutdown 34 seconds ago
	6g982qiy3eg6        swarmservice-nginx.5       nginx:1.16          manager             Running             Running 20 seconds ago
	msyyxjr0jatc         \_ swarmservice-nginx.5   nginx:latest        manager             Shutdown            Shutdown 21 seconds ago
	```  
	
	- `--image` 옵션을 사용해서 업데이트 할 이미지를 명시할 수 있다.
- 현재 Service 의 Inspector 를 확인해 보면 아래와 같다.

	```bash
	vagrant@manager:~$ sudo docker service inspect --pretty swarmservice-nginx
	
	ID:             jkw9imb7k23om9yjmkllzgbvk
	Name:           swarmservice-nginx
	Service Mode:   Replicated
	 Replicas:      5
	UpdateStatus:
	 State:         completed
	 Started:       4 minutes ago
	 Completed:     3 minutes ago
	 Message:       update completed
	Placement:
	UpdateConfig:                   # Update 시 적용되는 설정
	 Parallelism:   1               # Update 는 Task 1개씩 수행
	 On failure:    pause
	 Monitoring Period: 5s          # Update 간격은 5초
	 Max failure ratio: 0
	 Update order:      stop-first
	RollbackConfig:
	 Parallelism:   1
	 On failure:    pause
	 Monitoring Period: 5s
	 Max failure ratio: 0
	 Rollback order:    stop-first
	 
	.. 생략 ..
	```  
	
	- `UpdateConfig` 부분을 보면 업데이트를 수생할 때 적용되는 정보를 볼 수 있는데, 기본설정으로 1개의 Task 씩 5초 간격으로 업데이트를 수행하고 있다.
- 기본 설정 값의 경우 2개의 Task 가 구동중이고 이를 업데이트 할때 1번 Task 를 업데이트를 위해서 내리고, 5초후에 2번 Task 를 업데이트를 위해서 내리는데 2번 Task 가 내려가고 5초 전까지 1번 Task 가 뜨지 않으면 안정적인 서비스가 불가하다.
- 현재 구성된 Service 에서 안정적으로 서비스를 제공을 위해 Rolling Update 관련 설정을 아래와 같은 옵션 값으로 수정할 수 있다.
	- `--update-delay` : Update 를 수행할 때 시간 간격
	- `--update-failure-action` : Update 가 실패했을 때 동작
	- `--update-max-failure-ratio` : Update 실패로 간주하는 시간
	- `--update-monitor` : Update 시에 실패인지 확인하는 시간 간격
	- `--update-order` : Update 순서 (start-first, stop-first)
	- `--update-parallelism` : Update 를 한번에 몇개의 Task 를 수행할지
- 아래와 같이 옵션을 주고 Service 를 수정 후 Inspector 의 출력이다.
	
	```bash
	.. 생성 할때도 동일하게 옵션을 부여하면 된다 ..
	vagrant@manager:~$ sudo docker service create --name swarmservice-nginx -p 80:80 --replicas=5 --update-delay 10s --update-parallelism=2 nginx
	
	vagrant@manager:~$ sudo docker service update --update-delay=10s --update-parallelism=2 swarmservice-nginx
	swarmservice-nginx
	overall progress: 5 out of 5 tasks
	1/5: running   [==================================================>]
	2/5: running   [==================================================>]
	3/5: running   [==================================================>]
	4/5: running   [==================================================>]
	5/5: running   [==================================================>]
	verify: Service converged	
	
	$ sudo docker service inspect --pretty swarmservice-nginx

	ID:             jkw9imb7k23om9yjmkllzgbvk
	Name:           swarmservice-nginx
	Service Mode:   Replicated
	 Replicas:      5
	Placement:
	UpdateConfig:
	 Parallelism:   2
	 Delay:         10s
	 On failure:    pause
	 Monitoring Period: 5s
	 Max failure ratio: 0
	 Update order:      stop-first
	RollbackConfig:
	 Parallelism:   1
	 On failure:    pause
	 Monitoring Period: 5s
	 Max failure ratio: 0
	 Rollback order:    stop-first
	 
	.. 생략 ..
	```  

	- 현재 Service 의 설정에서 업데이트를 수행하면 2개의 Task 를 10초 간격으로 업데이터를 수행하게 된다.
	

### Service Rollback
- Swarm Service 는 이전에 올라갔던 Task(Container)로 Rollback 하는 명령어를 제공한다.
- Rollback 의 동작은 Update 와 매우 유사하다.
- `docker service rollback <서비스이름>` 으로 수행 할수 있다.

	```bash
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE	             ERROR               PORTS
	zoii7pjn8omw        swarmservice-nginx.1       nginx:1.16          manager             Running             Running 27 minutes ago
	dtyilrkv1q69         \_ swarmservice-nginx.1   nginx:latest        manager             Shutdown            Shutdown 27 minutes ago
	cidmpxippq5l        swarmservice-nginx.2       nginx:1.16          manager             Running             Running 27 minutes ago
	qf4zbbu89r3n         \_ swarmservice-nginx.2   nginx:latest        manager             Shutdown            Shutdown 27 minutes ago
	o7hegt4gttyi        swarmservice-nginx.3       nginx:1.16          manager             Running             Running 27 minutes ago
	hxpjqiompd5z         \_ swarmservice-nginx.3   nginx:latest        manager             Shutdown            Shutdown 27 minutes ago
	ktvbxklispwn        swarmservice-nginx.4       nginx:1.16          manager             Running             Running 27 minutes ago
	ueuiz1scfwpc         \_ swarmservice-nginx.4   nginx:latest        manager             Shutdown            Shutdown 27 minutes ago
	6g982qiy3eg6        swarmservice-nginx.5       nginx:1.16          manager             Running             Running 27 minutes ago
	msyyxjr0jatc         \_ swarmservice-nginx.5   nginx:latest        manager             Shutdown            Shutdown 27 minutes ago
	
	vagrant@manager:~$ sudo docker service rollback swarmservice-nginx
	swarmservice-nginx
	rollback: manually requested rollback
	overall progress: rolling back update: 5 out of 5 tasks
	1/5: running   [>                                                  ]
	2/5: running   [>                                                  ]
	3/5: running   [>                                                  ]
	4/5: running   [>                                                  ]
	5/5: running   [>                                                  ]
	verify: Service converged
	
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE	                 ERROR               PORTS
	b1pnsiezijro        swarmservice-nginx.1       nginx:latest        manager             Running             Running 18 seconds ago
	f0w1kt4ups7i         \_ swarmservice-nginx.1   nginx:1.16          manager             Shutdown            Shutdown 19 seconds ago
	ygqidwhxt9k4         \_ swarmservice-nginx.1   nginx:latest        manager             Shutdown            Shutdown about a minute ago
	zoii7pjn8omw         \_ swarmservice-nginx.1   nginx:1.16          manager             Shutdown            Shutdown 2 minutes ago
	dtyilrkv1q69         \_ swarmservice-nginx.1   nginx:latest        manager             Shutdown            Shutdown 33 minutes ago
	sh9socgz4w8b        swarmservice-nginx.2       nginx:latest        manager             Running             Running 22 seconds ago
	oksll84y0p1o         \_ swarmservice-nginx.2   nginx:1.16          manager             Shutdown            Shutdown 23 seconds ago
	dw0kqjbly540         \_ swarmservice-nginx.2   nginx:latest        manager             Shutdown            Shutdown about a minute ago
	cidmpxippq5l         \_ swarmservice-nginx.2   nginx:1.16          manager             Shutdown            Shutdown 2 minutes ago
	qf4zbbu89r3n         \_ swarmservice-nginx.2   nginx:latest        manager             Shutdown            Shutdown 33 minutes ago
	qzsl77m6vlhz        swarmservice-nginx.3       nginx:latest        manager             Running             Running 36 seconds ago
	owlpn7l6gp8k         \_ swarmservice-nginx.3   nginx:1.16          manager             Shutdown            Shutdown 37 seconds ago
	mhigpafnpq7r         \_ swarmservice-nginx.3   nginx:latest        manager             Shutdown            Shutdown about a minute ago
	o7hegt4gttyi         \_ swarmservice-nginx.3   nginx:1.16          manager             Shutdown            Shutdown about a minute ago
	hxpjqiompd5z         \_ swarmservice-nginx.3   nginx:latest        manager             Shutdown            Shutdown 33 minutes ago
	tr50xl0bnllr        swarmservice-nginx.4       nginx:latest        manager             Running             Running 31 seconds ago
	sgf3p1gs4ivw         \_ swarmservice-nginx.4   nginx:1.16          manager             Shutdown            Shutdown 32 seconds ago
	vw3ujcnymtto         \_ swarmservice-nginx.4   nginx:latest        manager             Shutdown            Shutdown 58 seconds ago
	ktvbxklispwn         \_ swarmservice-nginx.4   nginx:1.16          manager             Shutdown            Shutdown 2 minutes ago
	ueuiz1scfwpc         \_ swarmservice-nginx.4   nginx:latest        manager             Shutdown            Shutdown 33 minutes ago
	kc713hjihlok        swarmservice-nginx.5       nginx:latest        manager             Running             Running 27 seconds ago
	uhqnn8h1cjj5         \_ swarmservice-nginx.5   nginx:1.16          manager             Shutdown            Shutdown 28 seconds ago
	z3i5mypg1fs1         \_ swarmservice-nginx.5   nginx:latest        manager             Shutdown            Shutdown about a minute ago
	6g982qiy3eg6         \_ swarmservice-nginx.5   nginx:1.16          manager             Shutdown            Shutdown 2 minutes ago
	msyyxjr0jatc         \_ swarmservice-nginx.5   nginx:latest        manager             Shutdown            Shutdown 33 minutes ago
	```  
	
	- Rollback 전 Nginx:1.16 이였던 버전이 Rollback 후 Nginx:latest 로 변경된 것을 확인 할 수 있다.
	- 각 Task 에는 현재 구동중인 Container 외에 이전에 구동 됐었던 컨테이너의 기록이 남겨져 있는 것을 확인 할 수 있다.

### Service Task 의 기록
- `docker info` 를 통해 현재 Docker Engine 에 설정된 Task 에 몇개의 기록을 저장할지에 대한 정보를 확인 할 수 있다.

	```bash
	vagrant@manager:~$ sudo docker info
	
	.. 생략 ..
	
	 Swarm: active
	  NodeID: z2h43qxvvr5u0poie80zjc5kg
	  Is Manager: true
	  ClusterID: cma0zqa900pmd2mddabi5uu7e
	  Managers: 1
	  Nodes: 1
	  Default Address Pool: 10.0.0.0/8
	  SubnetSize: 24
	  Data Path Port: 4789
	  Orchestration:
	   Task History Retention Limit: 5
	   
	.. 생략 ..
	```  
	
	- `Task History Retention Limit` 부분을 보면 개수가 5개로 제한된 것을 확인 할 수 있다.(구동 중인 컨테이너 포함)
- 커스텀하게 변경이 필요하다면 `docker swarm update --task-history-limit` 를 통해 변경 가능하다.
	
	```bash
	vagrant@manager:~$ sudo docker swarm update --task-history-limit 3
	Swarm updated.
	vagrant@manager:~$ docker info
	
	.. 생략 ..
	
	 Swarm: active
	  NodeID: z2h43qxvvr5u0poie80zjc5kg
	  Is Manager: true
	  ClusterID: cma0zqa900pmd2mddabi5uu7e
	  Managers: 1
	  Nodes: 1
	  Default Address Pool: 10.0.0.0/8
	  SubnetSize: 24
	  Data Path Port: 4789
	  Orchestration:
	   Task History Retention Limit: 3
	   
	.. 생략 ..
	```  
	
### Node 업데이트 하기
- Docker Swarm 에서 Node 는 하나의 Host 를 의미한다.
- Host 머신의 사양이나 네트워크 관련 설정을 하기위해서는 Swarm Cluster 에 잠시 제외 시키는 작업이 필요한데, 이러한 동작을 Drain 을 통해 할 수 있다.
- Worker Node 1 을 Swarm Cluster 에 추가하고 서비스를 띄우면 아래와 같다.

	```bash
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active                                  19.03.5
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                   IMAGE               NODE                DESIRED STATE       CURRENT STATE	        ERROR               PORTS
	myo364utv0pm        swarmservice-nginx.1   nginx:latest        manager             Running             Running 25 seconds ago
	xc42dpnjlryc        swarmservice-nginx.2   nginx:latest        manager             Running             Running 27 seconds ago
	pi2dmhwjp5ih        swarmservice-nginx.3   nginx:latest        manager             Running             Running 26 seconds ago
	hpmy2jad8mzc        swarmservice-nginx.4   nginx:latest        worker1             Running             Running 27 seconds ago
	94k3prz3stuf        swarmservice-nginx.5   nginx:latest        worker1             Running             Running 27 seconds ago
	```  
	
	- Swarm Cluster 에 구성된 모든 Node 가 `Active` 상태 인것과, `swarmservice-nginx` 서비스가 두 Node 에서 모두 실행 중인 것을 확인 할 수 있다.
- `worker1` Node 를 수정하기 위해 `drain` 명령어를 통해 상태 전환을 한다.
	- `drain` 명령어는 Manager Node 에서 수행한다.
	
	```bash
	vagrant@manager:~$ sudo docker node update --availability drain worker1
	worker1
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Drain                                   19.03.5
	```  
	
	- `worker1` Node 의 상태가 Drain 으로 변경된 것을 확인 할 수 있다.
- Swarm Cluster 에서 실행 중인던 `swarmservice-nginx` 를 조회하면 아래와 같다.
	
	```bash
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE	                 ERROR               PORTS
	myo364utv0pm        swarmservice-nginx.1       nginx:latest        manager             Running             Running 6 minutes ago
	xc42dpnjlryc        swarmservice-nginx.2       nginx:latest        manager             Running             Running 6 minutes ago
	pi2dmhwjp5ih        swarmservice-nginx.3       nginx:latest        manager             Running             Running 6 minutes ago
	zz1vhhl1ded5        swarmservice-nginx.4       nginx:latest        manager             Running             Running about a minute ago
	hpmy2jad8mzc         \_ swarmservice-nginx.4   nginx:latest        worker1             Shutdown            Shutdown about a minute ago
	hyr64yle9pg4        swarmservice-nginx.5       nginx:latest        manager             Running             Running about a minute ago
	94k3prz3stuf         \_ swarmservice-nginx.5   nginx:latest        worker1             Shutdown            Shutdown about a minute ago
	```  
	
	- `worker1` Node 에서 실행 중이던 Task 들이 중지 된 상태인 것을 확인 할 수 있다.
- `worker1` Node 의 변경 작업이 완료후 Drain 상태에서 다시 활성화를 시키는 명령어는 `active` 이다.

	```bash
	vagrant@manager:~$ sudo docker node update --availability active worker1
	worker1
	vagrant@manager:~$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	z2h43qxvvr5u0poie80zjc5kg *   manager             Ready               Active              Leader              19.03.5
	5x09cav0mappy087xxjm4rcb1     worker1             Ready               Active                                  19.03.5
	vagrant@manager:~$ sudo docker service ps swarmservice-nginx
	ID                  NAME                       IMAGE               NODE                DESIRED STATE       CURRENT STATE	            ERROR               PORTS
	myo364utv0pm        swarmservice-nginx.1       nginx:latest        manager             Running             Running 8 minutes ago
	xc42dpnjlryc        swarmservice-nginx.2       nginx:latest        manager             Running             Running 8 minutes ago
	pi2dmhwjp5ih        swarmservice-nginx.3       nginx:latest        manager             Running             Running 8 minutes ago
	zz1vhhl1ded5        swarmservice-nginx.4       nginx:latest        manager             Running             Running 4 minutes ago
	hpmy2jad8mzc         \_ swarmservice-nginx.4   nginx:latest        worker1             Shutdown            Shutdown 4 minutes ago
	hyr64yle9pg4        swarmservice-nginx.5       nginx:latest        manager             Running             Running 4 minutes ago
	94k3prz3stuf         \_ swarmservice-nginx.5   nginx:latest        worker1             Shutdown            Shutdown 4 minutes ago
	```  
	
	- `worker1` Node 가 Drain 상태에서 다시 Active 상태로 전환되었고, `worker1` Node 에서 중지 됐었던 Task 들도 모두 실행 된 것을 확인 할 수 있다.

---
## Reference
[[Docker 기본(7/8)] Docker Swarm의 구조와 Service 배포하기](https://medium.com/dtevangelist/docker-%EA%B8%B0%EB%B3%B8-7-8-docker-swarm%EC%9D%98-%EA%B5%AC%EC%A1%B0%EC%99%80-service-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0-1d5c05967b0d)
	