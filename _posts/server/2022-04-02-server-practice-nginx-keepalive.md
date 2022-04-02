--- 
layout: single
classes: wide
title: "[Nginx] Nginx Keepalive 설정과 주의사항"
header:
  overlay_image: /img/server-bg.jpg
excerpt: 'Nginx 의 Keepalive 설정 방법과 주의사항에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Nginx
  - Keepalive
  - tcpdump
toc: true
use_math: true
---  

## Nginx Keepalive
애플리케이션 서버를 운영할때 선호하는방법은 앞단에 `Nginx`, `Apache` 와 같은 웹서버를 두는 것이다. 
이러한 웹서버를 앞단에 두면서 `https` 설정이라던가 헤더 검사, 로드 밸런서 등 
다양한 역할을 수행할 수 있도록 설정할 수 있다. 
위 구조를 도식화 하면 아래와 같을 것이다.  

![그림 1]({{site.baseurl}}/img/server/practice-nginx-keepalive-1.png)  

`Client` 는 `80 port` 인 `Nginx` 에게 요청을 보내면 
`Nginx` 는 해당 경로와 포트에 미리 설정된 동작을 수행하고 `upstream` 에 해당하는 `WAS` 의 `8080 port` 로 해당 요청을 전달한다. 
`Nginx` 가 아무리 성능적으로 우수한 웹서버라고 하지만, 
다양한 동작을 수행하다보면 성능 저하는 피할 수 없을 것이다.  

이런 성능 저하를 조금이나마 줄일 수 있는 방법이 바로 `Keepalive` 이다. 
위 그림을 보면 `Nginx` 는 `Client` 와 `WAS` 사이에서 중간자 즉 프록시 역할을 수행해주고 있다. 
그러므로 `Keepalive` 또한 `Client-Nginx`, `Nginx-WAS` 처럼 2가지 종류의 설정을 할 수 있다. 
한가지만 설정하면 모두 적용 되는것이 아니라 각 구간 상황과 스펙에 맞는 설정이 필요하다.  

![그림 1]({{site.baseurl}}/img/server/practice-nginx-keepalive-2.png)  


[Keepalive]({{site.baseurl}}{% link _posts/server/2022-03-29-server-concept-keepalive.md %}) 
에서 설명한 것처럼 `Keepalive` 란 서버에서 제공해야 클라이언트가 사용할 수 있는 기능이다. 
즉 `Client-Nginx` 의 `Keepalive` 는 `Nginx` 의 설정에 따라 관리되고, 
`Nginx-WAS` 의 `Keepalive` 는 `WAS` 설정에 따라 관리 된다.  

### 테스트 환경
테스트는 `Docker` 와 `Docker Compose` 를 사용해서 구성한다. 
구성되는 구조는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/server/practice-nginx-keepalive-3.png)  

`Docker Compose` 템플릿 파일 내용은 아래와 같다.  

```yaml
version: '3'

services:
  nginx:
    container_name: nginx
    image: nginx:test
    ports:
      - 80:80
    networks:
      - test-net

  springapp:
    container_name: springapp
    image: upstream-application:latest
    ports:
      - 8080:8080
    networks:
      - test-net

networks:
  test-net:
```  

위 `Docker Compose` 템플릿은 아래 명령으로 실행 가능하다.  

```bash
$ docker-compose up --build
```

`Spring Boot` 애플리케이션 구현에 필요한 내용은 아래와 같다.  

- `build.gradle`
```groovy
plugins {
    id 'org.springframework.boot' version '2.5.0'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

group = 'com.windowforsun.upstreamapplication'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    compileOnly "org.projectlombok:lombok"
    testCompileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
    testAnnotationProcessor "org.projectlombok:lombok"
}

test {
    useJUnitPlatform()
}


jib {
    from {
        image = "openjdk:11-jre-slim"
    }
    to {
        image = "upstream-application"
        tags = ["${project.version}".toString()]
    }
    container {
        mainClass = "com.windowforsun.upstreamapplication.UpstreamApplication"
        ports = ["8080"]
    }
}
```  

- 코드

```java
@SpringBootApplication
public class UpstreamApplication {
    public static void main(String[] args) {
        SpringApplication.run(UpstreamApplication.class, args);
    }
}
```  

```java
@RestController
public class HelloController {
    public static ConcurrentHashMap<Integer, String> map = new ConcurrentHashMap<>();

    @GetMapping("/test/{millis}/{length}")
    private String test(@PathVariable long millis, @PathVariable int length) throws Exception {
        long start = System.currentTimeMillis();
        Thread.sleep(millis);
        String result = "";
        if(map.containsKey(length)) {
            result = map.get(length);
        } else {
            StringBuilder str = new StringBuilder();
            for(int i = 0; i < length; i++) {
                str.append("0");
            }
            result = str.toString();
            map.put(length, result);
        }
        long during = System.currentTimeMillis() - start;
        return String.format("%s, %d", result, during);
    }
    
    @GetMapping("/test")
    private String test() {
        return "ok";
    }
}
```  

```yaml
server:
  http2:
    enabled: true
  tomcat:
    threads:
      max: 3
      min-spare: 1
    connection-timeout: 1s
#    keep-alive-timeout: -1
```  

`Spring Boot` 도커 이미지는 아래 명령으로 빌드하면 `upstream-aplication` 이라는 이름으로 빌드 된다.  

```bash
$ ./gradlew jibDockerBuild
$ docker image ls | grep upstream
upstream-application                                          latest            ef6f95ec0b8e   52 years ago     245MB
```  

`Nginx` 이미지를 빌드에 필요한 `Dockerfile` 과 `nginx.conf` 내용은 아래와 같다.  

```dockerfile
FROM nginx:latest

RUN apt update
RUN apt install -y vim tcpdump net-tools

# if use nginx.conf
COPY nginx.conf /etc/nginx/nginx.conf

RUN mkdir /logs

CMD ["nginx", "-g", "daemon off;"]
```  

```
user  nginx;
worker_processes  8;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection';

    keepalive_timeout  0;
    #keepalive_disable msie6;
    #keepalive_requests 10;

    upstream app {
        server springapp:8080;
        #keepalive 128;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /logs/error.log warn;

      location / {
          proxy_pass http://app;
          #proxy_connect_timeout 1s;
          #proxy_send_timeout 1s;
          #proxy_read_timeout 3s;
          #proxy_http_version 1.1 ;
          #proxy_set_header Connection "";

      }

      location = /server-status {
          stub_status on;
          access_log off;
      }
    }
}
```  

> 위 `nginx.conf` 는 현재 `Client-Nginx`, `Nginx-WAS` 모두 `Keepalive` 를 사용하지 않는 설정이다.  

아래 명령으로 `nginx:test` 이미지를 빌드할 수 있다. 

```bash
$ docker build -t nginx:test .
```  


### Client Keepalive
`Client-Nginx` 간의 `Keepalive` 관련된 `Nginx` 설정 필드는 아래와 같다.  

필드|예시|설명
---|---|---
keepalive_timeout|keepalive_timeout 10|keepalive 커넥션을 최대 10초 동안 유지한다. 
keepalive_disable|keepalive_disable msie6|keepalive 를 사용하지 않을 브라우저를 명시할 수 있다. 
keepalive_requests|keepalive_requests 1000|keepalive 커넥션 마다 사용할 수 있는 최대 요청 수를 설정한다. 


`Cleint - Nginx` 간의 `Keepalive` 설정은 설정에 주의가 필요하다. 
일반적으로 오픈된 웹서비스의 경우 불특정 다수의 요청을 받아 처리하는 역할이 대부분이다. 
이런 상황에서 `Keepalive` 가 설정돼 있는데 
`keepalive_timeout` 이 길거나 `keepalive_requests` 요청 수가 너무 크게 설정돼 있다면, 
너무 많은 커넥션이 연결된 상태로 유지 되기 때문에 더이상 가용할 수 있는 포트가 없거나, 메모리 부족 현상이 발생 할 수 있다.  

> `Client-Nginx` 간의 `Keepalive` 는 사용하지 않는 것도 안정적인 운영의 한 방법이 될 수 있다. 

![그림 1]({{site.baseurl}}/img/server/practice-nginx-keepalive-4.png)  


`Client-Nginx` 의 `Keepalive` 설정은 `Nginx` 설정이 필요하다. 
`Keepalive` 설정이 추가된 `nginx.conf` 는 아래와 같다.  

```
user  nginx;
worker_processes  8;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection';

    keepalive_timeout  30;
    keepalive_disable safari;
    keepalive_requests 10;

    upstream app {
        server springapp:8080;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /logs/error.log warn;

      location / {
          proxy_pass http://app;
      }

      location = /server-status {
          stub_status on;
          access_log off;
      }
    }
}
```  

`nginx:test` 이미지 빌드후 `Docker Compose` 템플릿을 실행한다. 
그리고 `nginx` 컨테이너에 접속해 `80` 포트로 `tcpdump` 명령을 사용해서 모니터링을 수행한다. 

```bash
$ docker build -t nginx:test .
$ docker-compose up --build -d
$ docker exec -it nginx /bin/bash
$ tcpdump port 80
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes

```  

위 상태에서 `http://<nginx-host>/test` 로 요청을 보내면 아래와 같은 결과를 확인 할 수 있다.  

```bash
09:10:00.771463 IP client.58944 > nginx.80: Flags [S], seq 785244136, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
09:10:00.771477 IP nginx.80 > client.58944: Flags [S.], seq 1094522703, ack 785244137, win 64240, options[mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:10:00.771640 IP client.58944 > nginx.80: Flags [.], ack 1, win 8212, length 0
09:10:00.772312 IP client.58902 > nginx.80: Flags [P.], seq 1:519, ack 1, win 8212, length 518: HTTP: GET/test HTTP/1.1
09:10:00.772336 IP nginx.80 > client.58902: Flags [.], ack 519, win 501, length 0
09:10:00.777909 IP nginx.80 > client.58902: Flags [P.], seq 1:161, ack 519, win 501, length 160: HTTP: HTTP/1.1 200
09:10:00.826077 IP client.58902 > nginx.80: Flags [.], ack 161, win 8211, length 0

```  

`[S]` 으로 연결이 수행되고 `[P]` 로 요청과 응답을 주고 받았다. 
하지만 `Nginx` 에 `Keepalive` 설정이 돼있기 때문에 연결 종료인 `[F]` 는 수행되지 않았다. 
그리고 `Nginx` 설정에 `keepalive_timeout` 이 `30` 초로 설정돼 있기 때문에 
30초 후에 아래와 같이 `Nginx` 가 `[F]` 로 연결을 종료시키는 것을 확인 할 수 있다.   

```bash
09:10:30.804873 IP nginx.80 > client.58902: Flags [F.], seq 161, ack 519, win 501, length 0
09:10:30.805491 IP client.58902 > nginx.80: Flags [.], ack 162, win 8211, length 0
09:10:45.785395 IP client.58944 > nginx.80: Flags [.], seq 0:1, ack 1, win 8212, length 1: HTTP
09:10:45.785409 IP nginx.80 > client.58944: Flags [.], ack 1, win 502, options [nop,nop,sack 1 {0:1}], length 0
09:11:00.802452 IP nginx.80 > client.58944: Flags [F.], seq 1, ack 1, win 502, length 0
09:11:00.803254 IP client.58944 > nginx.80: Flags [.], ack 2, win 8212, length 0
```  

그리고 `Nginx` 설정에 `keepalive_requests` 를 `3` 으로 설정 했기 때문에 
`Keepalive` 커넥션 하나는 최대 3개의 요청까지만 처리하고 종료 된다. 
연속으로 4번 요청하면 아래와 같은 결과를 확인 할 수 있다.  

```bash
.. 1번째 요청 keepalive 커넥션 연결 ..
09:15:10.595480 IP client.59139 > nginx.80: Flags [S], seq 3376918171, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
09:15:10.595499 IP nginx.80 > client.59139: Flags [S.], seq 3978869574, ack 3376918172, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:15:10.595682 IP client.59140 > nginx.80: Flags [S], seq 2692480824, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
09:15:10.595695 IP client.59139 > nginx.80: Flags [.], ack 1, win 8212, length 0
09:15:10.595696 IP nginx.80 > client.59140: Flags [S.], seq 289045368, ack 2692480825, win 64240, options[mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:15:10.595858 IP client.59139 > nginx.80: Flags [P.], seq 1:519, ack 1, win 8212, length 518: HTTP: GET/test HTTP/1.1
09:15:10.595871 IP client.59140 > nginx.80: Flags [.], ack 1, win 8212, length 0
09:15:10.595908 IP nginx.80 > client.59139: Flags [.], ack 519, win 501, length 0
09:15:10.598057 IP nginx.80 > client.59139: Flags [P.], seq 1:161, ack 519, win 501, length 160: HTTP: HTTP/1.1 200
09:15:10.652185 IP client.59139 > nginx.80: Flags [.], ack 161, win 8211, length 0

.. 2번째 요청 keepalive 커넥션 재사용 ..
09:15:11.578779 IP client.59139 > nginx.80: Flags [P.], seq 519:1037, ack 161, win 8211, length 518: HTTP: GET /test HTTP/1.1
09:15:11.578834 IP nginx.80 > client.59139: Flags [.], ack 1037, win 501, length 0
09:15:11.582776 IP nginx.80 > client.59139: Flags [P.], seq 161:321, ack 1037, win 501, length 160: HTTP:HTTP/1.1 200
09:15:11.627895 IP client.59139 > nginx.80: Flags [.], ack 321, win 8211, length 0

.. 3번째 요청 keepalive 커넥션 종료 ..
09:15:12.563351 IP client.59139 > nginx.80: Flags [P.], seq 1037:1555, ack 321, win 8211, length 518: HTTP: GET /test HTTP/1.1
09:15:12.563390 IP nginx.80 > client.59139: Flags [.], ack 1555, win 501, length 0
09:15:12.565586 IP nginx.80 > client.59139: Flags [P.], seq 321:476, ack 1555, win 501, length 155: HTTP:HTTP/1.1 200
09:15:12.565645 IP nginx.80 > client.59139: Flags [F.], seq 476, ack 1555, win 501, length 0
09:15:12.565855 IP client.59139 > nginx.80: Flags [.], ack 477, win 8210, length 0
09:15:12.566221 IP client.59139 > nginx.80: Flags [F.], seq 1555, ack 477, win 8210, length 0
09:15:12.566229 IP nginx.80 > client.59139: Flags [.], ack 1556, win 501, length 0

.. 4번째 요청 keepalive 커넥션 연결 ..
09:15:13.621621 IP client.59145 > nginx.80: Flags [S], seq 2182012988, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
09:15:13.621637 IP nginx.80 > client.59145: Flags [S.], seq 1461440438, ack 2182012989, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:15:13.621763 IP client.59140 > nginx.80: Flags [P.], seq 1:519, ack 1, win 8212, length 518: HTTP: GET/test HTTP/1.1
09:15:13.621798 IP client.59145 > nginx.80: Flags [.], ack 1, win 8212, length 0
09:15:13.621800 IP nginx.80 > client.59140: Flags [.], ack 519, win 501, length 0
09:15:13.624157 IP nginx.80 > client.59140: Flags [P.], seq 1:161, ack 519, win 501, length 160: HTTP: HTTP/1.1 200
09:15:13.665532 IP client.59140 > nginx.80: Flags [.], ack 161, win 8211, length 0
```  

마지막으로 `keepalive_disable` 에 `safari` 를 설정해서 사파리 브라우저에서는 `Keepalive` 를 
사용하지 않고 연결당 하나의 요청만 처리하도록 설정했다. 
실제 사파리 브라우저에서 요청하면 아래와 같이 하나의 요청만 처리하고 `[F]` 로 연결을 끊는 것을 확인 할 수 있다.  

```bash
09:31:30.871055 IP client.57150 > nginx.80: Flags [S], seq 1351186244, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
09:31:30.871073 IP nginx.80 > client.57150: Flags [S.], seq 1140828745, ack 1351186245, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:31:30.871280 IP client.57150 > nginx.80: Flags [.], ack 1, win 1024, length 0
09:31:30.872371 IP client.57150 > nginx.80: Flags [P.], seq 1:316, ack 1, win 1024, length 315: HTTP: GET/test HTTP/1.1
09:31:30.872380 IP nginx.80 > client.57150: Flags [.], ack 316, win 501, length 0
09:31:30.874515 IP nginx.80 > client.57150: Flags [P.], seq 1:156, ack 316, win 501, length 155: HTTP: HTTP/1.1 200
09:31:30.874565 IP nginx.80 > client.57150: Flags [F.], seq 156, ack 316, win 501, length 0
09:31:30.874711 IP client.57150 > nginx.80: Flags [.], ack 156, win 1023, length 0
09:31:30.874716 IP client.57150 > nginx.80: Flags [F.], seq 316, ack 156, win 1023, length 0
09:31:30.874719 IP nginx.80 > client.57150: Flags [.], ack 317, win 501, length 0
09:31:30.874751 IP client.57150 > nginx.80: Flags [.], ack 157, win 1023, length 0
```  


### Upstream Keepalive
`Nginx-WAS` 간의 `Keepalive` 관련된 `Nginx` 설정 필드는 아래와 같다.  

필드 예시|기본값|설명
---|---|---
keepalive 128;|-|`WAS(upstream)` 의 keepalive 연결을 최대 128개 까지 유지한다. 
keepalive_timeout 30s|keepalive_timeout 60s;|`idle keepalive` 커넥션의 타임아윳 시간을 30초로 설정한다. (30초 동안 처리한 요청이 없으면 커넥션을 닫는다.)
keepalive_request 1000|keepalive_request 100;|하나의 `keepalive` 커넥션이 처리할 수 있는 최대 요청 수  
keepalive_time 50s|keepalive_time 1h;|`keepalive` 커넥션의 최대 유지 시간을 의미한다. 시간이 경과 후 마지막으로 전달된 요청까지 처리하고 커넥션을 종료 한다.   
proxy_http_version 1.1;|proxy_http_version 1.0;|Nginx 는 해당 설정을 하지 않으면 기본 값으로 `HTTP 1.0` 을 사용한다. keepalive 활성화를 위해서는 `HTTP 1.1` 설정이 필요하다. 
proxy_set_header Connection "";|-|응답 헤더에 `Connection` 헤더를 추가로 설정한다. 


`Nginx-WAS` 간 `Keepalive` 를 사용하지 않으면 매 요청당 항상 새로운 커넥션을 생성하고 끊는 작업을 반복한다. 
`Client-Nginx` 간 `Keepalive` 설정이 비효율적이라는 적은 불특정 다수의 요청에 대한 연결 유지 비용이 크고 예측 불가능 하기 때문이였다. 
하지만 `Nginx-WAS` 간 `Keepalive` 는 `keepalive` 설정값으로 최대 연결 개수를 설정 할 수 있기 때문에, 
`Keepalive` 를 사용하게 되면 연결된 커넥션을 재사용 함으로써 활용도가 매우 높다.  

`Nginx-WAS` 에서 `Nginx` 는 `WAS` 쪽으로 연결 요청을 하는 클라이언트와 같다. 
그러기 때문에 이 구간에서 `Keepalive` 에 대한 관리는 `WAS` 의 설정에 따르게 된다. 
`Spring Boot` 를 예시로 `Spring MVC` 즉 `Embedded Tomcat` 을 사용할 때 `Keepalive` 관련 프로퍼티는 아래와 같다.  

프로퍼티|설명
---|---
server.tomcat.connection-timeout: 1s|클라이언트(`Nginx`) 와 연결 타임아웃시간을 설정한다. 
server.tomcat.keep-alive-timeout: 2s|클라이언트(`Nginx`) 와 `Keepalive` 연결 유지 시간을 설정한다. 만약 해당 프로퍼티가 설정되지 않으면 `connection-timeout` 프로퍼티값을 따른다.

---
#### keepalive 동작 테스트
`Nginx-WAS` 의 `Keepalive` 설정은 `Nginx`, `Spring Boot` 설정이 필요하다.
`Keepalive` 설정이 추가된 `nginx.conf`, `application.yaml` 는 아래와 같다.  

```
user  nginx;
worker_processes  1;

events {
    worker_connections  4;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection';

    keepalive_timeout  0;
    #keepalive_disable safari;
    #keepalive_requests 3;

    upstream app {
        server springapp:8080;
        keepalive 1;
        keepalive_timeout 30s;
        keepalive_requests 4;
        keepalive_time 40s;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /logs/error.log warn;

      location / {
          proxy_pass http://app;
          proxy_http_version 1.1;
          proxy_set_header Connection "";
      }

      location = /server-status {
          stub_status on;
          access_log off;
      }
    }
}
```  

> `keepalive_timeout` 필드는 `0` 으로 설정해서 `Client-Nginx` 간에 `Keepalive` 를 사용하지 않도록 설정했다. 
> 그리고 `worker_connections` 도 정확한 테스트를 위해 가능한 최소 값으로 설정 했다.  

```yaml
server:
  tomcat:
    connection-timeout: 1s
    keep-alive-timeout: -1
```  

> `keep-alive-timeout` 프로퍼티를 `-1` 로 설정해서 `Keepalive` 커넥션을 `WAS` 가 끊는 일을 없도록 했다. 


`nginx:test`, `upstream-application` 이미지 빌드후 `Docker Compose` 템플릿을 실행한다.
그리고 `nginx` 컨테이너에 접속해 `8080` 포트로 `tcpdump` 명령을 사용해서 모니터링을 수행한다.  

> `Nginx-WAS` 간에 사용하는 포트는 `8080` 이므로 `tcpdump` 도 `8080` 포트에 대해서 수행한다.  

```bash
$ docker build -t nginx:test .
$ ./gradlew jibDockerBuild
$ docker-compose up --build -d
$ docker exec -it nginx /bin/bash
$ tcpdump port 8080
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes

```  

위 상태에서 `http://<nginx-host>/test` 로 요청을 보내면 아래와 같은 결과를 확인 할 수 있다.

```bash
10:33:15.960771 IP nginx.52638 > was.8080: Flags [S], seq 3529522290, win 64240, options [mss 1460,sackOK,TS val 2666446366 ecr 0,nop,wscale 7], length 0
10:33:15.960894 IP was.8080 > nginx.52638: Flags [S.], seq 38248529, ack 3529522291, win 65160, options [mss 1460,sackOK,TS val 4136271471 ecr 2666446366,nop,wscale 7], length 0
10:33:15.960906 IP nginx.52638 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2666446366 ecr 4136271471], length 0
10:33:15.961098 IP nginx.52638 > was.8080: Flags [P.], seq 1:486, ack 1, win 502, options [nop,nop,TS val 2666446366 ecr 4136271471], length 485: HTTP: GET /test HTTP/1.1
10:33:15.961113 IP was.8080 > nginx.52638: Flags [.], ack 486, win 506, options [nop,nop,TS val 4136271471 ecr 2666446366], length 0
10:33:16.059064 IP was.8080 > nginx.52638: Flags [P.], seq 1:115, ack 486, win 506, options [nop,nop,TS val 4136271569 ecr 2666446366], length 114: HTTP: HTTP/1.1 200
10:33:16.059075 IP nginx.52638 > was.8080: Flags [.], ack 115, win 502, options [nop,nop,TS val 2666446464 ecr 4136271569], length 0
```  

`Nginx` 가 먼저 `WAS` 에게 `[S]` 으로 커넥션 연결을 요청하고 이후 `Nginx` 가 먼저 `[P]` 로 `Client` 의 요청을 전달한다. 
그리고 `WAS` 가 `Nginx` 에게 요청에 대한 응답을 `[P]` 로 전달한다.  

`keepalive_timeout` 이 30초로 설정됐기 때문에 30초 동안 처리하는 요청이 없으면 아래와 같이 `Nginx` 가 먼저 `[F]` 패킷으로 연결 종료 요청을 보내게 된다.  

```bash
10:33:46.062743 IP nginx.52638 > was.8080: Flags [F.], seq 486, ack 115, win 502, options [nop,nop,TS val 2666476468 ecr 4136271569], length 0
10:33:46.066114 IP was.8080 > nginx.52638: Flags [F.], seq 115, ack 487, win 506, options [nop,nop,TS val 4136301576 ecr 2666476468], length 0
10:33:46.066138 IP nginx.52638 > was.8080: Flags [.], ack 116, win 502, options [nop,nop,TS val 2666476471 ecr 4136301576], length 0
```  

이번엔 총 5번의 요청을 연속으로 보낸다. 
`keepalive_requests` 가 4으로 설정 됐기 때문에 4번의 요청까지는 커넥선을 재사용하고 커넥션을 종료한다. 
5번째 요청 부터는 새로운 커넥션을 연결해서 처리하게 된다.  

```bash
.. 1번째 요청 keepalive 커넥션 연결 ..
11:00:47.695011 IP 9e9a32dae052.52694 > was.8080: Flags [S], seq 985514993, win 64240, options [mss 1460,sackOK,TS val 2668098100 ecr 0,nop,wscale 7], length 0
11:00:47.695046 IP was.8080 > 9e9a32dae052.52694: Flags [S.], seq 1123200671, ack 985514994, win 65160, options [mss 1460,sackOK,TS val 4137923205 ecr 2668098100,nop,wscale 7], length 0
11:00:47.695057 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2668098100 ecr 4137923205], length 0
11:00:47.695114 IP 9e9a32dae052.52694 > was.8080: Flags [P.], seq 1:486, ack 1, win 502, options [nop,nop,TS val 2668098100 ecr 4137923205], length 485: HTTP: GET /test HTTP/1.1
11:00:47.695125 IP was.8080 > 9e9a32dae052.52694: Flags [.], ack 486, win 506, options [nop,nop,TS val 4137923205 ecr 2668098100], length 0
11:00:47.698056 IP was.8080 > 9e9a32dae052.52694: Flags [P.], seq 1:115, ack 486, win 506, options [nop,nop,TS val 4137923208 ecr 2668098100], length 114: HTTP: HTTP/1.1 200
11:00:47.698065 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 115, win 502, options [nop,nop,TS val 2668098103 ecr 4137923208], length 0

.. 2번째 요청 keepalive 커넥션 재사용 ..
11:00:50.476456 IP 9e9a32dae052.52694 > was.8080: Flags [P.], seq 486:971, ack 115, win 502, options [nop,nop,TS val 2668100881 ecr 4137923208], length 485: HTTP: GET /test HTTP/1.1
11:00:50.476497 IP was.8080 > 9e9a32dae052.52694: Flags [.], ack 971, win 503, options [nop,nop,TS val 4137925986 ecr 2668100881], length 0
11:00:50.478890 IP was.8080 > 9e9a32dae052.52694: Flags [P.], seq 115:229, ack 971, win 503, options [nop,nop,TS val 4137925989 ecr 2668100881], length 114: HTTP: HTTP/1.1 200
11:00:50.478897 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 229, win 502, options [nop,nop,TS val 2668100884 ecr 4137925989], length 0

.. 3번째 요청 keepalive 커넥션 재사용 ..
11:00:52.451815 IP 9e9a32dae052.52694 > was.8080: Flags [P.], seq 971:1456, ack 229, win 502, options [nop,nop,TS val 2668102857 ecr 4137925989], length 485: HTTP: GET /test HTTP/1.1
11:00:52.451866 IP was.8080 > 9e9a32dae052.52694: Flags [.], ack 1456, win 501, options [nop,nop,TS val 4137927962 ecr 2668102857], length 0
11:00:52.454091 IP was.8080 > 9e9a32dae052.52694: Flags [P.], seq 229:343, ack 1456, win 501, options [nop,nop,TS val 4137927964 ecr 2668102857], length 114: HTTP: HTTP/1.1 200
11:00:52.454101 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 343, win 502, options [nop,nop,TS val 2668102859 ecr 4137927964], length 0

.. 4번째 요청 keepalive 커넥션 종료 ..
11:00:54.343179 IP 9e9a32dae052.52694 > was.8080: Flags [P.], seq 1456:1941, ack 343, win 502, options [nop,nop,TS val 2668104748 ecr 4137927964], length 485: HTTP: GET /test HTTP/1.1
11:00:54.343222 IP was.8080 > 9e9a32dae052.52694: Flags [.], ack 1941, win 501, options [nop,nop,TS val 4137929853 ecr 2668104748], length 0
11:00:54.345324 IP was.8080 > 9e9a32dae052.52694: Flags [P.], seq 343:457, ack 1941, win 501, options [nop,nop,TS val 4137929855 ecr 2668104748], length 114: HTTP: HTTP/1.1 200
11:00:54.345331 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 457, win 502, options [nop,nop,TS val 2668104750 ecr 4137929855], length 0
11:00:54.345445 IP 9e9a32dae052.52694 > was.8080: Flags [F.], seq 1941, ack 457, win 502, options [nop,nop,TS val 2668104750 ecr 4137929855], length 0
11:00:54.345770 IP was.8080 > 9e9a32dae052.52694: Flags [F.], seq 457, ack 1942, win 501, options [nop,nop,TS val 4137929856 ecr 2668104750], length 0
11:00:54.345778 IP 9e9a32dae052.52694 > was.8080: Flags [.], ack 458, win 502, options [nop,nop,TS val 2668104751 ecr 4137929856], length 0

.. 5번째 요청 keepalive 커넥션 연결 ..
11:00:56.273276 IP 9e9a32dae052.52696 > was.8080: Flags [S], seq 3149136942, win 64240, options [mss 1460,sackOK,TS val 2668106678 ecr 0,nop,wscale 7], length 0
11:00:56.273297 IP was.8080 > 9e9a32dae052.52696: Flags [S.], seq 1717799122, ack 3149136943, win 65160, options [mss 1460,sackOK,TS val 4137931783 ecr 2668106678,nop,wscale 7], length 0
11:00:56.273304 IP 9e9a32dae052.52696 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2668106678 ecr 4137931783], length 0
11:00:56.273354 IP 9e9a32dae052.52696 > was.8080: Flags [P.], seq 1:486, ack 1, win 502, options [nop,nop,TS val 2668106678 ecr 4137931783], length 485: HTTP: GET /test HTTP/1.1
11:00:56.273363 IP was.8080 > 9e9a32dae052.52696: Flags [.], ack 486, win 506, options [nop,nop,TS val 4137931783 ecr 2668106678], length 0
11:00:56.275809 IP was.8080 > 9e9a32dae052.52696: Flags [P.], seq 1:115, ack 486, win 506, options [nop,nop,TS val 4137931786 ecr 2668106678], length 114: HTTP: HTTP/1.1 200
11:00:56.275817 IP 9e9a32dae052.52696 > was.8080: Flags [.], ack 115, win 502, options [nop,nop,TS val 2668106681 ecr 4137931786], length 0
```  

`keepalive_time` 은 `Keepalive` 커넥션의 최대 유지 시간을 의미한다. 
20초 단위로 요청을 총 3번 보내게 되면, 
아래와 같이 아직 `keepalive_requests` 와 `keepalive_timeout` 에 만족하지 않지만 
3번째 요청까지만 처리하고 연결이 종료되는 것을 확인 할 수 있다.  

```bash
.. 1번째 요청 keepalive_time 시작 ..
10:58:50.686860 IP 9e9a32dae052.52692 > was.8080: Flags [S], seq 2469474786, win 64240, options [mss 1460,sackOK,TS val 2667981092 ecr 0,nop,wscale 7], length 0
10:58:50.686885 IP was.8080 > 9e9a32dae052.52692: Flags [S.], seq 4124222573, ack 2469474787, win 65160, options [mss 1460,sackOK,TS val 4137806197 ecr 2667981092,nop,wscale 7], length 0
10:58:50.686893 IP 9e9a32dae052.52692 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2667981092 ecr 4137806197], length 0
10:58:50.686951 IP 9e9a32dae052.52692 > was.8080: Flags [P.], seq 1:486, ack 1, win 502, options [nop,nop,TS val 2667981092 ecr 4137806197], length 485: HTTP: GET /test HTTP/1.1
10:58:50.686961 IP was.8080 > 9e9a32dae052.52692: Flags [.], ack 486, win 506, options [nop,nop,TS val 4137806197 ecr 2667981092], length 0
10:58:50.789708 IP was.8080 > 9e9a32dae052.52692: Flags [P.], seq 1:115, ack 486, win 506, options [nop,nop,TS val 4137806300 ecr 2667981092], length 114: HTTP: HTTP/1.1 200
10:58:50.789745 IP 9e9a32dae052.52692 > was.8080: Flags [.], ack 115, win 502, options [nop,nop,TS val 2667981195 ecr 4137806300], length 0

.. 2번째 요청 keepalive_time 24초 경과 ..
10:59:14.893237 IP 9e9a32dae052.52692 > was.8080: Flags [P.], seq 486:971, ack 115, win 502, options [nop,nop,TS val 2668005298 ecr 4137806300], length 485: HTTP: GET /test HTTP/1.1
10:59:14.893287 IP was.8080 > 9e9a32dae052.52692: Flags [.], ack 971, win 503, options [nop,nop,TS val 4137830403 ecr 2668005298], length 0
10:59:14.896223 IP was.8080 > 9e9a32dae052.52692: Flags [P.], seq 115:229, ack 971, win 503, options [nop,nop,TS val 4137830406 ecr 2668005298], length 114: HTTP: HTTP/1.1 200
10:59:14.896231 IP 9e9a32dae052.52692 > was.8080: Flags [.], ack 229, win 502, options [nop,nop,TS val 2668005301 ecr 4137830406], length 0

.. 3번째 요청 keepalive_time 44초 경과이므로 해당 요청까지 처리하고 종료 ..
10:59:34.863908 IP 9e9a32dae052.52692 > was.8080: Flags [P.], seq 971:1456, ack 229, win 502, options [nop,nop,TS val 2668025269 ecr 4137830406], length 485: HTTP: GET /test HTTP/1.1
10:59:34.863970 IP was.8080 > 9e9a32dae052.52692: Flags [.], ack 1456, win 501, options [nop,nop,TS val 4137850374 ecr 2668025269], length 0
10:59:34.866530 IP was.8080 > 9e9a32dae052.52692: Flags [P.], seq 229:343, ack 1456, win 501, options [nop,nop,TS val 4137850377 ecr 2668025269], length 114: HTTP: HTTP/1.1 200
10:59:34.866537 IP 9e9a32dae052.52692 > was.8080: Flags [.], ack 343, win 502, options [nop,nop,TS val 2668025272 ecr 4137850377], length 0
10:59:34.866612 IP 9e9a32dae052.52692 > was.8080: Flags [F.], seq 1456, ack 343, win 502, options [nop,nop,TS val 2668025272 ecr 4137850377], length 0
10:59:34.867703 IP was.8080 > 9e9a32dae052.52692: Flags [F.], seq 343, ack 1457, win 501, options [nop,nop,TS val 4137850378 ecr 2668025272], length 0
10:59:34.867711 IP 9e9a32dae052.52692 > was.8080: Flags [.], ack 344, win 502, options [nop,nop,TS val 2668025273 ecr 4137850378], length 0
```  


---
#### 주의 사항 테스트
`Keepalive` 에 대한 타임아웃은 클라이언트, 서버에서 모두 설정가능하다. 
여기서 주의사항이 있는데 클라이언트의 `Keepalive` 타임아웃 시간보다 서버 `Keepalive` 타임아웃 시간이 더 길어야 한다는 것이다. 
만약 같거나 작으면 서버가 먼저 클라이언트와 연결된 `Keepalive` 연결을 끊어 버릴 수 있기 때문에 에러가 발생 할 수 있다. 

> 연결과 연결 해제는 클라이언트의 요청으로 진행 돼야 비교적 안전한 처리가 가능하다. 
> 위 `Keepalive 동작 테스트` 에서 `WAS` 의 `Keepalive timeout` 을 `-1`(무한대)로 설정한 이유도 그때문이다. 

`Nginx-WAS` 의 `Keepalive` 에서 위 상황일 때 어떠한 에러가 발생하는지 살펴 본다. 
테스트에 필요한 `nginx.conf`, `application.yaml` 은 아래와 같다.  

```
user  nginx;
worker_processes  8;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection';

    keepalive_timeout  0;
    #keepalive_disable safari;
    #keepalive_requests 3;

    upstream app {
        server springapp:8080;
        keepalive 128;
        keepalive_timeout 30s;
        keepalive_requests 4;
        keepalive_time 40s;
    }
    
    .. 이하 동일 ..
}
```  

```yaml
server:
  tomcat:
    keep-alive-timeout: 1s # nginx 의 keepalive_timeout 30s 보다 적은 값으로 설정
```  

해당 테스트도 아래 명령어처럼 `8080` 포트에 대해서 `tcpdump` 를 수행한다.  

```bash
$ tcpdump port 8080 -w dump
.. tcpdump 결과를 dump 라는 파일에 저장한다. ..
```  

요청 한번을 보내고 20초 동안 기다리면 아래와 같이, 
`WAS` 에서 `Nginx` 쪽으로 연결을 끊기 위해 `[F]` 을 보내는 것을 확인 할 수 있다. 
아래 결과는 `WAS` 에서 먼저 연결해제 요청을 했지만 에러는 발생하지 않고 정상적으로 연결이 종료된 경우이다.  

```bash
11:14:19.953158 IP nginx.34544 > was.8080: Flags [S], seq 225645460, win 64240, options [mss 1460,sackOK,TS val 4138735463 ecr 0,nop,wscale 7], length 0
11:14:19.953175 IP was.8080 > nginx.34544: Flags [S.], seq 2916224507, ack 225645461, win 65160, options [mss 1460,sackOK,TS val 2668910358 ecr 4138735463,nop,wscale 7], length 0
11:14:19.953180 IP nginx.34544 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 4138735463 ecr 2668910358], length 0
11:14:19.953227 IP nginx.34544 > was.8080: Flags [P.], seq 1:486, ack 1, win 502, options [nop,nop,TS val 4138735463 ecr 2668910358], length 485: HTTP: GET /test HTTP/1.1
11:14:19.953237 IP was.8080 > nginx.34544: Flags [.], ack 486, win 506, options [nop,nop,TS val 2668910358 ecr 4138735463], length 0
11:14:20.027973 IP was.8080 > nginx.34544: Flags [P.], seq 1:115, ack 486, win 506, options [nop,nop,TS val 2668910433 ecr 4138735463], length 114: HTTP: HTTP/1.1 200
11:14:20.027981 IP nginx.34544 > was.8080: Flags [.], ack 115, win 502, options [nop,nop,TS val 4138735538 ecr 2668910433], length 0
11:14:40.064235 IP was.8080 > nginx.34544: Flags [F.], seq 115, ack 486, win 506, options [nop,nop,TS val 2668930469 ecr 4138735538], length 0
11:14:40.064504 IP nginx.34544 > was.8080: Flags [F.], seq 486, ack 116, win 502, options [nop,nop,TS val 4138755574 ecr 2668930469], length 0
11:14:40.064564 IP was.8080 > nginx.34544: Flags [.], ack 487, win 506, options [nop,nop,TS val 2668930470 ecr 4138755574], length 0
```  

이번엔 몇번의 요청이 아니라 `JMeter`, `ab` 명령어 등으로 반복적인 다수의 요청을 보내본다. 
그러다 보면 `Nginx` 에러 로그에 아래와 같은 메시지를 확인 할 수 있다.  

```bash
11:41:25 [error] 39#39: *1371 upstream prematurely closed connection while reading response header from upstream, client: 127.0.0.1, server: , request: "GET /test/500/50000 HTTP/1.0", upstream: "http://172.20.0.2:8080/test/500/50000", host: "localhost"

or 

11:41:25 [error] 39#39: *1371 upstream prematurely closed connection while reading upstream, client: 127.0.0.1, server: , request: "GET /test/500/50000 HTTP/1.0", upstream: "http://172.20.0.2:8080/test/500/50000", host: "localhost"
```  

위 시간대에 해당하는 `tcpdump` 를 확인하면 아래와 같다.  

```bash
root@nginx:/# tcpdump -r dump src port 48236 or dst port 48236
reading from file dump, link-type EN10MB (Ethernet), snapshot length 262144

.. keepalive 연결 요청 ..
11:41:03.160295 IP nginx.48236 > was.8080: Flags [S], seq 3679230282, win 64240, options [mss 1460,sackOK,TS val 2670513565 ecr 0,nop,wscale 7], length 0
11:41:03.160307 IP was.8080 > nginx.48236: Flags [S.], seq 883316901, ack 3679230283, win 65160, options [mss 1460,sackOK,TS val 4140338670 ecr 2670513565,nop,wscale 7], length 0
11:41:03.160312 IP nginx.48236 > was.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2670513565 ecr 4140338670], length 0

.. client 요청 전달 ..
11:41:03.160326 IP nginx.48236 > was.8080: Flags [P.], seq 1:86, ack 1, win 502, options [nop,nop,TS val 2670513565 ecr 4140338670], length 85: HTTP: GET /test/500/50000 HTTP/1.1
11:41:03.160332 IP was.8080 > nginx.48236: Flags [.], ack 86, win 509, options [nop,nop,TS val 4140338670 ecr 2670513565], length 0

.. client 요청에 대한 응답 전달 ..
11:41:12.214514 IP was.8080 > nginx.48236: Flags [P.], seq 1:7241, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670513565], length 7240: HTTP: HTTP/1.1 200
11:41:12.214523 IP nginx.48236 > was.8080: Flags [.], ack 7241, win 479, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214533 IP was.8080 > nginx.48236: Flags [P.], seq 7241:8193, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670513565], length 952: HTTP
11:41:12.214535 IP nginx.48236 > was.8080: Flags [.], ack 8193, win 472, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214621 IP was.8080 > nginx.48236: Flags [P.], seq 8193:16385, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 8192: HTTP
11:41:12.214624 IP nginx.48236 > was.8080: Flags [.], ack 16385, win 475, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214704 IP was.8080 > nginx.48236: Flags [P.], seq 16385:24577, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 8192: HTTP
11:41:12.214708 IP nginx.48236 > was.8080: Flags [.], ack 24577, win 443, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214760 IP was.8080 > nginx.48236: Flags [P.], seq 24577:32769, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 8192: HTTP
11:41:12.214763 IP nginx.48236 > was.8080: Flags [.], ack 32769, win 629, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214815 IP was.8080 > nginx.48236: Flags [P.], seq 32769:40961, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 8192: HTTP
11:41:12.214818 IP nginx.48236 > was.8080: Flags [.], ack 40961, win 757, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214889 IP was.8080 > nginx.48236: Flags [P.], seq 40961:49153, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 8192: HTTP
11:41:12.214892 IP nginx.48236 > was.8080: Flags [.], ack 49153, win 885, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:12.214964 IP was.8080 > nginx.48236: Flags [P.], seq 49153:50123, ack 86, win 509, options [nop,nop,TS val 4140347725 ecr 2670522620], length 970: HTTP
11:41:12.214967 IP nginx.48236 > was.8080: Flags [.], ack 50123, win 908, options [nop,nop,TS val 2670522620 ecr 4140347725], length 0
11:41:13.725148 IP nginx.48236 > was.8080: Flags [P.], seq 86:171, ack 50123, win 908, options [nop,nop,TS val 2670524130 ecr 4140347725], length 85: HTTP: GET /test/500/50000 HTTP/1.1
11:41:13.725162 IP was.8080 > nginx.48236: Flags [.], ack 171, win 509, options [nop,nop,TS val 4140349235 ecr 2670524130], length 0

.. was 가 nginx 에게 연결 종료 요청 ..
11:41:25.796792 IP was.8080 > nginx.48236: Flags [F.], seq 50123, ack 171, win 509, options [nop,nop,TS val 4140361307 ecr 2670524130], length 0
.. nginx 가 응답없자(요청 처리 중) 강제 연결 종료 ..
11:41:25.796866 IP was.8080 > nginx.48236: Flags [R.], seq 50124, ack 171, win 509, options [nop,nop,TS val 4140361307 ecr 2670524130], length 0
```  


`upstream prematurely closed connection while reading ...` 에러가 발생하는 이유는 
`tcpdump` 결과를 보면 알겠지만 잘못된 `keepalive` 설정이 원인이다. 
`Client - Nginx - WAS` 의 관계에서 `Nginx` 가 `WAS` 에게 요청을 전달하고 응답을 받고 있는 도중 
`WAS` 가 잘못된 `Keepalive` 타임아웃 설정으로 `[R]` 패킷을 통해 강제로 연결을 끊어 발생하는 것이다. 
일반적으로 해당 경우에는 다시 요청이 수행되므로 500 에러는 발생하지 않고 200으로 정상 응답이 `Client` 에게 전달 된다.  

만약 `upstream prematurely closed connection while reading ...` 에러가 발생한다면, 
`Nginx-WAS` 간의 `Keepalive` 타임아웃 설정을 `WAS` 쪽이 더 길거나 무한대로 설정 하는 방법으로 해결 할 수 있다. 
혹은 `Keepalive` 를 아예 사용하지 않도록 해도 에러는 발생하지 않는다.  



---
#### 성능 테스트

......




---
## Reference
[What is keepalive in Nginx](https://linuxhint.com/what-is-keepalive-in-nginx/)  
[Module ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html)  
[Module ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)  
[Module ngx_http_upstream_module](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)  
    