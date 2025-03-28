--- 
layout: single
classes: wide
title: "[Docker] Container PID 1 Problem"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Container 환경에서 PID 1 문제에 대해 알아보고, dumb-init 을 사용한 해결 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Kubernetes
  - Container
  - PID 1
  - Signal
  - Orphaned Process
  - Zombie Process
  - Dumb-init
  - Init System
  - tini
toc: true
use_math: true
---  

## PID 1
`PID 1` 은 `Linux` 시스템에서 가장 먼저 시작되는 사용자 모드 프로세스를 의미한다. 
일반적으로 `init` 프로세스라고(시스템) 불리며, 시스템이 종료될 떄까지 실행된다. 
그리고 `PID 1` 은 다른 서비스(데몬)들을 관리하고 프로세스 트리의 루트역할을 한다.  

대표적인 `init` 시스템으로는 `systemd` 와 `SysV` 등이 있는데, 
이러한 `init` 시스템은 일반적으로 `PID 1` 을 할당 받아 시스템 부팅 관리, 서비스 관리, 프로세스 관리, 시스템 상태 관리, 시스템 종료 관리 등을 수행한다. 

```bash
$ docker run -d --rm --name test --privileged centos:8 /sbin/init

$ $ docker exec -it test ps auxf  
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.0  22808  9916 ?        Ss   09:26   0:00 /sbin/init
root         126  0.0  0.0   8608  3028 pts/0    Rs+  09:27   0:00 ps auxf
root          25  0.0  0.0  28048 10164 ?        Ss   09:26   0:00 /usr/lib/systemd/systemd-journald
root          33  0.0  0.0  22396  9188 ?        Ss   09:26   0:00 /usr/lib/systemd/systemd-udevd
dbus          89  0.0  0.0  10020  3564 ?        Ss   09:26   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --system

```  

이번 포스팅에서는 이런 `PID 1` 의 역할과 그리고 `Container` 환경에서 `PID 1` 수행 방식에 따른 발생할 수 있는 문제점 들을 알아보고 
간단한 예제에 대해서도 살펴본다.  

### Signal
`PID 1` 프로세스 즉 `init` 시스템이 수행해 주는 역할 중에는 프로세스들의 `시그널` 관리를 해주는 역할 도 있다. 
여기서 `시그널` 이란 프로세스 간 통신의 한 형태로, 운영체제나 다른 프로세스가 프로세스에 특정 이벤트를 알리는 매커니즘을 의미한다. 
주요 시그널의 종류로는 아래와 같은 것들이 있다. 

- `SIGTERM`(15) : 정상 종료 요청
- `SIGKILL`(9) : 상제 종료(무시 불가)
- `SIGINT`(2) : 인터럽트 (Ctrl + C)
- `SIGHUP`(1) : 터미널 연결 종료
- `SIGCHLD`(17) : 자식 프로세스 상태 변경

즉 `PId 1` 프로세스가 `Signal` 처리 및 관리의 역할이 부족하다면 자식 프로세스들은 정상적으로 시그널을 받지 못하게 될 수 있다. 


### Orphaned/Zombie Process
`PID 1` 프로세스는 부모 프로세스의 강제 종료로 남겨진 자식 프로세스들의 새로운 부모 프로세스가 된다. 
그리고 주기적으로 `wait()` 시스템 콜을 호출하여 자식 프로세스들의 종료 상태를 수집해 `Zombie` 프로세스를 정리하고, 
프로세스 테이블에서 제거해 좀비 프로세스가 되는 것을 막아준다.  

`Orphaned Process` 는 부모 프로세스가 자식 프로세스보다 먼저 종료되어 부모가 없는 상태가 된 프로세스를 의미하고, 
`Zombie Process` 는 실행이 종료되었지만 프로세스 테이블에 여전히 남아있는 프로세스를 의미한다.  

즉 `PID 1` 프로세스가 `Orphaned/Zombie` 프로세스 관리에 대한 역할이 부족하면 하위 프로세스 관리가 되지 않아, 
불필요한 리소스가 사용될 수 있고, 과도하게 누적되면 시스템의 프로세스 테이블을 포화시켜 새로운 프로세스 생성을 방해할 수 있다.  


### Container PID 1 Problem
`Container` 환경(`Docker`, `Kubernetes`, ..) 에서는 `ENTRYPOINT(CMD)` 로 명시된 프로세스를 `PID 1` 으로 실행한다. 
그리고 `Container` 에 전달하는 모든 `Signal` 은 해당 `PID 1` 프로세스에만 전달돼 종료를 시킬 수 있다. 
이러한 이유로 컨체이너는 경량화 이미지를 사용해 단일 프로세스만 실행하는 경우가 많다.  

하지만 몇가지 상황에서는 `PID 1` 프로세스가 정상적인 역할 수행을 하지 못해 문제가 발생할 수 있다.  

전체 예제 코드는 [여기](https://github.com/windowforsun/docker-pid-1-problem-exam)
에서 확인할 수 있다.  


#### User App is PID 1
컨테이너에서 실행할 애플리케이션을 `ENTRYPOINT` 혹은 `CMD` 로 지정해 `PID 1` 으로 실행하는 상황이다. 
이렇게 사용할 경우 언어 레벨에서 시그널 이벤트 핸들러를 등록할 수 있기 때문에 단일 종료 시그널 등에 대한 처리는 정상적으로 전달되고 수행된다. 

```java
@Slf4j
@SpringBootApplication
public class ExamApplication {
	public static void main(String... args) {
		ConfigurableApplicationContext context = SpringApplication.run(ExamApplication.class, args);

		Runtime.getRuntime().addShutdownHook(new Thread(() -> {
			// log.info("Received SIGTERM. Shutting down ..");
			log.info("Received ShutdownHook. Shutting down ..");
			context.close();
		}));


		Signal.handle(new Signal("TERM"), sig -> {
			log.info("Received SIGTERM. cleanup ..");
			context.close();
			System.exit(0);
		});

		Signal.handle(new Signal("INT"), sig -> {
			log.info("Received SIGINT. cleanup ..");
			context.close();
			System.exit(0);
		});
	}

	@PreDestroy
	public void onShutdown() {
		log.info("do PreDestroy ..");
	}
}
```  

```dockerfile
# Build stage
FROM gradle:7.4.2-jdk17 AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle build --no-daemon

# Run stage
FROM openjdk:17-jdk-slim

WORKDIR /app

# Copy the jar file from the build stage
COPY --from=build /app/build/libs/*.jar app.jar

# Expose the port the app runs on
EXPOSE 8080

# Run the jar file directly, causing PID 1 issue
CMD ["java", "-jar", "app.jar"]
```  

위 내용으로 이미지를 빌드하고 컨테이너를 실행한다. 
그리고 각 종료 시그널을 보내면 시그널 처리가 정상적으로 수행되기 때문에 시그널 이벤트를 정상적으로 받고 바로 종료처리가 되는 것을 확인할 수 있다.    

```bash
$ docker build -t pid-1-test-cmd-jar -f Dockerfile-cmd-jar .

$ docker run --rm --name pid-1-test-cmd-jar pid-1-test-cmd-jar:latest

.. PID 1 화인 ..
$ docker exec -it pid-1-test-cmd-jar cat /proc/1/cmdline
java-jarapp.jar

.. SIGTERM ..
$ kill -TERM $(ps aux | grep "[p]id-1-test-cmd-jar" | awk '{print $2}')
INFO 7 --- [           main] c.w.pid1.problem.ExamApplication         : Started ExamApplication in 1.873 seconds (process running for 2.194)
INFO 1 --- [SIGTERM handler] c.w.pid1.problem.ExamApplication         : Received SIGTERM. cleanup ..
INFO 1 --- [SIGTERM handler] o.apache.catalina.core.StandardService   : Stopping service [Tomcat]
INFO 1 --- [       Thread-1] c.w.pid1.problem.ExamApplication         : Received ShutdownHook. Shutting down ..
INFO 1 --- [SIGTERM handler] c.w.pid1.problem.ExamApplication         : do PreDestroy ..

.. SIGINT ..
$ kill -INT $(ps aux | grep "[p]id-1-test-cmd-jar" | awk '{print $2}')
INFO 7 --- [           main] c.w.pid1.problem.ExamApplication         : Started ExamApplication in 1.873 seconds (process running for 2.194)
INFO 1 --- [ SIGINT handler] c.w.pid1.problem.ExamApplication         : Received SIGINT. cleanup ..
INFO 1 --- [ SIGINT handler] o.apache.catalina.core.StandardService   : Stopping service [Tomcat]
INFO 1 --- [       Thread-1] c.w.pid1.problem.ExamApplication         : Received ShutdownHook. Shutting down ..
INFO 1 --- [ SIGINT handler] c.w.pid1.problem.ExamApplication         : do PreDestroy ..

```  


#### User Shell is PID 1
사용자 쉘 스크립트를 `ENTRYPOINT` 혹은 `CMD` 로 지정해 `PID 1` 으로 지정한 경우 사용자 쉘은 
`PID 1` 이 수행해야 할 역할에 대한 처리가 돼있지 않기 때문에 문제가 발생할 수 있다.  

```dockerfile
# Build stage
FROM gradle:7.4.2-jdk17 AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle build --no-daemon

# Run stage
FROM openjdk:17-jdk-slim

WORKDIR /app

# Copy the jar file from the build stage
COPY --from=build /app/build/libs/*.jar app.jar

# Expose the port the app runs on
EXPOSE 8080

COPY ./entrypoint.sh entrypoint.sh
RUN chmod +x entrypoint.sh

# Run the jar file directly, causing PID 1 issue
ENTRYPOINT ["/app/entrypoint.sh"]
```  

```shell
#!/usr/bin/env bash

java -jar app.jar
```

이는 주로 내부 `shell` 파일에 컨테이너 내부에서 초기화 작업이 필요한 경우 초기화 작업 수행 후 애플리케이션 실행이 필요한 경우 사용할 수 있다. 
하지만 이때 `PID 1` 이 되는 프로세스는 `entrypoint.sh` 이 된다. 
이미지로 빌드 후 컨테이너를 실행한 뒤 종료 시그널을 각 보내면 시그널을 정상적으로 받지 못하고, `SIGKILL` 시그널을 보내야 종료된다.  

```bash
$ docker build -t pid-1-test-entry-shell -f Dockerfile-entry-shell .

$ docker run --rm --name pid-1-test-entry-shell pid-1-test-entry-shell:latest

.. PID 1 화인 ..
$ docker exec -it pid-1-test-entry-shell cat /proc/1/cmdline
bash/app/entrypoint.sh

.. SIGTERM ..
$ kill -TERM $(ps aux | grep "[p]id-1-test-entry-shell" | awk '{print $2}')
.. 반응 없음 ..

.. SIGINT ..
$ kill -INT $(ps aux | grep "[p]id-1-test-entry-shell" | awk '{print $2}')
.. 반응 없음 ..

.. SIGKILL ..
$ kill -KILL $(ps aux | grep "[p]id-1-test-entry-shell" | awk '{print $2}')
INFO 7 --- [           main] c.w.pid1.problem.ExamApplication         : Started ExamApplication in 1.873 seconds (process running for 2.194)
[1]    25077 killed     docker run --rm --name pid-1-test-entry-shell pid-1-test-entry-shell:latest
```  

앞서 언급한 것처럼 해당 컨테이너에 시그널을 보내면 `PID 1` 인 `entrypoint.sh` 가 받게 되는데 
해당 쉘 스크립트에는 관련 처리가 돼있지 않기 때문에 자식 프로세스인 애플리케이션으로 시그널이 전달되지 않는 것이다.  


### Dumb-init
[Dumb-init](https://github.com/Yelp/dumb-init)
은 컨테이너 환경에서 발생할 수 있는 이러한 문제를 해결하기 위해 `Yelp` 에서 만든 도구로
컨테이너 환경에서 사용하기 위해 설계된 경량 `init` 시스템이다. 
컨테이너 내에서 `PID 1` 으로 실행해 초기화 시스템역할을 한다. 
그리고 일반적인 운영체제의 `init` 시스템을 대체하여 컨테이너에 특화된 시그널 처리, 좀비 프로세스 처리, 프로세스 그룹관리 등을 제공한다.  

예제는 앞선 예제에서 `ENTRYPOINT` 에 사용자 쉘인 `entrypint.sh` 를 지정한 예제를 개선하는 방식으로 진행한다. 
사용법은 복잡하지 않다. 
우선 아래와 같이 `Dockerfile` 에서 `dumb-init` 다운로드/설치 명령을 추가하고, 
`ENTRYPOINT` 의 명령에 `dump-init` 구문을 추가한다.  

```dockerfile
# Build stage
FROM gradle:7.4.2-jdk17 AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle build --no-daemon

# Run stage
FROM openjdk:17-jdk-slim

# Install dumb-init
RUN apt-get update && apt-get install -y dumb-init

WORKDIR /app

# Copy the jar file from the build stage
COPY --from=build /app/build/libs/*.jar app.jar

# Expose the port the app runs on
EXPOSE 8080

COPY ./entrypoint.sh entrypoint.sh
RUN chmod +x entrypoint.sh

# Use dumb-init as the entry point
ENTRYPOINT ["/usr/bin/dumb-init", "--", "/app/entrypoint-exec.sh"]
```  

그리고 쉘 스크립트에서 애플리케이션을 실행할 때 앞에 아래와 같이 `exec` 구문을 추가한다.  

```shell
#!/usr/bin/env bash

exec java -jar app.jar
```  

이제 이미지를 빌드한 뒤 컨테이너를 실행하고 시그널을 각 보내면, 
아래와 같이 모두 정상 처리가 되는 것을 확인할 수 있다.  

```bash
$ docker build -t pid-1-test-dumb-init -f Dockerfile-dumb-init .

$ docker run --rm --name pid-1-test-dumb-init pid-1-test-dumb-init:latest

.. PID 1 화인 ..
$ docker exec -it pid-1-test-dumb-init cat /proc/1/cmdline
/usr/bin/dumb-init--/app/entrypoint-exec.sh

.. SIGTERM ..
$ kill -TERM $(ps aux | grep "[p]id-1-test-dumb-init" | awk '{print $2}')
INFO 7 --- [           main] c.w.pid1.problem.ExamApplication         : Started ExamApplication in 1.128 seconds (process running for 1.355)
INFO 7 --- [SIGTERM handler] c.w.pid1.problem.ExamApplication         : Received SIGTERM. cleanup ..
INFO 7 --- [SIGTERM handler] o.apache.catalina.core.StandardService   : Stopping service [Tomcat]
INFO 7 --- [       Thread-1] c.w.pid1.problem.ExamApplication         : Received ShutdownHook. Shutting down ..
INFO 7 --- [SIGTERM handler] c.w.pid1.problem.ExamApplication         : do PreDestroy ..

.. SIGINT ..
$ kill -INT $(ps aux | grep "[p]id-1-test-dumb-init" | awk '{print $2}')
INFO 7 --- [           main] c.w.pid1.problem.ExamApplication         : Started ExamApplication in 1.166 seconds (process running for 1.379)
INFO 7 --- [ SIGINT handler] c.w.pid1.problem.ExamApplication         : Received SIGINT. cleanup ..
INFO 7 --- [ SIGINT handler] o.apache.catalina.core.StandardService   : Stopping service [Tomcat]
INFO 7 --- [       Thread-1] c.w.pid1.problem.ExamApplication         : Received ShutdownHook. Shutting down ..
INFO 7 --- [ SIGINT handler] c.w.pid1.problem.ExamApplication         : do PreDestroy ..
```  
