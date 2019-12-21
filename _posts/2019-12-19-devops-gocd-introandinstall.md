--- 
layout: single
classes: wide
title: "[DevOps] GoCD 설치 및 기본 사용"
header:
  overlay_image: /img/gocd-bg.png
excerpt: 'Spring '
author: "window_for_sun"
header-style: text
categories :
  - DevOps
tags:
  - DevOps
  - GoCD
  - CD
---  


## GoCD

![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-1.png)

- 이름에서 나와 있는 것과 같이 CD(Continuous Delivery / Continuous Deploy) 역할을 해주는 솔루션이다.
- GoCD 는 크게 `Server` 와 `Agent` 로 구성된다.
- `Server` 는 Web UI 인터페이스를 제공하고, `Agent` 에게 명령을 내려 모든것을 컨트롤하는 역할을 수행한다.
- `Agent` 는 `Server` 의 명령을 받아 실질적으로 명령어를 실행해 작업을 하는 역할을 수행한다.
- `Server` 는 `CD` 관련 역할을 수행하지 않고, `CD` 관련 역할은 `Agent` 가 수행한다.
- GoCD 의 작업흐름을 구성하는 요소는 `Pipeline`, `Stage`, `Job`, `Task` 가 있다.
- 현존하는 다양한 `CI/CD` 솔루션(Jenkins ..)들이 있지만 현재까지 GoCD 의 인지도는 그렇제 높지 않는 듯하다.
- 더 자세한 설명은 [여기](https://www.gocd.org/help/)에서 확인 가능하다.

## GoCD/Jenkins
- GoCD 공식 레퍼런스에서는 Jenkins 와 차이점을 아래와 같이 설명하고 있다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-2.png)
	
- 다른 사이트에서는 [다음](https://stackshare.io/stackups/go-cd-vs-jenkins)과 같이 비교하고 있다.

## GoCD 설치
- Docker 를 이용해서 GoCD 설치 및 환경을 구성한다.
- 위에서 언급했던 것과 같이 GoCD 는 `Server`와 `Agent` 루 구성되기 때문에 모두 구성하는 작업이 필요하다.
- GoCD 관련 Docker 이미지는 [여기](https://hub.docker.com/u/gocd)에서 확인 가능하다.
- 전체 구성이 작성된 `docker-compose.yml` 파일은 아래와 같다.

	```yaml
	version: '3.7'
	
	services:
	  gocd-server:
	    image: gocd/gocd-server:v19.11.0
	    ports:
	      # Web UI port
	      - "8153:8153"
	      # Agent port
	      - "8154:8154"
	    volumes:
	       # GoCD 설정 저장을 위해 마운트
	      - ./godata/config/cruise-config.xml:/godata/config/cruise-config.xml
	    networks:
	      - gocd-net
	
	  # 아무것도 설치 되지 않은 Agent
	  gocd-agent-centos:
	    image: gocd/gocd-agent-centos-7:v19.11.0
	    restart: always
	    depends_on:
	      - gocd-server
	    environment:
	      # GoCD Server URL
	      GO_SERVER_URL: https://gocd-server:8154/go
	    networks:
	      - gocd-net
	
	# server 와 agent 들이 같은 네트워크를 사용하기 위해 설정한다.
	networks:
	  gocd-net:
	```  
	
	- `Server` 와 `CentOS Agent` 는 공식 이미지를 그대로 사용한다.
	
- `docker-compose up --build` 명령어를 수행하면 작성된 구성 대로 이미지를 다운 받고 설치를 진행한다.

	```
	[root@localhost intro]# docker-compose up --build
	Creating network "intro_gocd-net" with the default driver
	Creating intro_gocd-server_1 ... done
	Creating intro_gocd-agent-centos_1 ... done
	Attaching to intro_gocd-server_1, intro_gocd-agent-centos_1
	gocd-server_1        | /docker-entrypoint.sh: Creating directories and symlinks to hold GoCD configuration, data, and
	 logs
	gocd-server_1        | $ mkdir -v -p /godata/artifacts
	gocd-server_1        | created directory: '/godata/artifacts'
	gocd-server_1        | $ ln -sv /godata/artifacts /go-working-dir/artifacts
	gocd-server_1        | '/go-working-dir/artifacts' -> '/godata/artifacts'

	생략 ..
	```  
	
	- `Agent` 는 `Server` 가 완전히 뜬다음 성공적으로 구동이 가능하다.
	
- `docker ps` 로 현재 구동중인 도커를 조회하면 아래와 같다.

	```
	[root@localhost intro]# docker ps
	CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS                PORTS                                            NAMES
	d213b4b649b7        gocd/gocd-agent-centos-7:v19.11.0          "/docker-entrypoint.…"   6 minutes ago       Up 6 minutes                                                           intro_gocd-agent-centos_1
	3b927d058503        gocd/gocd-server:v19.11.0                  "/docker-entrypoint.…"   6 minutes ago       Up 6 minutes          0.0.0.0:8153-8154->8153-8154/tcp                 intro_gocd-server_1
	```  
	
- `http://localhost:8153` 으로 접근하면 아래와 같은 페이지가 뜨고, 상단의 `AGENTS` 를 눌렀을 때 `CentOS Agent` 가 표시되면 모두 성공이다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-3.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-4.png)
	
- `AGENTS` 에서 `CentOS Agent` 를 활성화 시켜준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-4_1.png)
	
	
## Pipeline 이란

![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-5.png)

- `Pipline` 은 GoCD 에서 하나의 작업 흐름을 뜻한다.
- `Pipeline` 은 한개 이상의 `Stage` 로 구성된다.
- `Material` 은 `Pipelin` 의 Trigger 역할로 하나의 작업 흐름을 시작시키는 역할을 수행한다.
- `Material` 은 소스코드 형상관리 툴인 Git, SVN 등 을 통해 설정할 수 있고 다른 `Pipline` 으로 도 설정 할 수 있다.
- `Pipeline` 은 하나 이상의 `Material` 을 가질 수 있다.



## Stages, Jobs, Tasks

![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-5_1.png)

### Task
- 하나의 Job 에서 무언가를 설정 및 수행하는 명령어 라인 하나 하나를 뜻한다.
- 플러그인도 사용 가능하다.

### Job
- Job 은 Task 의 집합이다.
- Job 은 포함된 Task 를 순서대로 실행하는 역할을 수행한다.
- Stage 에 포함된 Job 들은 비동기로 실행된다.
- 하나의 Job 은 하나의 Agent 에서 담당해서 수행한다.
- 기본설정에서는 Task 중간에 실패가 발생하면, Job 은 실패한다.

### Stage
- Stage 는 Job 의 집합이다.
- Job 부분에서 언급한 것과 같이, 동일한 Stage 에 포함된 여러 Job 은 순서대로 실행되지 않고, 비동기로 실행된다.
- 순서대로 실행되는 Job 의 Task 들과 비동기로 실행되는 Stage 의 Job 들을 어떻게 잘 설계하느냐가 GoCD 에서는 중요하다.
- Stage 에 포함된 여러 Job 들은 각기다른 Agent 에서 실행 가능하다.

## Pipeline 만들기
- 간단한 Pipeline 을 만들기 위해 GoCD 에서 예제용으로 제공하는 Git Repository 를 사용한다.
	- https://github.com/gocd-contrib/getting-started-repo.git
- 해당 Git 주소를 Material 로 설정하고 `Test Connection` 을 눌러 연결 여부를 확인한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-6.png)

- Pipeline 이름을 작성해 준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-7.png)

- Stage 이름을 작성해 준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-8.png)

- Job 이름과 `./build` 커멘드를 실행하는 Task 를 작성해 준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-9.png)
	
- 하단의 `Save + Run This Pipeline` 을 눌러 Pipeline 을 실행 시킨다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-10.png)
	
- Pipeline 이 성공하면 아래와 같다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-11.png)
	
- 성공한 화면에서 초록색 바를 누르게 되면 실행된 결과에 대해 확인 할 수 있다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-12.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-13.png)
	

## Pipeline Chaining 하기
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	








































	

	
	
	































































	
---
## Reference
[INTRODUCTION TO GoCD](https://www.gocd.org/getting-started/part-1/)