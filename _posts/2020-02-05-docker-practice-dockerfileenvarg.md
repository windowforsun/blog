--- 
layout: single
classes: wide
title: "[Docker 실습] Dockerfile 의 ENV 와 ARG "
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Dockerfile 을 작성할때 비슷해 보이는 ENV 와 ARG 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
---  

![그림 1]({{site.baseurl}}/img/docker/practice-dockerfileenvarg-1.png)

## ENV
- Dockerfile 과 컨테이너에서 사용할 수 있는 환경 변수를 설정한다.
- `ENV <key> <value>` 형식으로 설정가능하고, `ENV <key>=<value> ...` 처럼하면 여려 환경변수를 한번에 설정 할 수 있다.

	```dockerfile
	ENV MY_ENV ENV_VALUE
	ENV MY_ENV_2=ENV_VALUE_2 MY_ENV_3=ENV_VALIE_3
	```  

- Dockerfile 에 설정된 환경변수는 자식 Dockerfile 에서도 동일하게 적용된다.
- 값을 사용할 때는 `$KEY` 혹은 `${KEY}` 로 사용 가능하다.
- `${KEY:-VALUE}` 와 같이 사용하면 기본값을 설정할 수 있다. (`${KEY:+VALUE}` 는 반대의 경우이다)
- `docker run` 명령어에 옵션 `-e` 로 오버라이딩 할 수 있다.

## ARG
- Dockerfile 빌드 시에만 사용되는 변수를 설정한다.
- `ARG <key>=<value>` 형식으로 설정한다.

	```dockerfile
	ARG MY_ARG=ARG_VALUE
	```  
	
- `${KEY:-VALUE}` 로 기본값을 설정할 수 있다.(`${KEY:+VALUE}` 는 반대의 경우)
- `docker run` 명령어 옵션 `--build-tag` 로 오버라이딩 할 수 있다.

## 테스트
- 테스트로 사용할 Dockerfile 은 아래와 같다.

	```dockerfile
	FROM alpine:latest

	ENV MY_ENV env
	ENV MY_ENV_1=env1 MY_ENV_2=env2
	
	ARG MY_ARG=arg
	ARG MY_ARG_1=arg1
	
	## build 시 실행
	RUN echo "MY_ENV = ${MY_ENV}" \
	    && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}" \
	    && echo "MY_ARG = ${MY_ARG}" \
	    && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	
	## docker run 시 실행
	CMD echo "MY_ENV = ${MY_ENV}" \
	    && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}" \
	    && echo "MY_ARG = ${MY_ARG}" \
	    && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	```  
	
- `docker build -t test .` 명령어로 빌드를 하면 아래와 같은 출력결과를 확인 할 수 있다.

	```bash
	$ docker build -t test .
	Sending build context to Docker daemon  2.048kB
	Step 1/7 : FROM alpine:latest
	 ---> e7d92cdc71fe
	Step 2/7 : ENV MY_ENV env
	 ---> Running in 069058953a50
	Removing intermediate container 069058953a50
	 ---> ea6f861bc325
	Step 3/7 : ENV MY_ENV_1=env1 MY_ENV_2=env2
	 ---> Running in e9edc9c82040
	Removing intermediate container e9edc9c82040
	 ---> 785c1f4b39e0
	Step 4/7 : ARG MY_ARG=arg
	 ---> Running in 1e838b182f74
	Removing intermediate container 1e838b182f74
	 ---> 2d040866cafd
	Step 5/7 : ARG MY_ARG_1=arg1
	 ---> Running in c3cb4c53cf34
	Removing intermediate container c3cb4c53cf34
	 ---> bd860a611176
	Step 6/6 : RUN echo "MY_ENV = ${MY_ENV}"     && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}"     && echo "MY_ARG = ${MY_ARG}"     && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	 ---> Running in d32dae09b763
	
	.. 출력 결과 ..
	MY_ENV = env
	MY_ENV_1(default) = env1
	MY_ARG = arg
	MY_ARG_1(default) = arg1
	
	Step 7/7 : CMD echo "MY_ENV = ${MY_ENV}"     && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}"     && echo "MY_ARG = ${MY_ARG}"     && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	 ---> Running in 8558573b1728
	Removing intermediate container d32dae09b763
	 ---> 6cea7b770b00
	Successfully built 6cea7b770b00
	Successfully tagged test:latest
	```  
	
- `docker run test` 로 실행시키면 아래와 같은 출력결과를 확인 할 수 있다.

	```bash
	$ docker run test
	MY_ENV = env
	MY_ENV_1(default) = env1
	MY_ARG =
	MY_ARG_1(default) = default-arg
	```  

- `docker build -t test --build-arg MY_ARG_1=new-arg1 .` 로 다시 빌드하면 아래와 같은 출력결과를 확인할 수 있다.

	```bash
	$ docker build -t test --build-arg MY_ARG_1=new-arg1 .
	Sending build context to Docker daemon  2.048kB
	Step 1/7 : FROM alpine:latest
	 ---> e7d92cdc71fe
	Step 2/7 : ENV MY_ENV env
	 ---> Using cache
	 ---> ea6f861bc325
	Step 3/7 : ENV MY_ENV_1=env1 MY_ENV_2=env2
	 ---> Using cache
	 ---> 785c1f4b39e0
	Step 4/7 : ARG MY_ARG=arg
	 ---> Using cache
	 ---> 2d040866cafd
	Step 5/7 : ARG MY_ARG_1=arg1
	 ---> Using cache
	 ---> bd860a611176
	Step 6/7 : RUN echo "MY_ENV = ${MY_ENV}"     && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}"     && echo "MY_ARG = ${MY_ARG}"     && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	 ---> Running in 9a6b40a3fe1c
	 
	.. 출력결과 ..
	MY_ENV = env
	MY_ENV_1(default) = env1
	MY_ARG = arg
	MY_ARG_1(default) = new-arg1
	
	Removing intermediate container 9a6b40a3fe1c
	 ---> 3a53d4183a7e
	Step 7/7 : CMD echo "MY_ENV = ${MY_ENV}"     && echo "MY_ENV_1(default) = ${MY_ENV_1:-default-evn}"     && echo "MY_ARG = ${MY_ARG}"     && echo "MY_ARG_1(default) = ${MY_ARG_1:-default-arg}"
	 ---> Running in 8558573b1728
	Removing intermediate container 8558573b1728
	 ---> ed6c2705ab48
	Successfully built ed6c2705ab48
	Successfully tagged test:latest
	```  
	
- `docker run -e MY_ENV_1=new-env1 test` 명령어로 빌드된 이미지를 실행하면 아래와 같은 출력결과를 확인 할 수 있다.

	```bash
	$ docker run -e MY_ENV_1=new-env1 test
	MY_ENV = env
	MY_ENV_1(default) = new-env1
	MY_ARG =
	MY_ARG_1(default) = default-arg
	```  

---
## Reference
	