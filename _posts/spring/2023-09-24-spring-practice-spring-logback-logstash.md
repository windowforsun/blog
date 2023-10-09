--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Logback logging with Logstash"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 발생하는 로그를 Logback 을 사용해 Logstash 에 전송해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
toc: true
use_math: true
---  

## Spring Logback Logstash
[logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder)
는 `Spring` 에서 기본 로깅으로 사용하는 `Logback` 을 통해 `Logstash` 로 바로 애플리케이션 로그를 전송할 수 있는 
`Appender` 와 `Encoder` 를 제공한다.  

일반적으로 애플리케이션의 로그를 `Aggegation` 하기 위해 파일, `stdout` 혹은 `syslog` 
등으로 남긴 로그를 `Filebeat` 을 통해 `Logstash` 로 전송하는 경우가 많다. 
물론 이러한 방식을 사용하는 것은 각 요소마다 역할이 있기 때문이지만, 
애플리케이션의 출력 로그를 바로 `Logstash` 를 전송 할 수 있는 라이브러리가 있어 사용해보려 한다.  

라이브러를 사용해서 `logstash` 에 로그를 전송하는 간단한 예제와 
그 성능에 대해 알아본다.  

### How to use
사용법을 알아보기 위해 `Webflux` 의 `access log` 를 `Logstash` 로 전송하는 예제를 만들어 본다.  

라이브러리 적용을 위해서는 `build.gradle` 에 아래와 같은 의존성을 추가한다. 

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'net.logstash.logback:logstash-logback-encoder:7.3'
    implementation 'ch.qos.logback:logback-access:1.4.5'
    implementation 'ch.qos.logback:logback-classic:1.4.5'
}
```   

애플리케이션을 설정하는 `application.yaml` 에는 `Logback` 설정 파일의 위치를 명시해 준다.  

```yaml
logging:
  config: classpath:${spring.profiles.active}-logback.xml
```  

그리고 설정한 경로에 `logback.xml` 파일을 아래 내용으로 작성한다. 
아래 파일 내용은 `LogstashTcpSocketAppender` 를 사용해서 `info` 레벨에 해당하는 
로그를 모두 `localhost:5000` 의 타겟이 되는 `Logstash` 로 전송하겠다는 내용이다. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />
    
    <appender name="LOGBACK" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>localhost:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        </encoder>
    </appender>

    <logger name="org.springframework" level="info"/>

    <root level="info">
        <appender-ref ref="LOGBACK"/>
    </root>

</configuration>
```  

테스트로 `Logstash` 가 필요하다면 `Docker` 와 `docker-compose` 를 사용해 아래 템플릿으로 구성 할 수 있다.  

```yaml
version: '3.3'

services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    container_name: logstash
    ports:
      - "5000:5000"
    volumes:
      - ./config/:/usr/share/logstash/pipeline/
```  

`config` 경로에 위치하는 `Logstash` 설정 파일인 `logstash.conf` 내용은 아래와 같다. 

```conf
# logstash.conf

input {
    tcp {
        port => 5000
        codec => json_lines
    }

}

output {
    stdout {
        codec => rubydebug
    }
}
```  

실행 할 땐 `docker-compose up --build` 명령을 사용하고, 
종료하고 삭제할 때는 `docker-compose stop` 혹은 `docker-compose rm` 을 사용한다.  

이제 구성한 애플리케이션을 실행 할때 `VM` 옵션으로 
`-Dreactor.netty.http.server.accessLogEnabled=true` 값을 설정해 준다. 
그리고 `info` 레벨 로그를 발생하거나, 요청을 보내면 아래와 같이 `Logstash` 로 애플리케이션 로그가 전송된 것을 확인 할 수 있다.  

```bash
$ docker logs -f logstash

logstash  | {
logstash  |         "message" => "Started LogbackLogstashApplication in 0.813 seconds (JVM running for 1.205)",
logstash  |     "level_value" => 20000,
logstash  |        "@version" => "1",
logstash  |      "@timestamp" => 2023-09-16T08:55:12.162Z,
logstash  |     "logger_name" => "com.windowforsun.logstash.LogbackLogstashApplication",
logstash  |     "thread_name" => "main",
logstash  |           "level" => "INFO"
logstash  | }
logstash  | {
logstash  |         "message" => "127.0.0.1 - - [16/9월/2023:17:55:25 +0900] \"GET /ok HTTP/1.1\" 200 2 56",
logstash  |     "level_value" => 20000,
logstash  |        "@version" => "1",
logstash  |      "@timestamp" => 2023-09-16T08:55:26.022Z,
logstash  |     "logger_name" => "reactor.netty.http.server.AccessLog",
logstash  |     "thread_name" => "reactor-http-nio-2",
logstash  |           "level" => "INFO"
logstash  | }
logstash  | {
logstash  |         "message" => "127.0.0.1 - - [16/9월/2023:17:55:27 +0900] \"GET /ok HTTP/1.1\" 200 2 3",
logstash  |     "level_value" => 20000,
logstash  |        "@version" => "1",
logstash  |      "@timestamp" => 2023-09-16T08:55:27.400Z,
logstash  |     "logger_name" => "reactor.netty.http.server.AccessLog",
logstash  |     "thread_name" => "reactor-http-nio-3",
logstash  |           "level" => "INFO"
logstash  | }
logstash  | {
logstash  |         "message" => "127.0.0.1 - - [16/9월/2023:17:55:27 +0900] \"GET /ok HTTP/1.1\" 200 2 1",
logstash  |     "level_value" => 20000,
logstash  |        "@version" => "1",
logstash  |      "@timestamp" => 2023-09-16T08:55:27.997Z,
logstash  |     "logger_name" => "reactor.netty.http.server.AccessLog",
logstash  |     "thread_name" => "reactor-http-nio-4",
logstash  |           "level" => "INFO"
logstash  | }
```  

### Performance test
[logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder)
를 실제 애플리케이션에 사용 했을 때 어느정도의 성능을 보이는지 알아보기 위해 테스트를 진행한다. 
진행할 테스트 유형은 아래와 같다. 

- `file` : 로그를 파일로 남기는 경우

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />

    <appender name="FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
        <file>app-log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>app-log.%d{yyyyMMdd}</fileNamePattern>
        </rollingPolicy>
    </appender>

    <logger name="org.springframework" level="info"/>

    <root level="info">
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```  

- `logstash` : 로그를 `Logstash appender` 의 `TCP` 방식을 사용해 남기는 경우  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />

    <appender name="LOGSTASH_TCP" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>logstash:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"type":"LOGSTASH_TCP"}</customFields>
        </encoder>
    </appender>

    <logger name="org.springframework" level="info"/>

    <root level="info">
        <appender-ref ref="LOGSTASH_TCP"/>
    </root>

</configuration>
```  

- `aync-logstash` : 로그를 `Logstash async appender` 의 `TCP` 방식을 사용해 남기는 경우

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />

    <appender name="LOGSTASH_TCP" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>logstash:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"type":"ASYNC_LOGSTASH_TCP"}</customFields>
        </encoder>
    </appender>

    <appender name="ASYNC_LOGSTASH_TCP" class="net.logstash.logback.appender.LoggingEventAsyncDisruptorAppender">
        <appender-ref ref="LOGSTASH_TCP"/>
    </appender>

    <logger name="org.springframework" level="info"/>

    <root level="info">
        <appender-ref ref="ASYNC_LOGSTASH_TCP"/>
    </root>

</configuration>
```

테스트 대상 `Endpoint` 가 되는 `Controller` 내용은 아래와 같다.  

```java
@RestController
public class HelloController {
    @GetMapping("/ok")
    public Mono<String> ok() {
        return Mono.fromSupplier(() -> "ok");
    }
}
```  

`Docker` 와 `docker-compose` 를 테스트 환경 구성에 사용하는데 `Docker image` 를 빌드하는 `build.gradle` 내용은 아래와 같다.  

```groovy
plugins {
    id 'org.springframework.boot' version '2.6.4'
    id 'java'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

ext {
    // 원하는 환경 프로필 지정
    // file, logstash, async-logstash
    profile = 'file'
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
        image = "logback-app"
        tags = ["${profile}".toString()]
    }
    container {
        mainClass = "com.windowforsun.logstash.LogbackLogstashApplication"
        jvmFlags = [
                "-Dreactor.netty.http.server.accessLogEnabled=true"
        ]
        ports = ["8080"]
        environment = [
                'SPRING_PROFILES_ACTIVE' : "${profile}".toString()
        ]
    }

}
```  

위 `build.gradle` 내용을 사용하면 `./gradlew jibDockerBuild` 명령으로 원하는 애플리케이션 이미지를 빌드할 수 있다.  

빌드한 이미지를 사용해 테스트 환경을 구성하는 `docker-compose` 템플릿 내용은 아래와 같다. 
모든 컨테이너는 `cpu` 제한을 `2core` 로 설정해서 동일 리소스에서 테스트가 진행 될 수 있도록 했다.  

```yaml
version: '3.3'

services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    container_name: logstash
    deploy:
      resources:
        limits:
          cpus: "2"
    ports:
      - "5000:5000"
      - target: 514
        published: 514
        protocol: udp
    volumes:
      - ./config/:/usr/share/logstash/pipeline/
    networks:
      - test-net

  app-logstash:
    image: logback-app:logstash
    container_name: app-logstash
    deploy:
      resources:
        limits:
          cpus: "2"
    ports:
      - "8081:8080"
    networks:
      - test-net

  app-async-ogstash:
    image: logback-app:async-logstash
    container_name: app-async-logstash
    deploy:
      resources:
        limits:
          cpus: "2"
    ports:
      - "8082:8080"
    networks:
      - test-net

  app-file:
    image: logback-app:file
    container_name: app-file
    deploy:
      resources:
        limits:
          cpus: "2"
    ports:
      - "8083:8080"
    networks:
      - test-net

networks:
  test-net:
```  

#### 테스트 결과
`docker-compose up --build` 로 전체 구성을 실행하고,
테스트는 `Jmeter` 를 사용해서 `/ok` 경로에 요청을 보내 `Access log` 가 발생되는 상황에서
얼마 만큼의 `TPS` 가 나오는지에 대해 측정 했다. 
모든 테스트 결과는 5번 반복한 평균 값을 구했다.  


로그 방식|CPU|Throughput/s|Avg response time
---|---|---|---
File|100%|10826|8ms
Logstash|100%|11999|7ms
Async Logstash|100%|11222|8ms

테스트 결과를 보면 생각보다 `File` 로 로그를 남기는 것과 
큰 차이가 없거나 오히려 더 높은 성능으 보여 주었다. 
물론 로컬 환경에서 진행한 테스트 이기 때문에, 네트워크 환경 등이 실제 환경과 다를 순 있지만 
비교적 준수한 성능을 보여준 것을 확인 할 수 있다.  

성능적으로 민감하지 않는 애플리케이션의 경우 손쉬운 설정으로 로그를 `Logstash` 에 전송 할 수 있으므로 
잘 활용하면 좋을 것 같고, 
성능적으로 민감한 애플리케이션 또한 테스트 진행 후 도입도 검토해 볼만 하다.  

---  
## Reference
[logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder)  