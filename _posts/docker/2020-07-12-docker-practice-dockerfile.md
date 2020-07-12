--- 
layout: single
classes: wide
title: "[Docker 실습] Dockerfile(도커파일) 구성과 작성법"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
---  

## Dockerfile
`Dockerfile` 은 `Docker` 이미지를 설정하는 파일을 의미한다. 
그리고 이후에는 `Dockerfile` 로 구성한 `Docker Image` 를 컨테이너로 실행 시킬 수 있다. 
`Dockerfile` 은 기본적으로 미리 정의된 `<명령>` 과 `<매개 변수>` 를 사용해서, 아래와 같은 구조를 갖는다. 

```dockerfile
<명령> <매개 변수>
```  

여기서 명령은 대소문자 모두 가능하지만, 주로 대문자로 작성한다.  

`Dockerfile` 을 작성하게 되면 `<명령> <매개 변수>` 가 계속해서 나열되는 구조로 작성되는데, 
이를 위에서 부터 순서대로 수행한다. 그리고 `Dockerfile` 의 시작은 `FROM` 명령으로 시작해야 한다. 


### FROM 
`FROM` 은 구성하는 이미지의 기반 이미지를 설정하는 명령이다. 
`Docker Image` 구성은 기반 이미지를 사용하기 때문에, 
`Dockerfile` 에서 필수적이고 첫 시작이 되는 명령이다. 

`FROM` 명령은 아래와 같은 구성으로 사용할 수 있다. 

```dockerfile
FROM <이미지 이름>
FROM <이미지 이름>:<태그>

FROM ubuntu
FROM ubuntu:latest
FROM ubuntu:14.04
```  

`FROM` 구문에 설정된 이미지는 로컬에 있으면 로컬 이미지를 사용하고, 
로컬에 없으면 `Docker Engine` 에 설정된 저장소에서 이미지를 검색해 받아와서 사용한다.  

그리고 `Dockerfile` 에 `FROM` 명령은 하나 이상 존재할 수 있다. 
해당 `Dockerfile` 을 빌드하게 되면 `FROM` 의 수만큼 이미지가 빌드된다. 
만약 하나 이상 `FROM` 명령이 있는 상태에서 `--tag(-t)` 옵션으로 이미지 이름을 설정한 경우에는 마지막 `FROM` 이미지에만 적용된다. 


```dockerfile
FROM ubuntu:14.04

FROM ubuntu:latest
```  

```bash
$ docker build --tag myubuntu:latest .
```

### MAINTAINER
`MANITAINER` 명령은 이미지를 생성한 사람의 정보를 설정하는 명령이다. 
보편적으로 아래와 같이 이름과 이메일을 사용해 설정한다. 
필수 명령어는 아니지만, 이미지 관리를 위해서는 필요한 정보이다.  

명령은 아래와 같은 구성으로 사용할 수 있다. 

```dockerfile
MAINTAINER <작성자 정보>
```  

```dockerfile
FROM ubuntu:latest

MAINTAINER windowforsun <windowforsun@email.com>
```  

### RUN
`RUN` 명령은 `FROM` 의 이미지에 명령을 수행하는 명령이다. 
즉 `RUN` 명령어는 `Dockerfile` 를 `docker run` 명령으로 이미지로 빌드할 때 수행하는 명령들을 작성한다. 
빌를 하게 되면 `RUN` 명령을 수행한 결과가 이미지로 생성된다. 
`Dockerfile` 내에서 `RUN` 명령을 수행한 내역은 이후 이미지 히스토리를 통해 확인 할 수 있다.  

명령은 아래와 같은 구성으로 사용 할 수 있다. 

```dockerfile
RUN <명령>
```  

추후에 설명하는 `RUN` 을 포함해서 `CMD`, `ENTRYPOINT` 와 같이 이미지 상에서 명령을 수행하는 방식은 아래 2가지가 있다. 
 
실행 형식|쉘 형식(`/binsh`)
---|---
`RUN ["apt-get", "install", "-y", "nginx"]| `RUN apt-get install -y nginx`
실행 명령과 매개 변수를 문자열 배열 형식으로 설정한다. `/bin/sh` 실행 파일을 사용하지 않기 때문에, 쉘 스크립트 문법으로 인식하지 않는다.|`/bin/sh` 실행 파일을 사용해서 명령을 수행하기 때문에, `/bin/sh` 실행 파일이 있어야 명령 수행이 가능하다.

실행 형식으로 작성한 예시는 아래와 같다. 

```bash
FROM ubuntu:latest

RUN ["apt-get", "update", "-y"]
RUN ["apt-get", "install", "-y", "nginx"]
RUN ["mkdir", "-p", "/tmp/test"]
RUN ["touch", "/tmp/test/msg"]
```  

쉘 형식으로 작성한 예시는 아래와 같다. 

```dockerfile
FROM ubuntu:latest

RUN apt-get -y update
RUN apt-get install -y nginx
RUN mkdir -p /tmp/test
RUN touch /tmp/test/msg
```  

실행 형식과 쉘 형식은 필요에 따라 선택해서 사용할 수 있다.  

빌드를 수행하면 실행한 `RUN` 명령의 결과는 캐싱을 통해 다음 빌드때 해시값이 같다면 재사용하게 된다. 
캐시를 사용하지 않고 빌드마다 재수행을 하려면 `docker build --no-cache` 옵션을 사용 할 수 있다. 


### CMD
`CMD` 명령은 `RUN` 명령어와는 빌드된 이미지를 컨테이너로 실행한 시점에 수행하는 명령이다. (`docker run`)
추가로 `docker start` 명령으로 정지된 컨테이너를 다시 시작 할때도 `CMD` 명령은 수행된다. 
이러한 특성으로 `CMD` 명령은 `Dockerfile` 에서 한 번만 사용할 수 있다. 

앞서 언급한 것과 같이 `CMD` 명령 또한 실행 형식과 쉘 형식으로 작성해 사용 할 수 있다. 

```dockerfile
CMD <명령>
```  

실행 형식으로 작성한 예시는 아래와 같다. 

```dockerfile
FROM ubuntu:latest

CMD ["free", "-m"]
```  

쉘 형식으로 작성한 예시는 아래와 같다. 

```dockerfile
FROM ubuntu:latest

CMD free -m
```  

작성한 두 `Dockerfile` 은 모두 동일한 동작을 수행한다. 
실제로 빌드하고 실행하면 `free -m` 명령의 결과를 출력하고 컨테이너는 종료된다.  

`docker run` 명령을 수행할 때 마지막에 컨테이너에 수행할 명령을 인자로 전달 할 수 있다. 
이때 `CMD` 명령이 이미지에 지정돼 있더라도, `docker run` 인자로 전달된 명령으로 대체된다. 

```dockerfile
FROM ubuntu:latest

CMD ["echo", "this is cmd"]
```  

위와 같은 `Dockerfile` 을 빌드하고, 한번은 명령 인자를 전달하고 다른 한번은 전달하지 않으면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ docker run --rm test
this is cmd
$ docker run --rm test echo this is args
this is args
```  


`CMD` 명령을 `ENTRYPOINT` 와 함께 사용하게 되면, 
`CMD` 는 `ENTRYPOINT` 에 작성된 명령에 전달되는 매개변수 역할만 수행한다. 

```dockerfile
FROM ubuntu:latest

ENTRYPOINT ["echo"]
CMD ["this is cmd"]
```  

위 `Dockerfile` 을 빌드하고 실행하면,
실제로 `this is cmd` 문자열을 `echo` 명령으로 출력하고 컨테이너는 종료된다. 

### ENTRYPOINT
`ENTRYPOINT` 명령은 컨테이너가 시작할 때 수행하는 명령이다. 
`docker run` 혹은 `docker start` 명령으로 컨테이너를 시작하면 수행하는 명령어로, 
`CMD` 와 비슷한 역할을 수행하지만, 

### EXPOSE

### ENV

### ADD

### COPY

### VOLUME

### USER

### WORKDIR

### ONBUILD

### .dockerignore

### 주석





---
## Reference
	