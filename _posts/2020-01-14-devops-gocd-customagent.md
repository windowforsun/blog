--- 
layout: single
classes: wide
title: "[DevOps] GoCD Custom Agent"
header:
  overlay_image: /img/gocd-bg.png
excerpt: 'Agent 를 커스텀해서 다양한 환경에서 CD 툴을 사용해보자'
author: "window_for_sun"
header-style: text
categories :
  - DevOps
tags:
  - DevOps
  - GoCD
  - Agent
---  

	
## Custom Agent
- GoCD 의 기본 구성요소들과 개념, 간단한 사용법은 [여기]({{site.baseurl}}{% link _posts/2019-12-19-devops-gocd-introandinstall.md %}) 를 참고한다.
- 이름에도 나와있듯이 GoCD 의 핵심 개념은 CD 툴로 지속적인 배포 및 전달을 수행하는 툴인 만큼, 그 환경은 다양하게 구성될수 있다.
- 모든 환경에 대해서는 다루지 못하지만, 아래와 같은 환경에 대해서 Agent 를 구성해 본다.
	- Docker Image 를 빌드하고 저장소로 배포
	- Golang 을 빌드하고 저장소로 배포
- Docker Agent 는 GoCD 공식 Docker 저장소에 있는 이미지를 사용한다.
- Golang Agent 는 기본 Agent 이미지에서 Golang 관련 설정을 추가해서 사용한다.



<!--## Docker GoAgent -->
<!--- 서버의 환경이 Docker 로 구성돼있다면 Docker 에 맞는 GoAgent 를 통해 이미지를 빌드해야 한다.-->
<!--- 구성하는 GoCD, GoAgent 또한 모두 Docker 로 구성하는 예제로 진행한다.-->
<!--- Docker GoAgent 는 GoCD 공식 Docker 저장소에 있는 이미지를 사용한다.-->


### Docker 환경 구성하기
- GoCD, Agent 을 Docker 로 구성한 `docker-compose.yml` 파일은 아래와 같다.

	```yml
	version: '3.7'
	
	services:
	  gocd-server:
	    image: gocd/gocd-server:v19.11.0
	    ports:
	      - "8153:8153"
	      - "8154:8154"
	    volumes:
	      # GoCD 설정 저장
	      - ./godata/config/cruise-config.xml:/godata/config/cruise-config.xml
	    networks:
	      - gocd-net
	
	  # Golang 환경이 구성된 Agent
	  gocd-agent-go:
	    # 호스트 이름 지정
    	hostname: gocd-agent-go
	    build:
	      context: gocd-agent-go
	      dockerfile: Dockerfile
	    depends_on:
	      - gocd-server
	    environment:
	      GO_SERVER_URL: https://gocd-server:8154/go
	    networks:
	      - gocd-net
	    # GoCD Server 에 Healthcheck 를 수행하며 완전히 올라갈때 까지 기다린다.
	    healthcheck:
	      test: ["CMD", "curl", "-f", "http://gocd-server:8154/go/api/v1/health"]
	      interval: 10s
	      timeout: 10s
	      retries: 10
	
	  # Docker 환경이 구성된 Agent
	  gocd-agent-docker:
	  	# 호스트 이름 지정
    	hostname: gocd-agent-docker
	    build:
	      context: gocd-agent-docker
	      dockerfile: Dockerfile
	    depends_on:
	      - gocd-server
	    volumes:
	      - /var/run/docker.sock:/var/run/docker.sock
	    environment:
	      GO_SERVER_URL: https://gocd-server:8154/go
	    networks:
	      - gocd-net
	    # GoCD Server 에 Healthcheck 를 수행하며 완전히 올라갈때 까지 기다린다.
	    healthcheck:
	      test: ["CMD", "curl", "-f", "http://gocd-server:8154/go/api/v1/health"]
	      interval: 10s
	      timeout: 10s
	      retries: 10
	
	networks:
	  gocd-net:
	```  
	
- `gocd-agent-go/Dockerfile` 은 공식 Agent 이미지에 Golang 설치 및 빌드를 위한 환경을 설정 한다.

	```dockerfile
	FROM gocd/gocd-agent-centos-7:v19.11.0
	USER root
	# go install
	RUN curl -LO https://storage.googleapis.com/golang/go1.13.linux-amd64.tar.gz
	RUN tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
	ENV PATH $PATH:/usr/local/go/bin
	RUN export PATH=$PATH:/usr/local/go/bin
	RUN source ~/.bash_profile	
	
	# openssl
	RUN yum -y update
	RUN yum install -y openssl	
	
	
	# github ssh
	RUN mkdir /root/.ssh/
	ADD ./id_rsa /root/.ssh/id_rsa
	RUN chmod 600 /root/.ssh/id_rsa
	RUN touch /root/.ssh/known_hosts
	RUN ssh-keyscan github.com >> /root/.ssh/known_hosts
	```  
	
	- Golang Agent 에서는 SSH 를 이용해서 다른 레포에 접근해 빌드결과물을 올린다.
	- SSH 비밀키는 Golang Agent Dockerfile 과 같은 디렉토리에 위치시킨다.
	
- `gocd-agent-docker/Dockerfile` 은 공식 Docker Agent 이미지에 빌드를 위한 환경을 설정한다.

	```dockerfile
	FROM gocd/gocd-agent-docker-dind:v19.11.0
	USER root
	
	RUN sudo addgroup docker
	RUN sudo adduser go docker
	```  
	
### GoCD Server 환경 설정
- 구성된 환경은 2개의 Agent 가 있고, 각 Agent 들은 수행되는 환경이 다르기 때문에 구분을 짓는 작업이 필요하다.
	- Golang 빌드를 Docker Agent 에서 수행할수 없고, Docker 빌드를 Golang Agent 에서 수행할 수 없기 때문에, Pipeline 에서 각 Job 들이 실제로 수행되는 Agent 를 Pipleline 단위로 할당해 준다.
- 환경을 구분짓는 작업은 GoCD Server 에서 `Environments` 에서 진행한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-1.png)

- `Add Environment` 를 눌러 `docker`, `go` 이라는 새로운 환경을 추가한다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-2.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-3.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-4.png)
	
- 상단에 `AGENTS` 를 눌러 Agent 관리 페이지로 이동한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-5.png)
	
- GoCD Agent 로 등록된 `gocd-agent-go`, `gocd-agent-docker` 의 상태를 `ENABLE` 로 변경해 준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-6.png)

- `gocd-agent-go` Agent 를 `go` Environment 에 추가한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-7.png)
	
- `gocd-agent-docker` Agent 를 `docker` Environment 에 추가한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-8.png)
	
- 이 작업을 통해 `gocd-agent-go` Agent 는 `go` Environment 로 설정된 Pipeline 에서만 동작하고, `gocd-agent-docker` Agent 는 `docker` Environment 로 설정된 Pipeline 에서만 동작한다.
	
	
### Docker Agent 에서 Docker 빌드 Pipleline 구성하기
- Docker 로 빌드할 이미지는 Spring 프로젝트로 [여기]({{site.baseurl}}{% link _posts/2019-09-05-docker-practice-spring-boot-docker-jar.md %}) 와 동일한 프로젝트이다.
- 새로운 Pipeline 를 위해 먼저 Material 설정에서 프로젝트의 소스코드가 있는 Git 주소를 업력하고 `Test Connection` 으로 연결 테스트를 진행한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-9.png)

- `docker-spring-pipeline` 으로 Pipeline 이름을 설정한다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-10.png)
	
- `docker-spring-stage` 으로 Stage 이름을 설정한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-11.png)
	
- `docker-spring-job` 으로 Job 이름을 설정하고, `Save + Edit Full Config` 를 눌러준다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-12.png)
	
- `docker-spring-job` 으로 이동해서 `Add new task` 를 눌러 새로운 Task 를 추가한다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-13.png)

- 아래와 같이 Docker build 관련 명령어를 작성해 준다.
	- Command : `Docker`
	- Arguments : `build --no-cache --tag windowforsun/ex-web:latest --file docker/web/Dockerfile .`
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-14.png)
	
- `docker-spring-stage` 로 가서 `Jobs` 를 누르고 `Add new Job` 을 눌러준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-15.png)
	
- `docker-spring-push-job` 이라는 Docker 저장소에 빌드한 이미지를 푸시하는 Job 을 추가한다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-16.png)
	
- `docker-spring-push-job` 에서 아래와 같이 Docker Hub 에 로그인 하는 Task 를 추가해 준다.
	- Command : `bin/bash`
	- Arguments : `-c docker login -u=<DockerHub-ID> --password=<DockerHub-Password>`
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-17.png)

- `docker-spring-push-job` 에서 아래와 같이 Docker Hub 에 이미지를 푸시하는 Task 를 추가해 준다.
	- Command : `/bin/bash`
	- Arguments : `-c docker push windowforsun/ex-web:latest`
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-18.png)
	
- 구성한 `docker-spring-pipeline` 을 `docker` Environment 에 추가해 준다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-19.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-20.png)
	
- DashBoard 에서 중지돼있는 `docker-spring-pipeline` 을 풀어 실행시킨다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-21.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-22.png)
	
- 아래는 실행된 Pipeline 이 성공한 화면이다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-23.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-24.png)
	
- Docker Hub 에서 확인해보면 새로운 이미지가 올라간것을 확인 할 수 있다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-25.png)
	
### Golang Agent 에서 Golang 빌드 Pipline 구성하기
- Golang 빌드에서 사용할 프로젝트는 공개된 [golang/example](https://github.com/golang/example) 을 사용한다.
- 공개된 프로젝트 중에서 `hello` 를 빌드해서 사용하는데 그 소스코드는 아래와 같다.

	```
	package main

	import (
		"fmt"
	
		"github.com/golang/example/stringutil"
	)
	
	func main() {
		fmt.Println(stringutil.Reverse("!selpmaxe oG ,olleH"))
	}
	```  
	
- `golang-pipeline` 이라는 새로운 Pipeline 을 추가하는 과정은 아래와 같다.

	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-26.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-27.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-28.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-29.png)
	
	- Job 까지 설정이 끝났으면 `Save + Edit Full Conf` 를 눌러 준다.
	
- `golang-job` 에 아래 2개 Task 를 추가해 준다.
	- Golang 프로젝트 의존성관련
		- Command : `/bin/bash`
		- Arguments : `-c go get github.com/golang/example/hello`
	- GoLang 프로젝트 빌드
		- Command : `/bin/bash`
		- Arguments : `-c GOOS=linux go build -o myapp`
		- Working Directory : `hello`
		
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-30.png)
	
	- Golang 프로젝트 빌드 파일의 이름은 `myapp` 이고 `hello` 디렉토리에 위치한다.
	
- `golang-stage` 에서 Artifacts 설정으로 `hello/myapp` 을 설정해준다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-31.png)
	
- 빌드 파일을 빌드 저장소에 푸시하는 `golang-push-stage` 와 `golang-push-job` 을 추가하고. `golang-job` 의 프로젝트 빌드 파일을 가져온다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-32.png)
	
- `golang-push-job` 에 프로젝트 빌드파일을 저장하는 레포를 클론받고 파일을 설정 하는 Task 를 추가한다.
	- Command : `/bin/bash`
	- Arguments : `-c git clone git@github.com:windowforsun/golang_repo.git; rm -rf golang_repo/$GO_REVISION; mkdir golang_repo/$GO_REVISION; mv myapp golang_repo/$GO_REVISION; echo $GO_REVISION >> golang_repo/revision_log`
	- 저장소에 Golang 프로젝트의 Git Revision 으로 디렉토리를 만들고 그 안에 빌드 파일을 추가하기 위한 명령어들이다.
	- Git Revision 의 경우 GoCD 의 환경변수인 `$GO_REVISION` 으로 가져오는데 더 자세한 관련내용은 [여기](https://docs.gocd.org/current/faq/dev_use_current_revision_in_build.html) 에서 확인 가능하다.
- 클론 받은 빌드 저장소를 커밋하고 푸시하는 Task 를 아래와 같이 추가한다.
	- Command : ` /bin/bash`
	- Arguments : ` -c git config --global user.email "gocd@email.com"; git config --global user.name "GoCD"; git add --all; git commit -m "new build"; git push origin master`
	- Working Directory : `golang_repo`
- 최종적으로 설정된 `golang-push-job` 의 Task 는 아래와 같다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-33.png)
	
- 설정이 완료된 `golang-pipeline` 을 `go` Environment 에 추가한다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-34.png)
	
- `golang-pipeline` 의 일시정지를 풀어주고, 실행이 완료되면 Job 에서 클론 받았던 레포에 빌드 파일이 올라간 것을 확인 할 수 있다.
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-35.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-36.png)
	
	![그림 1]({{site.baseurl}}/img/devops/gocd-customagent-37.png)
	
---
## Reference