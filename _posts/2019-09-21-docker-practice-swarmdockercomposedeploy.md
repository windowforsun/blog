--- 
layout: single
classes: wide
title: "[Docker 실습] Docker Swarm 을 이용한 Docker Compose Application 배포"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Swarm 을 이용해서 Docker Compose 에 작성된 Application 을 배포해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Practice
    - Spring
    - SpringBoot
    - Swarm
---  

# 환경
- Docker
- Spring Boot
- Maven
- Redis

# 목표
- Docker Swarm 을 이용해서 분산된 서버 구조로 애플리케이션 배포를 한다.
- Docker Compose 를 사용해서 배포될 애플리케이선을 구성한다.

# 방법
- Dockerfile 에 각 Container 에 대한 구성을 작성한다.
- Docker Compose 에 작성된 Dockerfile 이미지를 기반으로 전체적인 애플리케이션을 구성한다.
- 분산된 서버에서 Docker Swarm 을 설정하고 이를 연동한다.
- 애플리케이션을 배포하고 결과를 확인한다.

# 예제
## Container 구성하기
- Container 는 Image 가 실해애되어 메모리에 올라간 상태를 뜻한다.
- Container 는 하나의 프로세스와 같은 의미를 같는다.
- Dockerfile 을 통해 하나의 Container 를 정의 할 수 있다.

### 프로젝트 구조 

![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-swarm-deploy-1.png)


- `TestController` 클래스

	```java
	@RestController
	public class TestController {
	
	    @GetMapping("/")
	    public String root() throws Exception{
	        return InetAddress.getLocalHost().getHostName() + ", current time : " + System.currentTimeMillis();
	    }
	}
	```   

- `pom.xml` 내용

	```xml
	<?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.1.8.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <groupId>com.example</groupId>
        <artifactId>demo</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>spring-boot-docker-swarm-deploy</name>
        <description>Demo project for Spring Boot</description>
    
        <properties>
            <java.version>1.8</java.version>
        </properties>
    
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-redis</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
    
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <optional>true</optional>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>
    
        <build>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                </plugin>
            </plugins>
        </build>
    
    </project>
	```  
	
- `docker/web/Dockerfile`

	```dockerfile
	### BUILD image
    FROM maven:3-jdk-11 as builder
    # create app folder for sources
    RUN mkdir -p /build
    WORKDIR /build
    COPY pom.xml /build
    #Download all required dependencies into one layer
    RUN mvn -B dependency:resolve dependency:resolve-plugins
    #Copy source code
    COPY src /build/src
    # Build application
    #RUN mvn package
    RUN mvn package -DskipTests
    
    
    
    FROM openjdk:11-slim as runtime
    EXPOSE 8810
    #Set app home folder
    ENV APP_HOME /app
    #Possibility to set JVM options (https://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
    ENV JAVA_OPTS=""
    #Create base app folder
    RUN mkdir $APP_HOME
    #Create folder to save configuration files
    RUN mkdir $APP_HOME/config
    #Create folder with application logs
    RUN mkdir $APP_HOME/log
    VOLUME $APP_HOME/log
    VOLUME $APP_HOME/config
    WORKDIR $APP_HOME
    #Copy executable jar file from the builder image
    COPY --from=builder /build/target/*.jar app.jar
    ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar" ]
    #Second option using shell form:
    #ENTRYPOINT exec java $JAVA_OPTS -jar app.jar $0 $@
	```  
	
	- `build` 에서 Maven 이미지를 사용해서 Spring Boot 프로젝트를 `jar` 로 패키징 한다.
	- `runtime` 에서는 Java 이미지를 통해 `build` 에서 빌드한 `jar` 파일을 실행 시킨다.
	
### Dockerfile 빌드하기 및 실행
- 프로젝트의 루트 경로로 이동한다.
- `docker build` 명령어를 통해 현재 생성한 프로젝트를 이미지로 빌드시킬 수 있다.

	```
	$ docker build --tag=ex-web -f docker/web/DockerFile .
    Sending build context to Docker daemon    258kB
    Step 1/19 : FROM maven:3-jdk-11 as builder
     ---> d9d0b7c97e99
    Step 2/19 : RUN mkdir -p /build
     ---> Using cache
     ---> 4bae9f5c16c4
    Step 3/19 : WORKDIR /build
     ---> Using cache
     ---> 27998ca5c811
    Step 4/19 : COPY pom.xml /build
     ---> Using cache
     ---> c5bc3d53e2ca
    Step 5/19 : RUN mvn -B dependency:resolve dependency:resolve-plugins
     ---> Using cache
     ---> 17859ee4bf34
    
    생략 .. 
    
    Step 18/19 : COPY --from=builder /build/target/*.jar app.jar
     ---> 6f4da5680239
    Step 19/19 : ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar" ]
     ---> Running in c5aa0b78f7ee
    Removing intermediate container c5aa0b78f7ee
     ---> 12a056191c32


	생략 ..
	```  
	
	- `--tag` 는 현재 빌드하는 이미지에 이름을 지정하는 옵션이다.
	- `-f` 옵션을 통해 프로젝트 루트에서 `docker/web/Dockerfile` 경로의 Dockerfile 을 사용했다.
	- 마지막 `.` 은 현재 Dockerfile 이 빌드시에 적용되는 `Context Path` 를 의미한다. 즉 현재 빌드에서는 프로젝트 루트 경로가 빌드의 `Context Path` 이다.
	
- `docker image ls | grep ex-web` 을 통해 생성된 이미지를 확인 할 수 있다.

	```
	$ docker image ls | grep ex-web
    ex-web                          latest              12a056191c32        About a minute ago   428MB
	```  
	
- `docker run` 명령어를 통해 생성된 이미지를 실행 시킨다.

	```
	$ docker run -d -p 8810:8080 ex-web
    77a1b2f4772a9b833cb21da4a167cffa91f2e2a32f4fd855ff070e463bec7d37
	```  
	
	- `-d` 옵션은 Container 를 백그라운드로 실행시키고 Container 의 ID 값을 반환한다.
	- `-p` 옵션은 포트포워딩 설정으로 8810 포트를 실행하는 컨테이너의 8080 포트와 연결한다.
	- 마지막에는 컨테이너로 구동시킬 이미지 태그 이름이 온다.
	
- `http://localhost:8810` 으로 접속하면 호스트 이름과 타임스탬프를 출력하는 것을 확인 할 수 있다.

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-swarm-deploy-2.png)

- `docker container ls` 명령어로 현재 실행중인 컨테이너 목록을 확인 할 수 있다.
- `docker container stop` 명령어를 통해 현재 실행 중인 컨테이너를 중지 시킬 수 있다.

	```
	$ docker container stop 77a
    77a
	```  
	
### 생성된 이미지 Docker Hub 에 올리기
- Docker Hub 에 로그인이 되어 있지 않다면 `docker login` 명령어를 통해 로그인 한다.
- `docker tag` 명령어를 통해 현재 이미지를 `<username>/<repository>:<tag>` 태킹 한다.

	```
	$ docker tag ex-web windowforsun/ex-web:latest
	```  
	
	```
	$ docker image ls | grep windowforsun/ex-web
    windowforsun/ex-web             latest              12a056191c32        21 minutes ago      428MB
	```  

- `docker push` 를 통해 태깅한 이미지를 Docker Hub 에 푸시한다.

	```
	$ docker push windowforsun/ex-web:latest
    The push refers to repository [docker.io/windowforsun/ex-web]
    997c21399047: Preparing
    5453e7d13f51: Preparing
    6556de81127e: Preparing
    f8242d3d39e6: Preparing

	생략 ..
	```  
	
## Service 구성하기
- 여러 애플리케이션으로 구성된 구조에서 하나의 애플리케이션을 Service 라고 한다.
- 하나의 애플리케이션을 뜻하는 Service 는 여러 머신이나 여러개로 분산되어 질 수 있다.
- Service 는 Dockerfile 로 만들어진 이미지를 사용해 Docker Compose 파일로 정의한다.

### Docker Compose 작성하기

```yaml
version: '3'

services:
  web:
    # 앞서 Docker Hub 로 푸시한 이미지 혹은 사용할 이미지
    image: windowforsun/ex-web:latest
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
    - "4400:8080"
    networks:
    - webnet
networks:
  webnet:
```  

- Docker Compose 에 대한 자세한 내용은 [여기](https://docs.docker.com/compose/compose-file/)에서 확인 가능하다.
- `deploy` 는 배포 명령어에서만 동작한다.
	- `replicas` 로 service 를 몇개의 `Task` 로 분산 시킬지 정의 할 수 있다.
	- `resources` 를 통해 service 에서 하나의 `Task` 가 사용할 리소스를 제한 시킬 수 있다.


### Service 실행
- 분산된 구조로 배포를 위해서 `docker swarm init` 명령어르 실행시켜 Swarm 을 활성화 시켜 준다.
	- Swarm 에 대해서는 이후 기술하도록 한다.
	
	```
	$ docker swarm init
	```  
	
- `docker stack deploy` 명령어로 Docker Compose 에 구성된 애플리케이션을 배포한다.
	- Stack 에 대해서도 이후에 다룬다.
	
	```
	$ docker stack deploy -c docker-compose.yml ex-deploy
    Creating network ex-deploy_webnet
    Creating service ex-deploy_web
	```  
	
- `docker service ls` 로 현재 실행 중인 서비스 목록을 확인 할 수 있다.
	
	```
	$ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
    yrcynumo2b8m        ex-deploy_web       replicated          5/5                 windowforsun/ex-web:latest   *:8810->8080/tcp
	```  

- `docker stack services` 명령어로 현재 애플리케이션에서 실행 중인 서비스를 확인 할 수 있다.

	```
	$ docker stack services ex-deploy
    ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
    c9k7hext7ibp        ex-deploy_web       replicated          5/5                 windowforsun/ex-web:latest   *:4400->8080/tcp
	```  
	
- `docker service ps` 명령어로 현재 하나의 서비스에서 실행 중인 `Task` 목록을 확인 할 수 있다.
	- `Task` 는 하나의 Service 를 `replicas` 로 분산한 구조에서 하나의 수행 단위를 뜻한다.

	```
	$ docker service ps ex-deploy_web
    ID                  NAME                IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
    uuk9mhhdofk7        ex-deploy_web.1     windowforsun/ex-web:latest   docker-desktop      Running             Running 10 minutes ago   
    npnrmp72df4t        ex-deploy_web.2     windowforsun/ex-web:latest   docker-desktop      Running             Running 10 minutes ago   
    x3chsdti01n4        ex-deploy_web.3     windowforsun/ex-web:latest   docker-desktop      Running             Running 10 minutes ago   
    v9qswkfgkoes        ex-deploy_web.4     windowforsun/ex-web:latest   docker-desktop      Running             Running 10 minutes ago   
    imnadoaodftk        ex-deploy_web.5     windowforsun/ex-web:latest   docker-desktop      Running             Running 10 minutes ago   
	```  
	
- `docker stack ps` 명령어로 현재 `stack` 을 구성하고 있는 `Task` 의 목록을 확인 할 수 있다.

	```
	$ docker stack ps ex-deploy
    ID                  NAME                IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
    uuk9mhhdofk7        ex-deploy_web.1     windowforsun/ex-web:latest   docker-desktop      Running             Running 15 minutes ago   
    npnrmp72df4t        ex-deploy_web.2     windowforsun/ex-web:latest   docker-desktop      Running             Running 15 minutes ago   
    x3chsdti01n4        ex-deploy_web.3     windowforsun/ex-web:latest   docker-desktop      Running             Running 15 minutes ago   
    v9qswkfgkoes        ex-deploy_web.4     windowforsun/ex-web:latest   docker-desktop      Running             Running 15 minutes ago   
    imnadoaodftk        ex-deploy_web.5     windowforsun/ex-web:latest   docker-desktop      Running             Running 15 minutes ago   
	```  
	
- `docker container ls -q` 명령어는 현재 실행 중인 컨테이너 들의 ID 만 출력한다.

	```
	$ docker container ls -q
    0f2ea8b4053f
    2ba63e188525
    02ace9cbe6c9
    d2372a72224e
    de4fa5729670
	```  
	
- `curl -4 http://localhost:8810` 으로 테스트를 수행하거나 웹브라우저에서 접속 테스트를 하면 요청을 수행할 때마다 다른 컨테이너의 아이디를 출력하는 것을 알 수 있다.

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-swarm-deploy-3.png)

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-swarm-deploy-4.png)

- 스케일을 조정하는 방법은 `docker-compose.yml` 파일에서 `replicas` 항목을 수정해서 스케일을 다시 조정하고, `docker stack deploy -c docker-compose.yml ex-deploy` 를 통해 다시 배포 하면 된다.
	
	```
	$ docker stack deploy -c docker-compose.yml ex-deploy
    Updating service ex-deploy_web (id: yrcynumo2b8m)
	```  
	
	```
	$ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
    yrcynumo2b8m        ex-deploy_web       replicated          3/3                 windowforsun/ex-web:latest   *:8810->8080/tcp
	```  

- `docker stack rm` 을 통해 배포된 `stack` 을 내릴 수 있다.

	```
	$ docker stack rm ex-deploy
    Removing service ex-deploy_web
    Removing network ex-deploy_webnet
	```  
	
- `docker swarm leave --force` 로 설정했던 `swarm` 을 내릴 수 있다.

	```
	$ docker swarm leave --force
    Node left the swarm.
	```  
	
## Swarm 으로 분산환경 구성하기
- Service 를 통해 단일 호스트에서 분산환경을 구성을 한것과 달리 여러 머신에 Service 의 분산환경을 구성한다.
- Swarm 은 여러 머신의 그룹으로 Docker 를 통해 구성되는 하나의 클러스터 단위이다.
- Swarm 은 Swarm Manager 와 Worker 로 구분된다.
- Docker 의 Swarm 명령어는 Manager 머신에서 실행하게 되면 구성된 머신들에 모두 적용 된다.
- Swarm 으로 구성된 각 머신들을 Node 라고 한다.

### Swarm 구성하기
- Docker Get Started 페이지에서는 Docker Machine 을 통해 내부에 여러 머신을 띄워 Swarm 으로 연결하는 예제를 보여주고 있는 것과 달리, 클라우드 머신을 이용해서 이를 구성한다. ???????????????


---
## Reference
[Swarm 모드에서 compose 애플리케이션 배포](https://www.joinc.co.kr/w/man/12/docker/swarmCompose)   
[docker swarm](https://setyourmindpark.github.io/2018/02/07/docker/docker-5/)   
[Get Started, Part 2: Containers](https://docs.docker.com/get-started/part2/)   
[Get Started, Part 3: Services](https://docs.docker.com/get-started/part3/)   
[Get Started, Part 4: Swarms](https://docs.docker.com/get-started/part4/)   
[Get Started, Part 5: Stacks](https://docs.docker.com/get-started/part5/)   
