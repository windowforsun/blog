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
toc: true
use_math: true
---  

# sfsdfsdfsdfsd

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


[여기]({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileaddcopy.md %})
에도 관련 설명이 있어 참고 할 수 있다.

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

[여기]({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileaddcopy.md %})
에도 관련 설명이 있어 참고 할 수 있다.

### ENTRYPOINT
`ENTRYPOINT` 명령은 컨테이너가 시작할 때 수행하는 명령이다. 
`docker run` 혹은 `docker start` 명령으로 컨테이너를 시작하면 수행하는 명령어로, 
`Dockerfile` 에서 단 한번만 사용할수 있다는 점에서 `CMD` 명령과 비슷하지만 `docker run` 명에서 차이점을 보인다.  

`ENTRYPOINT` 명령 또한 실행 형식과 쉘 형식 2가지 방법으로 사용할 수 있다. 

```dockerfile
ENTRYPOINT <명령>
```  

실행 형식으로 작성한 예시는 아래와 같다. 

```dockerfile
FROM ubuntu:latest

ENTRYPOINT ["free", "-m"]
```  

쉘 형식으로 작성한 예시느 아래와 같다. 

```dockerfile
FROM ubuntu:latest

ENTRYPOINT free -m
```  

앞서 `CMD` 와 `ENTRYPOINT` 는 비슷한 역할을 수행하지만, `docker run` 명령에서 차이점을 보인다고 언급했었다. 
`CMD` 는 `docker run` 명령에 실행 인자를 전달하면 `Dockerfile` 에 작성한 `CMD` 는 무시된다. 
하지만 `ENTRYPOINT` 는 `docker run` 명령에 실행 인자를 전달하면, 
`Dockerfile` 에 작성한 `ENTRYPOINT` 의 명령은 그대로 사용되고 `docker run` 의 실행 인자는 `ENTRYPOINT` 의 매개 변수값으로 사용 된다. 
아래 와 같은 `Dockerfile` 을 빌드하고 두 가지 종류로 `docker run` 을 수행한 결과는 아래와 같다. 

```dockerfile
FROM ubuntu:latest

ENTRYPOINT ["echo", "this is entrypoint"]
```  

```bash
$ docker run --rm test
this is entrypoint
$ docker run --rm test hello~!
this is entrypoint hello~!
```  

`Dockerfile` 에 작성된 `ENTRYPOINT` 를 `docker run` 시점에 무시하고 새로운 명령을 수행하는 방법은 `--entrypoint=<명령>` 옵션을 사용하는 것이다. 

```bash
$ docker run --rm --entrypoint="free" test -m
              total        used        free      shared  buff/cache   available
Mem:          25563        1379       22206           5        1977       24005
Swap:          7168           0        7168
```  


[여기]({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileaddcopy.md %})
에도 관련 설명이 있어 참고 할 수 있다. 

### EXPOSE
`EXPOSE` 명령은 호스트와 `Dockerfile` 에 정의한 이미지의 컨테이너와 연결할 포트를 정의하는 명령이다. 
`docker run` 명령에서 `--expose` 옵션과 동일한 역할을 수행한다. 

```dockerfile
EXPOSE <포트>
```  

아래는 `EXPOSE` 테스트를 위한 `Dockerfile` 예시이다. 
`EXPOSE` 는 여러 줄에 걸쳐서 사용하거나, 배열 형식으로 여러개를 한번에 설정할 수도 있다.  

```dockerfile
FROM ubuntu:latest

EXPOSE 11223
EXPOSE 22323
EXPOSE 55432 44333

ENTRYPOINT ["/bin/bash"]
```  

위 `Dockerfile` 을 빌드하고 아래 명령어로 실행한 다음, 
새로운 쉘을 켜서 `telnet` 으로 테스트를 수행하면 컨테이너 포트에 접속이 가능한 것을 확인 할 수 있다. 
`EXPOSE` 는 호스트와 연결한 수행하는 역할이므로 외부로 포트가 노출되지는 않는다. 
포트를 외부로 노출하기 위해서는 `docker run` 명령에서 `-p` 옵션을 사용해서 가능하다.  

```bash
docker run \
> --rm \
> -it \
> -p 11223:11223 \
> -p 22323:22323 \
> -p 55432:55432 \
> -p 44333:44333 \
> test
root@72936979f44d:/#
```  

```bash
telnet localhost 11223
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
$ telnet localhost 11223
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
$ telnet localhost 22323
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
$ telnet localhost 55432
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
$ telnet localhost 44333
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
```  

외부(호스트)에서 컨테이너로 각 포트를 사용해서 연결은 성공 했지만, 
현재 컨테이너에서는 각 포트에 수행하는 동작이 없기 때문에 연결이 바로 끊어진다. 

### ENV
`ENV` 명령은 환경 변수를 설정하는 명령이다.  

보다 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileenvarg.md %})
를 참고한다. 

### ARG
`ARG` 는 `Dockerfile` 빌드 시에만 사용되는 변수를 설정하는 명령이다. 

보다 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileenvarg.md %})
를 참고한다. 

### ADD
`ADD` 는 파일을 이미지에 추가하는 명령이다. 

```dockerfile
ADD <복사할 호스트 경로> <추가할 이미지 경로>
```  

`복사할 호스트 경로는` 빌드를 수행하는 컨텍스트를 기준으로 하고 아래와 같은 특징이 있다.  
- 컨텍스트의 상위경로, 절대경로는 사용할 수 없다.  
- 파일, 디렉토리를 경로로 설정 가능고 디렉토일 경우 포함되는 것을 모두 복사한다. 
- 경로에 와일드카드를 사용할 수 있다. 
- `URL` 을 사용할 수 있다. 
- 압축파일일 경우 압축을 해제한다. 

`추가할 이미지 경로` 는 아래와 같은 특징을 갖는다. 
- 절대경로를 사용해야 한다. 
- 경로가 `/` 로 끝나면 디렉토리에 복사할 파일을 복사한다. 

추가적인 정보는 [여기](({{site.baseurl}}{% link _posts/docker/2020-02-05-docker-practice-dockerfileaddcopy.md %}))
에서 확인 할 수 있다. 

### COPY

### VOLUME

### USER

### WORKDIR

### ONBUILD

### .dockerignore

### 주석





---
## Reference
	