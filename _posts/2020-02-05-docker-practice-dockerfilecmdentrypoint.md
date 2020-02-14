--- 
layout: single
classes: wide
title: "[Docker 실습] Dockerfile 의 RUN 과 CMD 와 ENTRYPOINT "
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Dockerfile 을 작성할때 비슷해 보이는 RUN 과 CMD 와 ENTRYPOINT 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
---  

## RUN
- 새로운 레이어에서 명령어를 실행하고, 새로운 이미지를 생성한다.
- 주로 패키지 설치 등에 사용된다. (yum, apk, apt-get)

```dockerfile
# Shell form 
RUN <command>

# Exec form
RUN ["executable", "param1", "param2"]
```  
	
## CMD
- Default 명령어나 파라미터를 설정한다.
- `docker run <image> <command>` 명령어에서 실행할 커맨드를 주지 않았을 때, CMD 명령어가 default 로 실행된다.
- Dockerfile 에서 한번만 사용가능하고(마지막 CMD 만 유효), 컨테이너가 시작할 때 실행 된다.

```dockerfile
# Exec form, preferred way
CMD ["executable", "param1", "param2"]

# set additional default parameters for ENTRYPOINT
CMD ["param1", "param2"]

# Shell form
CMD command param1 param2
```  

## ENTRYPOINT
- 컨테이너가 시작하면서 실행하는 명령어이다.
- CMD 와 함께 쓰일 경우, ENTRYPOINT 는 실행 파일, CMD 는 매개변수 역할을 한다.
- `docker run --entrypoint="<command> <image>" 를 사용해서 오바라이딩 가능하다.

```dockerfile
# Executable form prepferred way
ENTRYPOINT ["executable", "param1", "param2"]

# Shell form
ENTRYPOINT command param1 param2
```  

## 테스트
- RUN 과 CMD 를 테스트 할 Dockerfile 은 아래와 같다.

	```dockerfile
	FROM alpine:latest

	RUN echo "run shell form
	RUN echo ["echo", "run exec form"]
	
	CMD ["echo", "cmd exe form"]
	CMD echo "cmd shell form"
	```  
	
- `docker build -t test .` 로 빌드하면 아래와 같다.

	```bash
	$ docker build -t test .
	Sending build context to Docker daemon  2.048kB
	Step 1/5 : FROM alpine:latest
	 ---> e7d92cdc71fe
	Step 2/5 : RUN echo "run shell form"
	 ---> Running in 45146876da68
	 
	.. RUN 출력 ..
	run shell form
	
	Removing intermediate container 45146876da68
	 ---> 17bd5f83f9f9
	Step 3/5 : RUN ["echo", "run exec form"]
	 ---> Running in 8105b7ca5483
	
	.. RUN 출력 ..
	run exec form
	
	Removing intermediate container 8105b7ca5483
	 ---> 6110a79293bc
	Step 4/5 : CMD ["echo", "cmd exe form"]
	 ---> Running in 6bcdbd5ff702
	Removing intermediate container 6bcdbd5ff702
	 ---> cd795fd1789b
	Step 5/5 : CMD echo "cmd shell form"
	 ---> Running in e9b7bd73e0ad
	Removing intermediate container e9b7bd73e0ad
	 ---> 876e2e0e89c3
	Successfully built 876e2e0e89c3
	Successfully tagged test:latest
	```  
	
- `docker run test` 로 실행하면 아래와 같다.

	```bash
	$ docker run test
	cmd shell form
	```  
	
	- 마지막 라인의 CMD 만 수행되는 것을 확인 할 수 있다.
	
- `docker run test echo "new cmd"` 명령어로 컨테이너 시작할때 실행할 명령어를 넣어주면 아래와 같다.

	```bash
	$ docker run test echo "new cmd"
	new cmd
	```  
	
	- Dockerfile 에 작성된 CMD 가 실행되지 않고, `docker run` 수행 시 작성한 명령어가 실행된 것을 확인 할 수 있다.
	
- RUN + ENTRYPOINT 를 테스트 할 Dockerfile 은 아래와 같다.

	```dockerfile
	FROM alpine:latest

	RUN echo "run shell form
	RUN echo ["echo", "run exec form"]
	
	ENTRYPOINT ["echo"]
	CMD ["cmd parameters for ENTRYPOINT"]
	```  
	
- `docker build -t test .` 로 빌드하면 아래와 같다.

	```bash
	$ docker build -t test .
	Sending build context to Docker daemon  2.048kB
	Step 1/5 : FROM alpine:latest
	 ---> e7d92cdc71fe
	Step 2/5 : RUN echo "run shell form"
	 ---> Running in 3ff3e93d7fd9
	 
	.. RUN 출력 결과 ..
	run shell form
	
	Removing intermediate container 3ff3e93d7fd9
	 ---> e9ffb33898ca
	Step 3/5 : RUN ["echo", "run exec form"]
	 ---> Running in 4624e0ccf630
	 
	.. RUN 출력 결과 ..
	run exec form
	
	Removing intermediate container 4624e0ccf630
	 ---> 6c2bea2b29fd
	Step 4/5 : ENTRYPOINT ["echo"]
	 ---> Running in fa384b630f28
	Removing intermediate container fa384b630f28
	 ---> 1386584d1087
	Step 5/5 : CMD ["cmd parameters for ENTRYPOINT"]
	 ---> Running in c907e2cd5df2
	Removing intermediate container c907e2cd5df2
	 ---> c63d4cc379e5
	Successfully built c63d4cc379e5
	Successfully tagged test:latest
	```  
	
- `docker run test` 로 실행하면 아래와 같다.

	```bash
	$ docker run test
	cmd parameters for ENTRYPOINT
	```  
	
	- ENTRYPOINT 는 명령어, CMD 는 파라미터로 역할을 수행한 것을 확인 할 수 있다.
	
- `docker run --entrypoint="cat" test /etc/hostname` 로 실행 시키면 아래와 같다.
	
	```bash
	$ docker run --entrypoint="cat" test /etc/hostname
	464b1808bc08
	```  
	
	- Dockerfile 에 작성된 ENTRYPOINT 와 CMD 가 실행되지 않고, `docker run` 에서 `--entrypoint` 옵션으로 준 명령어와 COMMAND 가 수행된 것을 확인 할 수 있다.

---
## Reference
[understand-how-cmd-and-entrypoint-interact](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)