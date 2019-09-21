--- 
layout: single
classes: wide
title: "[Docker 개념] 미 Docker Compose"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Concept
    - Docker
    - DockerCompose
---  

## Environment
- CentOS 7

## Docker Compose Define
- 여러개의 Docker Container 를 한번에 실행 할 수 있는 도구이다.
- Docker Compose 의 레퍼런스는 [여기](https://docs.docker.com/compose/)에서 확인 가능하다.

## Docker Compose Install
- 아래 커멘트를 통해 docker-compose 의 안전한 버전을 다운로드 받는다.

	```
	# curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
	```  
	
- 다운받은 파일에 실행권한을 추가해 준다.

	```
	# chmod +x /usr/local/bin/docker-compose
	```  
	
- 심볼릭 링크 경로를 걸어준다.

	```
	# ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
	```  
	
- 설치를 확인 한다.

	```
	# docker-compose --version
    docker-compose version 1.24.1, build 1110ad01
	```  
	
- Windows 환경에서는 Docker Desktop 을 설치했다면 docker-compose 는 함께 설치된다.

## Docker Compose Uninstall
- `curl` 로 다운 받았던 파일을 삭제해 준다.

	```
	# rm /usr/local/bin/docker-compose
	```  

## docker-compose.yml
- Docker Compose 는 `docker-compose.yml` 이라는 파일을 기반으로 실행된다.
- `docker-compose.yml` 파일에 형식에 맞춰 자신이 구성하고 싶은 환경을 작성해 주면 된다.
- `docker-compose.yml` 에는 `version` 이 있는데, 버전에 따라 지원하는 기능과 구조가 다르다. [자세히](https://docs.docker.com/compose/compose-file/)
- `docker-compose.yml` 에 대한 더더더더욱 자세한 설명은 [여기](https://docs.docker.com/compose/compose-file/)를 참고한다.

## docker-compose.yml Architecture
- `docker-compose.yml` 의 구조에 대한 설명은 현재(2019-09-05)기준 최신인 3.7로 한다.
- 파일의 확장자는 `yml`, `yaml` 모두 가능한다.
- 크게 아래 3가지로 구성된다.
	- `services`
	- `networks`
	- `volumes`

## services
- `services` 는 `docker` 명령어를 사용해서 옵션 파라미터로 전달 했던 요소들을 각 서비스의 컨테이너에 맞게 정의할 수 있다.
- 하나의 Docker Container 에 하나 이상의 Docker 이미지를 정의해 환경을 구성 할 수 있다.
- `Dockerfile` 에서 하나의 컨테이너에 각각 정의하던 `CMD`, `EXPOSE`, `VOLUMES` 등을 `docker-compose.yml` 파일에서는 좀 더 효율정으로 정의할 수 있다.
- 쉘에 등록되어 있는 환경변수를 `docker-compose.yml` 파일에 사용 가능하다.

## networks
- ``



### build
- Docker 서비스를 빌드

```
version: "3.7"
services:

  redis:
    image: redis:alpine
    ports:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]

  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - "5000:80"
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - "5001:80"
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes:
  db-data:
```  












	

---
## Reference
[]()  
