--- 
layout: single
classes: wide
title: "[Docker 실습] Vagrant 로 Docker Swarm 테스트 환경 구성하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Vagrant 를 사용해서 Docker Swarm 환경을 구성해 테스트 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Vagrant
  - VirtualBox
---  

## 환경
- Host : Windows 10

## Docker Swarm 환경
- `Docker Swarm` 은 단일 호스트 환경보다는 가진 의미처럼 많은 호스트들이 떼를 이루고 있는(분산) 구조에 적합하다.
- 이런 `Docker Swarm` 을 사용하면서 익숙해 지기 위해서는 단일 환경에서 하는 테스트 만으론 부족할 수 있다.
- 로컬 머신에서 분산환경을 구축하기 위해 `Vagrant` 와 `VirtualBox` 를 사용하고, `Docker Swarm` 을 테스트해 본다.
- `Vagrant` 가 `VirtualBox` 를 사용해서 가상 머신을 띄우기 때문에 두가지 모두 설치 해야 한다.
	- `Vagrant` 와 `VirtualBox` 의 관계 및 더 자세한 설명과 구성은 따로 포스트 한다.

## VirtualBox 설치
- `VirtualBox` 는 호스트 머신에 가상머신을 띄워 Guest OS 를 구동 시킬 수 있도록 도와주는 툴이다.
- `VirtualBox` 를 원활하게 사용하기 위해서는 `Hyper-V` 설정을 OFF 시켜야 한다.
	- `제어판` -> `프로그램 및 기능` -> `Windows 기능 켜기/끄기` -> `Hyper-V` 체크 해제
- `BIOS` 설정에서 `VT-X` 설정을 ON 시킨다.
- [VirtualBox](https://www.virtualbox.org/) 에서 운영체제에 맞는 설치 프로그램으로 설치 해준다.


## Vagrant 설치
- `Vagrant` 는 간소화된 가상머신 관리를 해주는 솔루션으로, VM 툴에서 설정을 만져가며 하나씩 구성해야 했던 것들을 스크립트로 간단하게 구성할 수도 있고 개발환경의 공유에도 유용하다.
- [Vagrant](https://www.vagrantup.com/downloads.html) 에서 설치 파일을 통해 설치를 할 수 있다.

## Vagrant 구성하기
- 지금부터 모든 명령어는 Windows 명령 프롬프트 또는 Windows PowerShell, Git Bash 에서 실행한다.
- `Vagrant` 는 스크립트를 통해 VM 환경을 구성할 수 있기 때문에, `Docker Swarm` 환경 구성에 사용할 스크립트는 아래와 같이 다운 받아 사용한다.

	```
	$ curl https://gist.githubusercontent.com/code-machina/1994fb4c8546a680d58b61a5cdbc1fe2/raw/fd29669ccc3e9424df713f24d1ed99dfa91bdb83/Vagrantfile -o Vagrantfile
	```  
	
- 구성할 환경에 대한 모든 내용이 있는 `Vagrantfile` 내용은 아래와 같다.

	```
	BOX_IMAGE = "ubuntu/xenial64"
	WORKER_COUNT = 2 # U can specify the number of workers.
	
	MANAGER_IP_ADDRESS = "192.168.100.10" # Also, can chagne the subnet network
	
	Vagrant.configure("2") do |config|
	 config.vm.box = BOX_IMAGE
	
	  config.vm.define "manager" do |subconfig|
	    subconfig.vm.box = BOX_IMAGE
	    subconfig.vm.hostname = "manager"
	    subconfig.vm.network :private_network, ip: MANAGER_IP_ADDRESS
	    subconfig.vm.network "forwarded_port", guest: 3000, host: 3500
	  end
	
	  (1..WORKER_COUNT).each do |worker_count|
	    config.vm.define "worker#{worker_count}" do |subconfig|
	      subconfig.vm.box = BOX_IMAGE
	      subconfig.vm.hostname = "worker#{worker_count}"
	      subconfig.vm.network :private_network, ip: "192.168.100.#{10 + worker_count}"
	      subconfig.vm.network "forwarded_port", guest: 3000, host: (3500 + worker_count)
	    end
	  end
	
	  config.vm.provider "virtualbox" do |vb|
	    # Customize the amount of memory on the VM:
	    vb.memory = "1024" # customize memory is required for pc
	  end
	
	  config.vm.provision "shell", inline: <<-SHELL
	     echo "provisioning"
	     # to set-up proper image running docker swarm, you should change some dependencies.....
	     apt-get install \
	             apt-transport-https \
	       ca-certificates \
	       curl \
	       software-properties-common
	
	    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	
	    apt-key fingerprint 0EBFCD88
	    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
	
	    apt-get update
	    apt-get install -y docker-ce
	    usermod -aG docker ubuntu
	
	  SHELL
	end
	```  
	
	- `ubunbu/xenial64` 이미지를 사용
	- `WORKER_COUNT` 의 값에 따라 `swarm woker node` 수 조정
	- 생성되는 VM 은 `manager` 와 `WORKER_COUNT` 수에 따라 `worker1`, `worker2` .. 생성된다.
	- 각 VM 에 할당되는 아이피는 `manager` 부터 `192.168.100.10` 으로 시작해서 `192.168.100.11` `192.168.100.12` 순으로 증가한다.
	- 호스트와 VM 간의 포트포워딩 설정도 있다.
	- `docker-ce` 를 설치를 통한 프로비저닝 설정
- `WORKER_COUNT` 르 수를 2로 하고 아래 명령어로 `Vagrant` 환경을 구동 시킨다.

	```
	$ vagrant up
	```  
	
- `manager`, `worker1`, `worker2` VM 이 생성된 상태이고 아래의 명령어로 각 VM 에 접속 할 수 있다.

	```
	$ vagrant ssh manager
	$ vagrant ssh worker1
	$ vagrant ssh worker2
	```  
	
## Swarm 구성 및 테스트
- `manager` 에 접속해서 아래 명령어로 swarm 을 설정 한다.

	```
	$ vagrant ssh manager
	$ sudo docker swarm init --advertise-addr 192.168.100.10
	Swarm initialized: current node (k0yfbr3mt8mocyvmmv5blcxbg) is now a manager.
	
	To add a worker to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-5i7ru7xlrhpcbtmvctndv0t4o26tahge1s8z1c38cduo4puvmn-6zm5vctdbir21xikv8byrknph 192.
	168.100.10:2377
	
	To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
	```  
	
- `worker1`, `worker2` 에 접속해 `manager` 에서 설정한 swarm 에 참여 한다.

	```
	$ vagrant ssh worker1
	$ sudo docker swarm join --token SWMTKN-1-5i7ru7xlrhpcbtmvctndv0t4o26tahge1s8z1c38cduo4puvmn-6zm5vctdbir21xikv8byrknph 192.168.100.10:2377
	This node joined a swarm as a worker.
	$ exit
	$ vagrant ssh worker2
	$ sudo docker swarm join --token SWMTKN-1-5i7ru7xlrhpcbtmvctndv0t4o26tahge1s8z1c38cduo4puvmn-6zm5vctdbir21xikv8byrknph 192.168.100.10:2377
	This node joined a swarm as a worker.
	```  
	
- `manager` 에서 노드를 확인해 보면 아래와 같다.

	```
	$ sudo docker node ls
	ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
	k0yfbr3mt8mocyvmmv5blcxbg *   manager             Ready               Active              Leader              19.03.5
	e1ru2j7lo8j87jnd3r154vxgp     worker1             Ready               Active                                  19.03.5
	effihs1fgwh6yvru8yn03h3is     worker2             Ready               Active                                  19.03.5
	```  
	
- `manager` 에서 아래 명령어를 사용해서 테스트로 사용할 서비스를 구성한다.

	```
	$ docker service create --replicas 10 -p 80:80 nginx
	4xmzw4aim3danncgoo9lvv4lw
	overall progress: 10 out of 10 tasks
	1/10: running   [==================================================>]
	2/10: running   [==================================================>]
	3/10: running   [==================================================>]
	4/10: running   [==================================================>]
	5/10: running   [==================================================>]
	6/10: running   [==================================================>]
	7/10: running   [==================================================>]
	8/10: running   [==================================================>]
	9/10: running   [==================================================>]
	10/10: running   [==================================================>]
	verify: Service converged
	```  
	
- 각 VM 에서 실행 중인 컨테이너를 확인하면 아래와 같다.

	```
	$ vagrant ssh manager
	$ sudo docker ps
	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS     NAMES
	ee15d83a8a99        nginx:latest        "nginx -g 'daemon of…"   51 seconds ago      Up 48 seconds       80/tcp      blissful_williamson.8.p79zqdg4f15a88m7qm1whh7rb
	28955a473ff4        nginx:latest        "nginx -g 'daemon of…"   51 seconds ago      Up 49 seconds       80/tcp      blissful_williamson.10.swyeeqkctohs5ysr5cizmy11l
	b6a53c088fc8        nginx:latest        "nginx -g 'daemon of…"   51 seconds ago      Up 47 seconds       80/tcp      blissful_williamson.2.3t10auewh3mq89ftmrb0856gu
	
	$ vagrant ssh worker1
	$ sudo docker ps
	CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS      NAMES
	cb01ca6c1c2c        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.4.4uic4b0jjbk8a71sqv7foje1b
	f4c5805689bd        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.1.l15gczsbatxp4d06dk4thwjxm
	6deaba623d83        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.9.okdm4in46nnm0mt29lvjtlhsh
	3e7bc58459b7        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.6.o061gsyx4xo8b93orana3w93g
	
	$ vagrant ssh worker2
	$ sudo docker ps
	CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS      NAMES
	fb82bc7d7600        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.7.dooe2lhhqotcfjohg2awl4gm8
	f89bf36abfe9        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.5.wkdhdms45266zew92au39nqhi
	ee651ec96387        nginx:latest        "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp       blissful_williamson.3.h533j2gjx8rn2ly28l374ggn7
	```  
	
- `http://192.168.100.10` 으로 요청을 보내면서 `manager` 에서 서비스 로그를 확인하면 아래와 같다.

	```
	$ vagrant ssh manager
	$ sudo docker service ls
	ID                  NAME                  MODE                REPLICAS            IMAGE               PORTS
	4xmzw4aim3da        blissful_williamson   replicated          10/10               nginx:latest        *:80->80/tcp 
	
	$ sudo docker service logs 4x -f
	blissful_williamson.2.3t10auewh3mq@manager     | 10.0.0.2 - - [03/Feb/2020:07:10:33 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.1.l15gczsbatxp@worker1     | 10.0.0.2 - - [03/Feb/2020:07:10:40 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.3.h533j2gjx8rn@worker2     | 10.0.0.2 - - [03/Feb/2020:07:10:51 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.5.wkdhdms45266@worker2     | 10.0.0.2 - - [03/Feb/2020:07:11:03 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.6.o061gsyx4xo8@worker1     | 10.0.0.2 - - [03/Feb/2020:07:11:12 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.7.dooe2lhhqotc@worker2     | 10.0.0.2 - - [03/Feb/2020:07:11:20 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.8.p79zqdg4f15a@manager     | 10.0.0.2 - - [03/Feb/2020:07:11:42 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.9.okdm4in46nnm@worker1     | 10.0.0.2 - - [03/Feb/2020:07:11:55 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.4.4uic4b0jjbk8@worker1     | 10.0.0.2 - - [03/Feb/2020:07:12:05 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	blissful_williamson.10.swyeeqkctohs@manager     | 10.0.0.2 - - [03/Feb/2020:07:12:30 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36" "-"
	```  

---
## Reference

	