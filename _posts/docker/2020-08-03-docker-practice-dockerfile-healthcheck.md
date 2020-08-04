--- 
layout: single
classes: wide
title: "[Docker 실습] Healthcheck"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 에서 제공하는 Healthcheck 를 알아보고, Dockerfile과 Docker Compose 에서 사용해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
  - Docker Compose
  - Healthcheck
toc: true
use_math: true
---  

## Healthcheck
`Healthcheck` 를 사용하면 사용자의 정의에 따라 컨테이너의 상태를 체크할 수 있다. 
컨테이너 시작 후 애플리케이션 초기화 과정에서 시간이 소요되는 상황이나, 
특정 예외 상황을 감지하고 관리할 수 있다.  

`Docker Swarm` 에서는 기본으로 `ingress network` 를 바탕으로 서비스에 포함되는 컨테이너들을 클러스터링 및 로드밸런싱 한다. 
여기서 `Healthcheck` 가 성공한 컨테이너만 `ingress network` 에 포함되어 실제 서비스의 대상이 된다. 
`Healthcheck` 가 실패한 컨테이너는 클러스터와 로드밸런싱에서 제외 된다. 

컨테이너에 `Healthcheck` 가 설정된 상황에서 실행되면 처음에는 `starting` 상태가 된다. 
그리고 정의된 `Healthcheck` 를 수행해서 성공하게 되면 상태는 `healthy` 로 변경되고 서비스가 가능한 상태이다. 
만약 설정된 횟수만큼 실패하게 되면, 상태는 `unhealthy` 로 변경되고 이는 서비스가 불가능한 상태이다.  

만약 여러개 `Healthcheck` 가 정의된 상태인 경우, 
마지막에 선언된 `Healthcheck` 만 적용된다.  

`Healthcheck` 에서 사용할 수 있는 옵션은 아래와 같다. 

옵션|설명|기본값
---|---|---
interval|수행 간격|30s
timeout|타임 아웃 시간|30s
start-period|초기화 시간|0s
retries|실패 최대 횟수|3

`Healthcheck` 수행은 `interval` 간격만큼 컨테이너가 실행 중인 동안 계속해서 실행 된다. 
그리고 `Healthcheck` 동작은 `timeout` 의 임계치를 넘게되면 실패로 판별된다. 
그러므로 `start-period` 와 같은 `Healthcheck` 를 수행하기 전 초기화 시간을 애플리케이션 상황에 맞춰 설정 할 필요가 있다. 
또한 `retries` 의 횟수만큼 실패하게 되면 `Healthcheck` 상태는 `unhealthy` 가 되고 컨테이너는 재시작 된다.  

만약 `Healthcheck` 를 `Shell Script` 로 사용할 때, 
종료 코드에 따른 `Healthcheck` 의 동작은 아래와 같다. 
- 0 : `success` 의 의미로 컨테이너의 애플리케이션이 사용 가능 상태임을 의미한다.  
- 1 : `unhealthy` 인 상태로 컨테이너 혹은 애플리케이션 동작에 문제가 있는 상태이다. 
- 2 : `reserved` 는 `Healthcheck` 에서 예약해둔 종료 코드로 사용해서는 안된다. 

## Healthcheck 사용하기
`Healthcheck` 를 수행하는 동작은 사용자가 정의한 쉘 스크립트를 사용하거나, 
애플리케이션에 요청을 보내는 방식을 주로 사용한다. 

### Dockerfile 
`Dockerfile` 에서 `Healthcheck` 의 사용예시는 아래와 같다. 

```dockerfile
HEALTHCHECK --interval=<주기> --timeout=<실행타임아웃시간> --start-period=<시작주기> --regies=<실패반복횟수> \
    CMD  <명령>
```  

### Docker Compose
사용하는 이미지에 `Healthcheck` 가 미리 정의되지 않은 경우, 
`Docker Compose` 에서 서비스 단위로 `Healthcheck` 를 정의할 수 있다. 
그 예시는 아래와 같다. 

```yaml
services:
    service-1:
      healthcheck:
        test: ["CMD", "<명령>"]
        invertial: 주기
        timeout: 실행타임아웃시간
        retries: 실패반복횟수
```  

### 테스트
`Healthcheck` 테스트를 위한 디렉토리, 파일 구성은 아래와 같다.

```bash
.
├── docker-compose.yaml
├── hc-compose
│   └── Dockerfile
├── hc-dockerfile
│   ├── curl
│   │   └── Dockerfile
│   └── shell
│       └── Dockerfile
└── healthcheck.sh
```  

전체 구성을 결정하는 `docker-compose.yaml` 파일 내용은 아래와 같고, 
모든 서비스에서 사용하는 이미지는 `Nginx` 를 기반으로 한다. 

```yaml
# ./docker-compose.yaml
version: '3.7'

services:
  compose-curl:
    image: hc-compose:latest
    deploy:
      replicas: 2
    ports:
      - 8100:80
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/healthcheck.html"]
      interval: 10s
      timeout: 2s
      retries: 3
      start_period: 30s

  compose-shell:
    image: hc-compose:latest
    deploy:
      replicas: 2
    ports:
      - 8101:80
    healthcheck:
      test: ["CMD", "/tmp/healthcheck.sh"]
      interval: 1s
      timeout: 2s
      retries: 3

  dockerfile-curl:
    image: hc-dockerfile-curl:latest
    deploy:
      replicas: 2
    ports:
      - 8102:80

  dockerfile-shell:
    image: hc-dockerfile-shell:latest
    deploy:
      replicas: 2
    ports:
      - 8103:80

  override-curl:
    image: hc-dockerfile-curl:latest
    deploy:
      replicas: 2
    ports:
      - 8104:80
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/fail.html"]
      interval: 1s
      timeout: 2s
      retries: 3

  override-shell:
    image: hc-dockerfile-shell:latest
    deploy:
      replicas: 2
    ports:
      - 8105:80
    healthcheck:
      test: ["CMD", "/tmp/fail.sh"]
      interval: 1s
      timeout: 2s
      retries: 3
```  

총 6개의 서비스를 구성해서 테스트를 진행한다. 
- `compose-curl` : `Docker Compose` 에서 `curl` 을 사용한 `Healthcheck` 를 정의한다.
- `compose-shell` : `Docker Compose` 에서 `Shell Script` 를 사용한 `Healthcheck` 를 정의한다.
- `dockerfile-curl` : `Dockerfile` 내에서 `curl` 을 사용한 `Healthcehck` 를 정의한다. 
- `dockerfile-shell` : `Dockerfile` 내에서 `Shell Script` 를 사용한 `Healthcheck` 를 정의한다. 
- `ovrride-curl` : `Dockerfile` 내에서 정의한 `Healthcheck` 를 `Docker Compose` 에서 `curl` 로 재정의 한다. 
- `override-shell` : `Dockerfile` 내에서 정의한 `Healthcheck` 를 `Docker Compose` 에서 `Shell Script` 로 재정의 한다.

먼저 `Healthcheck` 를 수행할 때 사용하는 `healthcheck.sh` 의 내용은 아래와 같이 정상적인 코드인 `0` 을 종료 코드로 사용한다.

```bash
#!/bin/sh
# ./healthcheck.sh

exit 0
```  

`Docker Compose` 에서 `Healthcheck` 를 정의할 때 사용하는 `hc-compose` 이미지의 `Dockerfile` 은 아래와 같다. 

```dockerfile
# ./hc-compose/Dockerfile

FROM nginx:latest

RUN apt-get update
RUN apt-get install -y vim

COPY healthcheck.sh /tmp/healthcheck.sh
RUN echo "ok~!" >> /usr/share/nginx/html/healthcheck.html
```  

- 에디팅을 위해 `vim` 을 설치한다. 
- `Shell Script` 를 이용한 `Healthcheck` 에서 사용할 `healthcheck.sh` 를 컨테이너로 복사한다. 
- `curl` 을 이용한 `Healthcheck` 에서 사용할 `healthcheck.html` 파일을 생성한다. 

`hc-compose` 이미지의 빌드는 아래 명령으로 가능하다. 

```bash
$ docker build --tag hc-compose -f hc-compose/Dockerfile .

.. 생략 ..

Successfully built 1157f8a595cb
Successfully tagged hc-compose:latest
```  

``

`Dockerfile` 에서 `curl` 방식의 `Healthcheck` 를 정의하는 `hc-dockerfile-curl` 이미지의 `Dockerfile` 은 아래와 같다. 

```dockerfile
# ./hc-dockerfile/curl/Dockerfile
FROM nginx:latest

RUN echo "ok~!" >> /usr/share/nginx/html/healthcheck.html

HEALTHCHECK --interval=10s --timeout=2s --retries=3 --start-period=60s \
    CMD curl -f http://localhost/healthcheck.html
```  

- `curl` 방식으로 `Healthcheck` 수행 할때 사용할 `healthcheck.html` 파일을 생성한다. 
- `HEALTHCHECK` 명령을 사용해서 `healthcheck.html` 파일에 `HTTP GET` 요청을 보내는 방식으로 `Healthceck` 를 설정한다. 


`Dockerfile` 에서 `Shell Script` 방식의 `Healthcheck` 를 정의하는 `hc-dockerfile-shell` 이미지의 `Dockerfile` 은 아래와 같다. 

```dockerfile
# ./hc-dockerfile/shell/Dockerfile
FROM nginx:latest

RUN apt-get update
RUN apt-get install -y vim
COPY healthcheck.sh /tmp/healthcheck.sh

HEALTHCHECK --interval=1s --timeout=2s --retries=3 \
    CMD /tmp/healthcheck.sh
```  

- 에디팅을 위해 `vim` 을 설치한다. 
- `Healthcheck` 에서 사용할 `healthcheck.sh` 파일을 컨테이너로 복사한다. 
- `HEALTHCHECK` 명령을 사용해서 `/tmp/healthcheck.sh` 파일을 실행해 `Healthcheck` 를 수행하도록 설정한다. 

여기까지가 구성한 모든 파일의 내용과 관련 설명이다. 
`docker-compose.yaml` 파일과, `Dockerfile` 에 설정된 `Healthcheck` 의 내용을 서비스를 기준으로 다시 정리하면 아래와 같다. 

서비스|외부포트|방식|interval|timeout|start-period|retries|동작 순서
---|---|---|---|---|---|---|---
compose-curl|8100|curl|10s|2s|30s|3|2
compose-shell|8101|shell|1s|2s|0s|3|1
dockerfile-curl|8102|curl|10s|2s|60s|3|3
dockerfile-shell|8103|shell|1s|2s|0s|3|1
override-curl|8104|curl|2s|1s|0s|3|x
override-shell|8105|shell|1s|2s|0s|3|x

`compose-curl`, `dockerfile-curl` 서비스에서는 고의로 `start-period` 에 `30s`, `60s` 의 값을 설정했다. 
이런 설정을 바탕으로 `compose-shell` 과 `dockerfile-shell` 은 서비스가 올라가는 즉시 실행이 가능하지만, 
`compose-curl` 은 30초 대기 후 실행되고 `dockerfile-curl` 은 60초 대기 후 실행 된다. 
그리고 `override-shell`, `override-curl` 은 실패하도록 설정했기 때문에 서비스는 실행되지 않는다.    

테스트는 `Docker Swarm` 에서 진행한다. 
`docker stack deploy -c docker-compose.yaml hc` 명령으로 서비스를 클러스터에 등록한다. 
그리고 계속해서 `docker service ls` 명령으로 계속해서 서비스를 확인하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ docker service ls
ID                  NAME                  MODE                REPLICAS            IMAGE                        PORTS
oekozh0rwkn6        hc_compose-curl       replicated          0/2                 hc-compose:latest            *:8100->80/tcp
zddmav8cvsn2        hc_compose-shell      replicated          2/2                 hc-compose:latest            *:8101->80/tcp
f7jtto86xnnw        hc_dockerfile-curl    replicated          0/2                 hc-dockerfile-curl:latest    *:8102->80/tcp
qfj6w0rp6qty        hc_dockerfile-shell   replicated          2/2                 hc-dockerfile-shell:latest   *:8103->80/tcp
hvjl5ehtdamr        hc_override-curl      replicated          0/2                 hc-dockerfile-curl:latest    *:8104->80/tcp
kmr7ppj7fb6m        hc_override-shell     replicated          0/2                 hc-dockerfile-shell:latest   *:8105->80/tcp
$ docker service ls
ID                  NAME                  MODE                REPLICAS            IMAGE                        PORTS
oekozh0rwkn6        hc_compose-curl       replicated          2/2                 hc-compose:latest            *:8100->80/tcp
zddmav8cvsn2        hc_compose-shell      replicated          2/2                 hc-compose:latest            *:8101->80/tcp
f7jtto86xnnw        hc_dockerfile-curl    replicated          0/2                 hc-dockerfile-curl:latest    *:8102->80/tcp
qfj6w0rp6qty        hc_dockerfile-shell   replicated          2/2                 hc-dockerfile-shell:latest   *:8103->80/tcp
hvjl5ehtdamr        hc_override-curl      replicated          0/2                 hc-dockerfile-curl:latest    *:8104->80/tcp
kmr7ppj7fb6m        hc_override-shell     replicated          0/2                 hc-dockerfile-shell:latest   *:8105->80/tcp
$ docker service ls
ID                  NAME                  MODE                REPLICAS            IMAGE                        PORTS
oekozh0rwkn6        hc_compose-curl       replicated          2/2                 hc-compose:latest            *:8100->80/tcp
zddmav8cvsn2        hc_compose-shell      replicated          2/2                 hc-compose:latest            *:8101->80/tcp
f7jtto86xnnw        hc_dockerfile-curl    replicated          2/2                 hc-dockerfile-curl:latest    *:8102->80/tcp
qfj6w0rp6qty        hc_dockerfile-shell   replicated          2/2                 hc-dockerfile-shell:latest   *:8103->80/tcp
hvjl5ehtdamr        hc_override-curl      replicated          0/2                 hc-dockerfile-curl:latest    *:8104->80/tcp
kmr7ppj7fb6m        hc_override-shell     replicated          0/2                 hc-dockerfile-shell:latest   *:8105->80/tcp
```  

예상 했던 것과 동일하게 먼저 `compose-shell`, `dockerfile-shell` 서비스가 실행되고 나서 `compose-curl` 이 실행되고 마지막으로 `dockerfile-curl` 이 실행되는 것을 확인 할 수 있다. 
그리고 `ovrride-curl` 과 `override-shell` 서비스는 실행되지 않는다.  


### 특정 컨테이너 Healthcheck 실패 테스트

이번에는 `compose-curl` 서비스 중 컨테이너 하나만 `Healthcheck` 를 실패 시킨다음, 
실제로 로드밸런싱에서 제외되면서 서비스는 계속해서 가능한지 확인해 본다. 
테스트를 위해서 `Bash` 창을 여러개 사용하면 보다 편리하게 진행할 수 있다.  

먼저 `compose-curl` 서비스 로그를 확인하면 아래와 같다.   

```bash
$ docker service logs hc_compose-curl
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:20:23 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:20:24 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:20:34 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:20:34 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
```  

1, 2번 컨테이너에 계속해서 `Healthcheck` 요청이 들어오는 것을 확인 할 수 있다. 
여기서 `curl localhost:8100` 명령으로 요쳥을 2번 이상 보내면 
아래와 같이 1, 2번에 번갈아가며 `/` 경로로 요청이 들어가고 처리되는 것을 확인 할 수 있다. 

```bash
$ curl localhost:8100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
$ curl localhost:8100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
$ curl localhost:8100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
$ curl localhost:8100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
$ curl localhost:8100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..

$ docker service logs hc_compose-curl
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:22:28 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:22:29 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:22:29 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:22:30 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:22:30 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
```  

1번 컨테이너에 접속해 `Healthcheck` 가 실패하도록 `heathcheck.html` 파일을 지우기 위해 먼저 1번 컨테이너의 아이디를 아래와 같이 확인한다. 

```bash
$ docker ps | grep hc_compose-curl
7642b1ad9c56        hc-compose:latest                            "/docker-entrypoint.…"   24 minutes ago      Up 24 minutes (healthy)             80/tcp                              hc_compose-curl.1.wdms8kae6wg4p28r4t8r7j29w
cfa59afe8cbb        hc-compose:latest                            "/docker-entrypoint.…"   24 minutes ago      Up 24 minutes (healthy)            80/tcp                              hc_compose-curl.2.3wotmh3le20t846u0wfq1v88p
```  

그리고 `docker exec -it <컨테이너 아이디> /bin/bash` 명령으로 컨테이너에 접속하고, 
`rm -f /usr/share/nginx/html/healthcheck.html` 명령으로 `Healthcheck` 를 수행하는 파일을 지운다. 

```bash
$ docker exec -it 7642b1ad9c56 /bin/bash
root@7642b1ad9c56:/# rm -f /usr/share/nginx/html/healthcheck.html
root@7642b1ad9c56:/# exit
exit
```  

컨테이너에서 빠져와서 `curl localhost:8100` 으로 요청을 계속해서 보내고, 
로그를 확인해보면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ docker service logs hc_compose-curl
.. 첫 번째 Healthcheck ..
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:48 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 2020/08/04 03:29:31 [error] 28#28: *70 open() "/usr/share/nginx/html/healthcheck.html" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /healthcheck.html HTTP/1.1", host: "localhost"

.. 두 번째 Healthcheck ..
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:29:31 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:36 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:38 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:29:41 +0000] "GET /healthcheck.html HTTP/1.1" 404 153 "-" "curl/7.64.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 2020/08/04 03:29:41 [error] 28#28: *73 open() "/usr/share/nginx/html/healthcheck.html" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /healthcheck.html HTTP/1.1", host: "localhost"

.. 세 번째 Healtcheck ..
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:29:41 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:48 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 2020/08/04 03:29:51 [error] 28#28: *76 open() "/usr/share/nginx/html/healthcheck.html" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /healthcheck.html HTTP/1.1", host: "localhost"

.. 1 번 컨테이너 로드밸런싱에서 제외 ..
hc_compose-curl.1.wdms8kae6wg4@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:29:51 +0000] "GET /healthcheck.html HTTP/1.1" 404 153 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:29:51 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:52 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:53 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:29:58 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68

.. 1번 컨테이너가 자동으로 재시작 되고 다시 로드밸런싱에 포함 ..
hc_compose-curl.1.x1r0tzt3uy4a@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:30:10 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 127.0.0.1 - - [04/Aug/2020:03:30:11 +0000] "GET /healthcheck.html HTTP/1.1" 200 5 "-" "curl/7.64.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:30:11 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.x1r0tzt3uy4a@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:30:12 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.2.3wotmh3le20t@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:30:12 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
hc_compose-curl.1.x1r0tzt3uy4a@docker-desktop    | 10.0.0.2 - - [04/Aug/2020:03:30:13 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
```  

`compose-curl` 의 `Healthcheck` 에서 `retries` 옵션은 3으로 설정돼있다. 
그래서 1번 컨테이너의 `Healthcheck` 가 3번 실패가 되기 전까지는 `healthy` 상태이다가, 그 이후에 `unhealthy` 상태로 변경되고 로드밸런싱에서 제외 된다.  

다시 `compose-curl` 에서 실행되는 컨테이너를 조회하면 아래와 같다. 

```bash
ocker ps | grep hc_compose-curl
c680161e0009        hc-compose:latest                            "/docker-entrypoint.…"   About an hour ago   Up About an hour (healthy)         80/tcp                              hc_compose-curl.1.x1r0tzt3uy4apnludd1ae1loo
cfa59afe8cbb        hc-compose:latest                            "/docker-entrypoint.…"   About an hour ago   Up About an hour (healthy)         80/tcp                              hc_compose-curl.2.3wotmh3le20t846u0wfq1v88p
```  

이전 1번에 해당하는 컨테이너 아이디는 `7642b1ad9c56` 이고, 새로 올라온 컨테이너 아이디는 `c680161e0009` 이다. 
`docker inspect --format '{{.State.Health.Status}}' <컨테이너 아이디>` 명령으로 각각 상태 값을 조회하면 아래와 같다. 

```bash
$ docker inspect --format '{{.State.Health.Status}}' 7642b1ad9c56
unhealthy
$ docker inspect --format '{{.State.Health.Status}}' c680161e0009
healthy
```  

이렇게 `Healthcheck` 기능을 사용하면 애플리케이션에 맞는 방법으로 상태를 체크할 수 있다. 
그리고 이는 자동으로 클러스터에서 관리를 해준다. 
사용자가 정의한 방법과 설정 값으로 관리하기 때문에, 
몇번에 테스트와 검증 과정을 거치며 애플리케이션과 서버 구성에 알맞는 `Healthcheck` 설정을 해줄 필요가 있다. 

---
## Reference
[Docker Compose - healthcheck](https://docs.docker.com/compose/compose-file/#healthcheck)  
[Dockerfile - healthcheck](https://docs.docker.com/engine/reference/builder/#healthcheck)  