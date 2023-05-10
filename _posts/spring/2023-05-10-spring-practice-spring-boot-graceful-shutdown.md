--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Web Graceful Shutdown"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Boot Web
    - Graceful
toc: true
use_math: true
---  

## Spring Boot Web Graceful Shutdown
`Spring Boot Web` 을 사용해서 웹 애플리케이션을 만들었다면, 
애플리케이션을 배포하거나(Blue/Green, Rolling), 스테일 인/아웃 등으로 기존 애플리케이션이 종료되는 경우는 빈번하다. 
이런 상황에서 구현한 애플리케이션이 정상적인 종료를 보장하는 것은 트래픽 유실을 막는데 중요하다. 
만약 처리 중인 요청이 있는 상태에서 아무런 고려 없이 애플리케이션을 종료한다면, 
해당 클라이언트는 정상적인 응답을 받을 수 없기 때문이다. 

`Graceful Shutdown` 이란 위와 같은 상황에서도 항상 정상 응답을 클라이언트에게 내려 줄 수 있도록, 
정상 종료를 수행할 수 있도록하는 방법을 의미한다.  

### 애플리케이션 종료 시그널
`Docker`, `Kubernetes` 환경에서 배포를 할때 기존 애플리케이션을 죽이고 새로운 버전의 애플리케이션을 올린다면, 
보통 `SIGTERM`(15) 시그널을 보내고 이후 설정에 따라 `SIGKILL`(9)를 보내게 된다. 

- `SIGTERM` : 프로세스를 종료시키기전에 해당 시그널을 핸들링 할 수 있다. 시그널 핸들링에 정상적인 프로세스 정상 종료 절차를 수행할 수 있다.  
- `SIGKILL` : 프로세스를 즉시 종료시킨다. 시그널을 받은 즉시 해당 프로세스는 바로 종료되기 때문에 정상 종료 절차를 수행할 수 없다. 

해당 포스트에서 알아볼 내용은 `SIGTERM` 시그널이 애플리케이션으로 들어왔을 때 웹 애플리케이션이 정상 종료가 될 수 있도록 절차를 밟는 방법에 대해 알아본다.  


## Graceful Shutdown 적용
`Spring Boot` 에서 `Graceful Shutdown` 을 적용하는 방법은 
`Spring Boot 2.3` 이상인 경우와 `Spring Boot 2.2` 이하인 경우로 나뉜다. 
`Spring Boot 2.3` 이상의 경우 프로퍼티 설정을 바탕으로 손쉽게 적용 할 수 있지만, 
`Spring Boot 2.2` 이하의 경우에는 별도의 구현 클래스를 추가해야 한다.  


### Spring Boot 2.3
`Spring Boot 2.3` 이상 버전에서는 아래 `Properties` 에 값을 애플리케이션에 맞게 설정해서 적용 할 수 있다.  

```yaml
server:
  # default is immediate
  # 종료 시그널을 받은 경우 새로운 요청은 받지 않는다. 
  # 종료 시그널을 받기 전에 처리 중인 요청을 완료 할떄까지 애플리케이션은 종료되지 않는다. 
  shutdown: graceful
spring:
  lifecycle:
    # default is 30s
    # graceful 처리를 위해 대기하는 최대 시간값
    timeout-per-shutdown-phase: 20s
```  

간단한 애플리케이션 구현으로 동작을 테스트해본다. 
구현할 애플리케이션의 디렉토리 구조는 아래와 같다.  

```
.
├── build.gradle
└── src
    └── main
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── spring23
        │               └── graceful
        │                   └── Spring23GracefulApplication.java
        └── resources
            └── application.yaml
```  

- `build.gradle`
  - 테스트는 `Docker` 환경에서 수행하기 위해 `jib` 플러그인을 사용해서 애플리케이션 이미지를 빌드한다. 
  - 비교 테스트를 위해 여러 버전 빌드를 위해 `gradle` 빌드 시점에 명령으로 `server.shutdown` 과 `spring.lifecycle.timeout-per-shutdown-phase` 을 설정한다. 

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.6.4'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'

repositories {
    mavenCentral()
}

ext {
    BUILD_VERSION = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
}

test {
    useJUnitPlatform()
}

jib {
    from {
        image = "openjdk:11-jre-slim"
        // for mac m1
        platforms {
            platform {
                architecture = "arm64"
                os = "linux"
            }
        }
    }
    to {
        image = "spring23-graceful-test"
        tags = [project.findProperty("shutdown_type")]
    }
    container {
        mainClass = "com.windowforsun.spring23.graceful.Spring23GracefulApplication"
        ports = ["8080"]
        environment = [
            'SHUTDOWN_TYPE' : project.findProperty("shutdown_type"),
            'SHUTDOWN_TIMEOUT' : project.findProperty("shutdown_timeout")
        ]
    }

}
```  

- `Spring23GracefulApplication`
  - `/{timeout}` 요청을 받으면 타임 아웃 시간 만큼 대기한 후 `OK` 를 응답한다. 

```java
@Slf4j
@SpringBootApplication
@RestController
public class Spring23GracefulApplication {
    public static void main(String... args) {
        SpringApplication.run(Spring23GracefulApplication.class, args);
    }

    @GetMapping("/{timeout}")
    public String timeout(@PathVariable long timeout) throws InterruptedException {
        log.info("start request timeout : {}", timeout);
        Thread.sleep(timeout);
        log.info("end request timeout : {}", timeout);
        return "OK";
    }
}
```  

- `application.yaml`
  - `gradle` 빌드 명령에서 파리미터로 받은 `shutdown.type` 값을 `server.shutdown` 에 설정한다. 
  - `gradle` 빌드 명령에서 파라미터로 받은 `shutdown.timeout` 값을 `spring.lifecycle.timeout-per-shutdown-phase` 에 설정한다. 

```yaml
server:
  shutdown: ${shutdown.type}
  tomcat:
    threads:
      min-spare: 100
      max: 200

spring:
  lifecycle:
    timeout-per-shutdown-phase: ${shutdown.timeout}s
```  

아래 `gradle` 명령을 사용해서 총 3가지 버전의 애플리케이션 이미지를 빌드한다.  

```bash
./gradlew jibDockerBuild -Pshutdown_type=graceful -Pshutdown_timeout=20
./gradlew jibDockerBuild -Pshutdown_type=graceful -Pshutdown_timeout=20000
./gradlew jibDockerBuild -Pshutdown_type=immediate
```  










---  
## Reference
[Stream Application Development](https://dataflow.spring.io/docs/stream-developer-guides/streams/standalone-stream-sample/)  