--- 
layout: single
classes: wide
title: "[Java 개념] 외부 Docker 를 사용해서 TestContainers 사용하기"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'TestContainers 를 외부 Docker 와 연동해서 사용하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - TestContainers
  - JUnit
  - Test
toc: true 
use_math: true
---  

## TestContainers External Docker Host
[TestContainers]({{site.baseurl}}{% link _posts/spring/2020-09-25-spring-practice-testcontainer.md %})
는 `Java`, `Spring` 프로젝트에서 `Docker` 를 기반으로 외부 의존성에 대한 환경을 코드상으로 구성하면서, 
테스트까지 수행할 수 있는 라이브러리이다.  

물론 `TestContainers` 를 사용하기 위해서는 `Host` 에 `Docker Engine` 이 설치가 돼있어야 한다. 
이번 포스트에서는 만약 특별한 이유로 `Host` 에 `Docker` 를 설치하지 못하고 외부에 `Docker` 가 설치된 경우, 
외부 `Docker` 와 `TestContainers` 를 연동해서 사용하는 방법에 대해 알아본다.  

참고로 현재 `Windows 10` 에는 `Docker` 가 설치되지 않은 상태이고, 
`Windows Subsystem for Linux` 인 `WSL2` 환경에 `Docker` 가 설치된 상황이다. 
`Windows 10` 입장에서 본다면 현재 `Docker` 가 설치된 환경은 외부에 설치된 상황이라고 볼 수 있다.  

이후 아래와 같이 `Server`, `Client` 로 역할을 부여한다.  
- `Server` : `Docker` 가 설치된 환경으로 `Client` 에서 `Server` 와 연결을 통해 원격으로 `Docker` 를 사용한다. 
- `Client` : `TestContainers` 가 수행되는 환경울 `Server` 와 연동으로 코드상의 `Docker` 명령을 원격으로 수행한다. 

### Server Docker daemon.json
가장 먼저 외부에서 `Docker` 를 사용할 수 있도록 아래와 같이 `hosts` 설정을 추가해 준다. 

```json
$ cat /etc/docker/daemon.json

{
	"hosts": [
		"tcp://0.0.0.0:2375",
		"unix:///var/run/docker.sock"
	]
}
```  

### Client .docker-java.properties
`TestContainers` 는 `Host` 에 있는 `.docker-java.properties` 파일의 내용을 바탕으로 사용할 `Docker` 를 파악할 수 있다. 
그러므로 `.docker-java.properties` 의 내용을 사용하고자 하는 외부 `Docker` 의 정보로 수정해주면 된다. 

`Windows` 를 기준으로 `.docker-java.properties` 파일은 아래 경로에 있다. 

```
C:\Users\<User Name>\.docker-java.properties
or
~/.docker-java.properties
```  

해당 파일 내용을 아래와 같이 변경해 준다. 
만약 파일이 없다면 아래 내용과 `.docker-java.properties` 파일 이름으로 생성해 준다.  

```properties
DOCKER_HOST=tcp://<Docker Server Hostname or IP>:2375
DOCKER_TLS_VERIFY=0
```  

### Client testcontainers.properties
마지막으로 `TestContainers` 에서 실제로 `.docker-java.properties` 파일을 읽어, 
사용할 `Docker` 의 엔드포인트를 알 수 있도록 설정을 변경해 줘야 한다.  

`Windows` 를 기준으로 `.testcontainers.properties` 파일은 아래 경로에 있다.  

```
C:\Users\<User Name>\.testcontainers.properties
or
~/.testcontainers.properties
```  

해당 파일 내용을 아래와 같이 수정해 준다.  

```properties
#docker.client.strategy=org.testcontainers.dockerclient.NpipeSocketClientProviderStrategy
docker.client.strategy=org.testcontainers.dockerclient.EnvironmentAndSystemPropertyClientProviderStrategy
```  

이제 `TestContainers` 를 사용해서 `Docker Container` 를 코드로 구성해주고, 
테스트를 실행하면 설정한 `Docker Host` 를 사용해서 컨테이너가 실행되는 것을 확인할 수 있다.  

> 만약 `Docker Host` 에 방화벽이 있다면, 사용하는 포트마다 방화벽을 풀어주는 작업이 추가로 필요하다. 

---
## Reference
[How to run Testcontainers on a different host](https://stackoverflow.com/questions/62298539/how-to-run-testcontainers-on-a-different-host)  
[Testcontainers Custom Configuration](https://www.testcontainers.org/features/configuration/)  

