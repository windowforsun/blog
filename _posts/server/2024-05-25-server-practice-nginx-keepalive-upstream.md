--- 
layout: single
classes: wide
title: "[Nginx] Nginx Upstream(Spring) Keepalive"
header:
  overlay_image: /img/server-bg.jpg
excerpt: 'Nginx 와 Spring Boot Web 에서 Keepalive 을 구성할 때 필요한 주요한 설정들에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Spring
  - Spring Boot
  - Spring Boot Web
  - Nginx
  - Keepalive
toc: true
use_math: true
---  

## Nginx Upstream Keepalive
[Nginx Keepalive]({{site.baseurl}}{% link _posts/server/2022-04-02-server-practice-nginx-keepalive.md %})
에서 Nginx Keepalive 설정에 대한 내용을 간략하게 알아보고 간단한 테스트도 진행해 보았다. 
이번에는 좀더 나아가 `Nginx` 에서 `Upstream` 에 대한 `Keepalive` 을 설정할 때, 
좀 더 최적화된 설정이 필요로 할 때 고민해 볼 수 있는 부분들에 대해 알아보고자 한다.  

앞선 포스팅에서 언급했던 것 처럼, `Keepalive` 설정이란 
`Client-Server` 관계에서 `Server` 가 `Keepalive` 를 지원해야 사용할 수 있는 기능이다. 
그러므로 `Nginx` 는 `Client` 가 되는 것이고, `Nginx` 에 `Upstream` 으로 등록된 대상이 `Server` 가 된다.  

또한 `Client-Server` 에서 안전한 연결과 연결해제는 `Client` 가 `Server` 에게 먼저 요청해서 수행돼야함을 기억해야한다. 
`3-Way`, `4-Way Handshake` 에서 연결을 맺을 때 보내는 `S(Sync)` 패킷과 연결해제를 위해 보내는 
`F(Fin)` 패킷은 클라이언트가 먼저 보내 연결을 안정적으로 끊어야 한다는 것이다.  

practice-nginx-keepalive-5.png

예제는 앞선 포스팅과 마찬가지로 `Nginx` 와 `Upstream` 으로는 `Spring Boot Web`(Tomcat) 을 사용할 것이다. 
`Keepalive` 설정은 `Upstream` 의 설정 또한 중요하기 때문에 
`Keepalive` 관련 `Nginx` 설정 뿐만아니라, `Spring Boot Web` 의 설정에 대해서도 알아볼 것이다. 
예제 구성은 앞선 포스팅과 동일하고 `Nginx` 설정값과 `Spring Boot Web` 관련 설정만 수정해가며 테스트를 진행할 것이다.  

예제를 통한 테스트를 보기에 앞서 먼저 `Nginx` 와 `Spring Boot Web`(Tomcat) 에서 `Keepalive` 사용과 어느정도 관련있는 
설정 값들을 나열하면 아래와 같다. 

| 구분     | 설정 필드             | 기본값  |설명
|--------|-------------------|------|---
|  Nginx | keepalive         | -    |Nginx 의 worker 당 Upstream 과 유지할 Keepalive connections 의 수를 설정한다.
| | keepalive_requests | 1000 |Upstream 과 연결한 Keepalive connection 에서 사용할 최대 요청 수이다. 요청 수를 넘어가면 해당 커넥션은 닫힌다. 
| | keepalive_time    | 1h   |Upstream 과 연결한 Keepalive connection 이 최대로 사용될 수 있는 시간이다. 설정된 시간이 넘어가면 해당 커넥션은 닫힌다. 
| | keepalive_timeout | 60s  |Upstream 과 연결한 Keepalive connection 이 `idle` 상태로 유지될 수 있는 시간이다. 설정된 시간동안 해당 커넥션을 사용하지 않으면 커넥션은 닫힌다. 
| | worker_processes  | 1    |Nginx instance 가 요청 처리를 위해 실행시키는 프로세스의 수이다. 그리고 Upstream 과 생성되는 Keepalive connection 총 개수는 worker_processes * keepalive 이다.  
| | worker_connections | 512  |Nginx worker 당 최대로 생성가능한 동시 커넥션 수이다. Client, Server 의 모든 connection 이 포함되고, Keepalive 에 설정된 값보다 작아선 안된다. 
|Spring Boot Web|server.tomcat.max-connections| 8192 | Tomcat 이 동시에 열수 있는 최대 연결 수이다. Nginx 에서 연결하는 Keepalive connection 수보다 같거나 많아야 한다.
| |server.tomcat.threads.max| 200  |Tomcat 이 동시에 처리할 수 있는 요청의 수로, Keeaplive connection 을 적절하게 처리할 수 있는 같거나 보다 큰 값으로 설정하는 것이 좋을 수 있다. 
| |server.tomcat.threads.min-spare| 10   |Tomcat 이 최소한으로 유지하는 요청 수로, Keepalive connection 총 개수와 같거나 작게 설정하는 것이 좋다. 
| |server.tomcat.keep-alive-timeout| -    | Keepalive connection 을 최대로 유지할 시간을 설정한다. Nginx 의 keepalive_time 보다 커야한다. (-1=unlimited)
| |server.tomcat.max-keep-alive-requests| 100  |Keepalive connection 당 처리할 최대 요청 수이다. -1 로 하면 무제한이고, Nginx 의 keepalive_requests 보다 커야한다. 
| |server.tomcat.accept-count|100|Tomcat 의 모든 스레드가 사용 중일 때, 연결요청의 대기열 길이이다. 큐가 가득 차면, 새 연결은 거부한다. 

이러한 두 구성간 설정이 맞지 않다고 하더라도 일반적인 상황에서 오류나 이슈상황이 발생하진 않을 수 있다. 
하지만 두 구성간 설정이 맞지 않기 때문에 서버가 부팅된 직후라던가 요청이 몰리는 상황에서는 
예기치 못한 에러나 이슈가 발생할 수 있음을 기억해야 한다.  

이후 부터는 `Nginx` 와 `Spring Boot Web` 의 각 옵션이 서로 호환되지 않는 설정일 때, 
어떠한 증상이 발생하는지 몇가지 예시를 바탕으로 알아보고자 한다.  


### 테스트 방식
테스트 환경과 예제는 [여기](https://windowforsun.github.io/blog/server/server-practice-nginx-keepalive/#%ED%85%8C%EC%8A%A4%ED%8A%B8-%ED%99%98%EA%B2%BD)
에서 확인 할 수 있다.  

테스트 조건마다 `nginx.conf` 에서 `Nginx` 설정을 변경하고 
`application.yaml` 을 통해 `Tomcat` 설정을 변경해서 진행할 예정이다. 
그리고 테스트에 사용되는 요청 스크립트는 아래와 같다. 

```bash
#!/bin/bash
# test.sh

# 요청 텀
shell_sleep=$1
# 응답 대기시간
request_sleep=$2

echo "shell_sleep ${shell_sleep}"
echo "request_sleep ${request_sleep}"

for i in {1..10}; do
  curl localhost/test/${request_sleep}/1 &
  sleep ${shell_sleep}
done

wait

echo "done!!"
```  

위 스크립트를 `. ./test.sh 1 1000` 와 같이 호출해 총 10번 요청에 대해
`Nginx` 와 `Upstream` 간의 `Keeaplive` 커넥션이 어떻게 사용됐는지 `tcpdump` 로 확인해 볼 것이다. 
`tcpdump port 8080 -w dump` 와 같이 `Spring Boot Web` 포트인 `8080` 와 통신하는 패킷 통신을 
`dump` 파일 파일에 저장하도록 지정하고 테스트가 끝난 후 증상 확인 및 검증으로 사용한다.  



### All Ok
가장 먼저 `Nginx` 와 `Spring Boot Web` 의 `Keepalive` 설정이 호환돼 문제없이 
모두 정상 동작 하는 경우를 먼저 살펴본다. 

| 구분     | 설정 필드             | 설정값
|--------|-------------------|------
|  Nginx | keepalive         | 1    
| | keepalive_requests | 3    
| | keepalive_time    | 10s   
| | keepalive_timeout | 9s   
| | worker_processes  | 2    
| | worker_connections | 1024 
|Spring Boot Web|server.tomcat.max-connections| 8 
| |server.tomcat.threads.max| 4   
| |server.tomcat.threads.min-spare| 2
| |server.tomcat.keep-alive-timeout| 20s 
| |server.tomcat.max-keep-alive-requests| 20
| |server.tomcat.accept-count| 1  

실제 설정파일 내용은 아래와 같다. 
전반적인 설정을 보면 `Spring Boot` 의 설정값이 `Nginx` 보다 좀 더 크게 설정 돼 있음을 알 수 있다. 
더 큰 `Timeout` 값을 가지고, 더 많은 요청과 커넥션을 허용하는 것이다. 
`Nginx` 와 `Spring Boot` 간 연결이 `Spring Boot` 가 아닌 `Nginx` 에서 끊기도록 해야 하기 때문이다.   


```
# nginx-keepalive-allok.conf

user  nginx;
worker_processes  2;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection $connection_requests';

    keepalive_timeout  0;

    upstream app {
        server springapp:8080;
        keepalive 1;
        keepalive_time 10s;
        keepalive_timeout 9s;
        keepalive_requests 3;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /logs/error.log warn;

      location / {
          proxy_connect_timeout 1s;
          proxy_send_timeout 1s;
          proxy_read_timeout 2s;
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

```properties
# applicatio-allok.yaml

server:
  http2:
    enabled: true
  tomcat:
    threads:
      max: 4
      min-spare: 1
    connection-timeout: 1s
    keep-alive-timeout: 20s
    max-keep-alive-requests: 20
    max-connections: 8
    accept-count: 1
```  

초당 1개의 요청을 10번 보냈을 떄 테스트 결과를 보면 
총 5개의 커넥션이 사용된 것을 확인 할 수 있다. 
이는 매번 5개의 커넥션이 사용되는 것은 아니고 상황이나 환경마다 달라 질 수 있다. 
그리고 각 커넥션은 `Nginx` 의 요청으로 부터 `Keepalive Connection` 이 맺어지고, 
설정된 값만큼 사용후 `Nginx` 의 요청으로 부터 커넥션이 끊겨야 한다. 
총 5개의 커넥션이 어떻게 사용됐는지 확인해 보면 아래와 같다.  


```bash
$ tcpdump -r dump | grep 49650

... 1번째 Keepalive Connection 연결 ...
09:57:56.721096 IP nginx.49650 > springapp.docker_test-net.8080: Flags [S], seq 2924545024, win 64240, options [mss 1460,sackOK,TS val 3471470170 ecr 0,nop,wscale 7], length 0
09:57:56.721122 IP springapp.docker_test-net.8080 > nginx.49650: Flags [S.], seq 1922386706, ack 2924545025, win 65160, options [mss 1460,sackOK,TS val 2146456756 ecr 3471470170,nop,wscale 7], length 0
09:57:56.721127 IP nginx.49650 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 3471470170 ecr 2146456756], length 0

... 1번째 요청 ...
09:57:56.721161 IP nginx.49650 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 3471470170 ecr 2146456756], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:57:56.721166 IP springapp.docker_test-net.8080 > nginx.49650: Flags [.], ack 78, win 509, options [nop,nop,TS val 2146456756 ecr 3471470170], length 0
09:57:57.741509 IP springapp.docker_test-net.8080 > nginx.49650: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 2146457776 ecr 3471470170], length 120: HTTP: HTTP/1.1 200 
09:57:57.741648 IP nginx.49650 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 3471471191 ecr 2146457776], length 0

... 2번째 요청 ...
09:57:57.753732 IP nginx.49650 > springapp.docker_test-net.8080: Flags [P.], seq 78:155, ack 121, win 502, options [nop,nop,TS val 3471471203 ecr 2146457776], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:57:57.753804 IP springapp.docker_test-net.8080 > nginx.49650: Flags [.], ack 155, win 509, options [nop,nop,TS val 2146457789 ecr 3471471203], length 0
09:57:58.762324 IP springapp.docker_test-net.8080 > nginx.49650: Flags [P.], seq 121:241, ack 155, win 509, options [nop,nop,TS val 2146458797 ecr 3471471203], length 120: HTTP: HTTP/1.1 200 
09:57:58.810870 IP nginx.49650 > springapp.docker_test-net.8080: Flags [.], ack 241, win 502, options [nop,nop,TS val 3471472260 ecr 2146458797], length 0

... 1번째 Keepalive Connection 해제 ...
09:57:59.773152 IP nginx.49650 > springapp.docker_test-net.8080: Flags [F.], seq 155, ack 241, win 502, options [nop,nop,TS val 3471473222 ecr 2146458797], length 0
09:57:59.781543 IP springapp.docker_test-net.8080 > nginx.49650: Flags [F.], seq 241, ack 156, win 509, options [nop,nop,TS val 2146459817 ecr 3471473222], length 0
09:57:59.781577 IP nginx.49650 > springapp.docker_test-net.8080: Flags [.], ack 242, win 502, options [nop,nop,TS val 3471473231 ecr 2146459817], length 0
```

```bash
$ tcpdump -r dump | grep 49656

... 2번째 Keepalive Connection 연결 ...
09:57:58.760953 IP nginx.49656 > springapp.docker_test-net.8080: Flags [S], seq 2309111746, win 64240, options [mss 1460,sackOK,TS val 3471472210 ecr 0,nop,wscale 7], length 0
09:57:58.760988 IP springapp.docker_test-net.8080 > nginx.49656: Flags [S.], seq 1346328569, ack 2309111747, win 65160, options [mss 1460,sackOK,TS val 2146458796 ecr 3471472210,nop,wscale 7], length 0
09:57:58.760993 IP nginx.49656 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 3471472210 ecr 2146458796], length 0

... 3번째 요청 ...
09:57:58.761050 IP nginx.49656 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 3471472210 ecr 2146458796], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:57:58.761057 IP springapp.docker_test-net.8080 > nginx.49656: Flags [.], ack 78, win 509, options [nop,nop,TS val 2146458796 ecr 3471472210], length 0
09:57:59.771338 IP springapp.docker_test-net.8080 > nginx.49656: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 2146459806 ecr 3471472210], length 120: HTTP: HTTP/1.1 200 
09:57:59.771438 IP nginx.49656 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 3471473221 ecr 2146459806], length 0

... 4번째 요청 ...
09:57:59.784171 IP nginx.49656 > springapp.docker_test-net.8080: Flags [P.], seq 78:155, ack 121, win 502, options [nop,nop,TS val 3471473233 ecr 2146459806], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:57:59.784210 IP springapp.docker_test-net.8080 > nginx.49656: Flags [.], ack 155, win 509, options [nop,nop,TS val 2146459819 ecr 3471473233], length 0
09:58:00.800338 IP springapp.docker_test-net.8080 > nginx.49656: Flags [P.], seq 121:241, ack 155, win 509, options [nop,nop,TS val 2146460835 ecr 3471473233], length 120: HTTP: HTTP/1.1 200 
09:58:00.850417 IP nginx.49656 > springapp.docker_test-net.8080: Flags [.], ack 241, win 502, options [nop,nop,TS val 3471474300 ecr 2146460835], length 0

... 5번째 요청 ...
09:58:01.807219 IP nginx.49656 > springapp.docker_test-net.8080: Flags [P.], seq 155:232, ack 241, win 502, options [nop,nop,TS val 3471475256 ecr 2146460835], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:01.807311 IP springapp.docker_test-net.8080 > nginx.49656: Flags [.], ack 232, win 509, options [nop,nop,TS val 2146461842 ecr 3471475256], length 0
09:58:02.819075 IP springapp.docker_test-net.8080 > nginx.49656: Flags [P.], seq 241:361, ack 232, win 509, options [nop,nop,TS val 2146462854 ecr 3471475256], length 120: HTTP: HTTP/1.1 200 
09:58:02.832071 IP nginx.49656 > springapp.docker_test-net.8080: Flags [.], ack 361, win 502, options [nop,nop,TS val 3471476268 ecr 2146462854], length 0


... 2번째 Keepalive Connection 해제 ...
09:58:02.832154 IP nginx.49656 > springapp.docker_test-net.8080: Flags [F.], seq 232, ack 361, win 502, options [nop,nop,TS val 3471476281 ecr 2146462854], length 0
09:58:02.833469 IP springapp.docker_test-net.8080 > nginx.49656: Flags [F.], seq 361, ack 233, win 509, options [nop,nop,TS val 2146462869 ecr 3471476281], length 0
09:58:02.833477 IP nginx.49656 > springapp.docker_test-net.8080: Flags [.], ack 362, win 502, options [nop,nop,TS val 3471476283 ecr 2146462869], length 0
```  

```bash
$ tcpdump -r dump | grep 40074

... 3번째 Keepalive Connection 연결 ...
09:58:00.803097 IP nginx.40074 > springapp.docker_test-net.8080: Flags [S], seq 3891372270, win 64240, options [mss 1460,sackOK,TS val 3471474252 ecr 0,nop,wscale 7], length 0
09:58:00.803251 IP springapp.docker_test-net.8080 > nginx.40074: Flags [S.], seq 2044650708, ack 3891372271, win 65160, options [mss 1460,sackOK,TS val 2146460838 ecr 3471474252,nop,wscale 7], length 0
09:58:00.803279 IP nginx.40074 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 3471474252 ecr 2146460838], length 0

... 6번째 요청 ...
09:58:00.803688 IP nginx.40074 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 3471474253 ecr 2146460838], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:00.803697 IP springapp.docker_test-net.8080 > nginx.40074: Flags [.], ack 78, win 509, options [nop,nop,TS val 2146460839 ecr 3471474253], length 0
09:58:01.823639 IP springapp.docker_test-net.8080 > nginx.40074: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 2146461859 ecr 3471474253], length 120: HTTP: HTTP/1.1 200 
09:58:01.823719 IP nginx.40074 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 3471475273 ecr 2146461859], length 0

... 7번째 요청 ...
09:58:02.832967 IP nginx.40074 > springapp.docker_test-net.8080: Flags [P.], seq 78:155, ack 121, win 502, options [nop,nop,TS val 3471476282 ecr 2146461859], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:02.833023 IP springapp.docker_test-net.8080 > nginx.40074: Flags [.], ack 155, win 509, options [nop,nop,TS val 2146462868 ecr 3471476282], length 0
09:58:03.848885 IP springapp.docker_test-net.8080 > nginx.40074: Flags [P.], seq 121:241, ack 155, win 509, options [nop,nop,TS val 2146463884 ecr 3471476282], length 120: HTTP: HTTP/1.1 200 
09:58:03.848961 IP nginx.40074 > springapp.docker_test-net.8080: Flags [.], ack 241, win 502, options [nop,nop,TS val 3471477298 ecr 2146463884], length 0

... 8번째 요청 ...
09:58:03.858334 IP nginx.40074 > springapp.docker_test-net.8080: Flags [P.], seq 155:232, ack 241, win 502, options [nop,nop,TS val 3471477307 ecr 2146463884], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:03.858381 IP springapp.docker_test-net.8080 > nginx.40074: Flags [.], ack 232, win 509, options [nop,nop,TS val 2146463894 ecr 3471477307], length 0
09:58:04.866953 IP springapp.docker_test-net.8080 > nginx.40074: Flags [P.], seq 241:361, ack 232, win 509, options [nop,nop,TS val 2146464902 ecr 3471477307], length 120: HTTP: HTTP/1.1 200 

... 3번째 Keepalive Connection 연결 ...
09:58:04.868835 IP nginx.40074 > springapp.docker_test-net.8080: Flags [F.], seq 232, ack 361, win 502, options [nop,nop,TS val 3471478318 ecr 2146464902], length 0
09:58:04.870655 IP springapp.docker_test-net.8080 > nginx.40074: Flags [F.], seq 361, ack 233, win 509, options [nop,nop,TS val 2146464906 ecr 3471478318], length 0
09:58:04.870670 IP nginx.40074 > springapp.docker_test-net.8080: Flags [.], ack 362, win 502, options [nop,nop,TS val 3471478320 ecr 2146464906], length 0
```  


```bash
$ tcpdump -r dump | grep 40086

... 4번째 Keepalive Connection 연결 ...
09:58:04.849554 IP nginx.40086 > springapp.docker_test-net.8080: Flags [S], seq 680315437, win 64240, options [mss 1460,sackOK,TS val 3471478299 ecr 0,nop,wscale 7], length 0
09:58:04.849703 IP springapp.docker_test-net.8080 > nginx.40086: Flags [S.], seq 4155862158, ack 680315438, win 65160, options [mss 1460,sackOK,TS val 2146464885 ecr 3471478299,nop,wscale 7], length 0
09:58:04.849714 IP nginx.40086 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 3471478299 ecr 2146464885], length 0

... 9번째 요청 ...
09:58:04.849794 IP nginx.40086 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 3471478299 ecr 2146464885], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:04.849803 IP springapp.docker_test-net.8080 > nginx.40086: Flags [.], ack 78, win 509, options [nop,nop,TS val 2146464885 ecr 3471478299], length 0
09:58:05.857451 IP springapp.docker_test-net.8080 > nginx.40086: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 2146465893 ecr 3471478299], length 120: HTTP: HTTP/1.1 200 
09:58:05.857493 IP nginx.40086 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 3471479307 ecr 2146465893], length 0

... 4번째 Keepalive Connection 해제 ...
09:58:06.878111 IP nginx.40086 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 121, win 502, options [nop,nop,TS val 3471480327 ecr 2146465893], length 0
09:58:06.881450 IP springapp.docker_test-net.8080 > nginx.40086: Flags [F.], seq 121, ack 79, win 509, options [nop,nop,TS val 2146466917 ecr 3471480327], length 0
09:58:06.881463 IP nginx.40086 > springapp.docker_test-net.8080: Flags [.], ack 122, win 502, options [nop,nop,TS val 3471480331 ecr 2146466917], length 0
```  

```bash
$ tcpdump -r dump | grep 40096

... 5번째 Keepalive Connection 연결 ...
09:58:05.853299 IP nginx.40096 > springapp.docker_test-net.8080: Flags [S], seq 325885423, win 64240, options [mss 1460,sackOK,TS val 3471479302 ecr 0,nop,wscale 7], length 0
09:58:05.853337 IP springapp.docker_test-net.8080 > nginx.40096: Flags [S.], seq 150803246, ack 325885424, win 65160, options [mss 1460,sackOK,TS val 2146465888 ecr 3471479302,nop,wscale 7], length 0
09:58:05.853345 IP nginx.40096 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 3471479302 ecr 2146465888], length 0

... 10번째 요청 ...
09:58:05.853387 IP nginx.40096 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 3471479303 ecr 2146465888], length 77: HTTP: GET /test/1000/1 HTTP/1.1
09:58:05.853396 IP springapp.docker_test-net.8080 > nginx.40096: Flags [.], ack 78, win 509, options [nop,nop,TS val 2146465889 ecr 3471479303], length 0
09:58:06.875279 IP springapp.docker_test-net.8080 > nginx.40096: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 2146466910 ecr 3471479303], length 120: HTTP: HTTP/1.1 200 
09:58:06.875387 IP nginx.40096 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 3471480324 ecr 2146466910], length 0

... 5번째 Keepalive Connection 해제 ...
09:58:15.892287 IP nginx.40096 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 121, win 502, options [nop,nop,TS val 3471489341 ecr 2146466910], length 0
09:58:15.894919 IP springapp.docker_test-net.8080 > nginx.40096: Flags [F.], seq 121, ack 79, win 509, options [nop,nop,TS val 2146475930 ecr 3471489341], length 0
09:58:15.894953 IP nginx.40096 > springapp.docker_test-net.8080: Flags [.], ack 122, win 502, options [nop,nop,TS val 3471489344 ecr 2146475930], length 0
```  

`Keepalive Connection` 은 동시에 총 2개의 커넥션을 사용 할 수 있다. 
그리고 각 커넥션 마다 최대 3개의 요청을 처리할 수 있고, 최대 10초 동안 유지하고, 9초 동안 요청이 없으면 커넥션을 닫게 된다. 
`Spring Boot` 의 관련 설정 값들이 `Nginx` 보다 크고 높게 잡혀 있으므로 기대한 것과 동일하게 모든 커넥션의 시작과 끝은 `Nginx` 로 부터 수행됐음을 확인 할 수 있다. 
이제 이후 예제에서 주요한 설정이 호환되지 않을 경우 발생하는 문제점에 대해 알아본다.  




### Invalid Keepalive Timeout

| 구분     | 설정 필드             | 설정값
|--------|-------------------|------
|  Nginx | keepalive         | 1    
| | keepalive_requests | 3    
| | keepalive_time    | 10s   
| | keepalive_timeout | 9s   
| | worker_processes  | 2    
| | worker_connections | 1024 
|Spring Boot Web|server.tomcat.max-connections| 8 
| |server.tomcat.threads.max| 4   
| |server.tomcat.threads.min-spare| 2
| |server.tomcat.keep-alive-timeout| 1s 
| |server.tomcat.max-keep-alive-requests| 20
| |server.tomcat.accept-count| 1  


다른 부분들은 모두 상단 `allok` 의 예제와 동일하다. 
차이점은 `server.tomcat.keep-alive-timeout` 값을 `20s -> 1s` 변경해 
`Nginx` 의 `keepalive_time` 의 `10s` 보다 더 작은 값으로 설정했다.  


```
# nginx-keepalive-invalidkeepalivetimeout.conf

user  nginx;
worker_processes  2;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection $connection_requests';

    keepalive_timeout  0;

    upstream app {
        server springapp:8080;
        keepalive 1;
        keepalive_time 10s;
        keepalive_timeout 9s;
        keepalive_requests 3;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /logs/error.log warn;

      location / {
          proxy_connect_timeout 1s;
          proxy_send_timeout 1s;
          proxy_read_timeout 2s;
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

```properties
# application-invalidkeepalivetimeout.yaml

server:
  http2:
    enabled: true
  tomcat:
    threads:
      max: 4
      min-spare: 1
    connection-timeout: 1s
    # 20s -> 1s 로 수정
    keep-alive-timeout: 1s
    max-keep-alive-requests: 20
    max-connections: 8
    accept-count: 1
```  

총 7개의 커넥션이 맺어지고 해제되었는데 
각 커넥션이 어떻게 사용됐는지 살펴보자. 


```bash
$ tcpdump -r dump | grep 48844

... 1번째 Keepalive Connection 연결 ...
10:06:26.434309 IP nginx.48844 > springapp.docker_test-net.8080: Flags [S], seq 3287234526, win 64240, options [mss 1460,sackOK,TS val 2146966469 ecr 0,nop,wscale 7], length 0
10:06:26.434418 IP springapp.docker_test-net.8080 > nginx.48844: Flags [S.], seq 3224910527, ack 3287234527, win 65160, options [mss 1460,sackOK,TS val 3471979884 ecr 2146966469,nop,wscale 7], length 0
10:06:26.434430 IP nginx.48844 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146966470 ecr 3471979884], length 0

... 1번째 요청 ...
10:06:26.434497 IP nginx.48844 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146966470 ecr 3471979884], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:26.434506 IP springapp.docker_test-net.8080 > nginx.48844: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471979884 ecr 2146966470], length 0
10:06:27.467721 IP springapp.docker_test-net.8080 > nginx.48844: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471980917 ecr 2146966470], length 120: HTTP: HTTP/1.1 200 
10:06:27.467764 IP nginx.48844 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146967503 ecr 3471980917], length 0

... 1번째 Keepalive Connection 해제 ...
10:06:28.473024 IP springapp.docker_test-net.8080 > nginx.48844: Flags [F.], seq 121, ack 78, win 509, options [nop,nop,TS val 3471981922 ecr 2146967503], length 0
10:06:28.473266 IP nginx.48844 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 122, win 502, options [nop,nop,TS val 2146968508 ecr 3471981922], length 0
10:06:28.473303 IP springapp.docker_test-net.8080 > nginx.48844: Flags [.], ack 79, win 509, options [nop,nop,TS val 3471981922 ecr 2146968508], length 0
```

```bash
$ tcpdump -r dump | grep 48852

... 2번째 Keepalive Connection 연결 ...
10:06:27.445404 IP nginx.48852 > springapp.docker_test-net.8080: Flags [S], seq 1130452829, win 64240, options [mss 1460,sackOK,TS val 2146967481 ecr 0,nop,wscale 7], length 0
10:06:27.445450 IP springapp.docker_test-net.8080 > nginx.48852: Flags [S.], seq 499543328, ack 1130452830, win 65160, options [mss 1460,sackOK,TS val 3471980895 ecr 2146967481,nop,wscale 7], length 0
10:06:27.445459 IP nginx.48852 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146967481 ecr 3471980895], length 0

... 2번째 요청 ...
10:06:27.445507 IP nginx.48852 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146967481 ecr 3471980895], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:27.445516 IP springapp.docker_test-net.8080 > nginx.48852: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471980895 ecr 2146967481], length 0
10:06:28.475116 IP springapp.docker_test-net.8080 > nginx.48852: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471981924 ecr 2146967481], length 120: HTTP: HTTP/1.1 200 
10:06:28.475152 IP nginx.48852 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146968510 ecr 3471981924], length 0

... 3번째 요청 ...
10:06:29.469904 IP nginx.48852 > springapp.docker_test-net.8080: Flags [P.], seq 78:155, ack 121, win 502, options [nop,nop,TS val 2146969505 ecr 3471981924], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:29.469949 IP springapp.docker_test-net.8080 > nginx.48852: Flags [.], ack 155, win 509, options [nop,nop,TS val 3471982919 ecr 2146969505], length 0
10:06:30.478039 IP springapp.docker_test-net.8080 > nginx.48852: Flags [P.], seq 121:241, ack 155, win 509, options [nop,nop,TS val 3471983927 ecr 2146969505], length 120: HTTP: HTTP/1.1 200 
10:06:30.478117 IP nginx.48852 > springapp.docker_test-net.8080: Flags [.], ack 241, win 502, options [nop,nop,TS val 2146970513 ecr 3471983927], length 0

... 4번째 요청 ...
10:06:30.490141 IP nginx.48852 > springapp.docker_test-net.8080: Flags [P.], seq 155:232, ack 241, win 502, options [nop,nop,TS val 2146970525 ecr 3471983927], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:30.490184 IP springapp.docker_test-net.8080 > nginx.48852: Flags [.], ack 232, win 509, options [nop,nop,TS val 3471983939 ecr 2146970525], length 0
10:06:31.499574 IP springapp.docker_test-net.8080 > nginx.48852: Flags [P.], seq 241:361, ack 232, win 509, options [nop,nop,TS val 3471984948 ecr 2146970525], length 120: HTTP: HTTP/1.1 200 

... 2번째 Keepalive Connection 해제 ...
10:06:31.501013 IP nginx.48852 > springapp.docker_test-net.8080: Flags [F.], seq 232, ack 361, win 502, options [nop,nop,TS val 2146971536 ecr 3471984948], length 0
10:06:31.503547 IP springapp.docker_test-net.8080 > nginx.48852: Flags [F.], seq 361, ack 233, win 509, options [nop,nop,TS val 3471984953 ecr 2146971536], length 0
10:06:31.503574 IP nginx.48852 > springapp.docker_test-net.8080: Flags [.], ack 362, win 502, options [nop,nop,TS val 2146971539 ecr 3471984953], length 0
```

```bash
$ tcpdump -r dump | grep 48854

... 3번째 Keepalive Connection 연결 ...
10:06:28.471007 IP nginx.48854 > springapp.docker_test-net.8080: Flags [S], seq 228758932, win 64240, options [mss 1460,sackOK,TS val 2146968506 ecr 0,nop,wscale 7], length 0
10:06:28.471098 IP springapp.docker_test-net.8080 > nginx.48854: Flags [S.], seq 1047666222, ack 228758933, win 65160, options [mss 1460,sackOK,TS val 3471981920 ecr 2146968506,nop,wscale 7], length 0
10:06:28.471117 IP nginx.48854 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146968506 ecr 3471981920], length 0

... 5번째 요청 ...
10:06:28.471340 IP nginx.48854 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146968506 ecr 3471981920], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:28.471362 IP springapp.docker_test-net.8080 > nginx.48854: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471981920 ecr 2146968506], length 0
10:06:29.479653 IP springapp.docker_test-net.8080 > nginx.48854: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471982929 ecr 2146968506], length 120: HTTP: HTTP/1.1 200 
10:06:29.479719 IP nginx.48854 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146969515 ecr 3471982929], length 0

... 3번째 Keepalive Connection 해제 ...
10:06:30.479458 IP nginx.48854 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 121, win 502, options [nop,nop,TS val 2146970515 ecr 3471982929], length 0
10:06:30.482761 IP springapp.docker_test-net.8080 > nginx.48854: Flags [F.], seq 121, ack 79, win 509, options [nop,nop,TS val 3471983932 ecr 2146970515], length 0
10:06:30.482779 IP nginx.48854 > springapp.docker_test-net.8080: Flags [.], ack 122, win 502, options [nop,nop,TS val 2146970518 ecr 3471983932], length 0
```

```bash
$ tcpdump -r dump | grep 50752

... 4번째 Keepalive Connection 연결 ...
10:06:31.515657 IP nginx.50752 > springapp.docker_test-net.8080: Flags [S], seq 676558160, win 64240, options [mss 1460,sackOK,TS val 2146971551 ecr 0,nop,wscale 7], length 0
10:06:31.515703 IP springapp.docker_test-net.8080 > nginx.50752: Flags [S.], seq 476368873, ack 676558161, win 65160, options [mss 1460,sackOK,TS val 3471984965 ecr 2146971551,nop,wscale 7], length 0
10:06:31.515708 IP nginx.50752 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146971551 ecr 3471984965], length 0

... 6번째 요청 ...
10:06:31.515754 IP nginx.50752 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146971551 ecr 3471984965], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:31.515759 IP springapp.docker_test-net.8080 > nginx.50752: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471984965 ecr 2146971551], length 0
10:06:32.522330 IP springapp.docker_test-net.8080 > nginx.50752: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471985971 ecr 2146971551], length 120: HTTP: HTTP/1.1 200 
10:06:32.522416 IP nginx.50752 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146972558 ecr 3471985971], length 0

... 7번째 요청 ...
10:06:32.532043 IP nginx.50752 > springapp.docker_test-net.8080: Flags [P.], seq 78:155, ack 121, win 502, options [nop,nop,TS val 2146972567 ecr 3471985971], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:32.532091 IP springapp.docker_test-net.8080 > nginx.50752: Flags [.], ack 155, win 509, options [nop,nop,TS val 3471985981 ecr 2146972567], length 0
10:06:33.544727 IP springapp.docker_test-net.8080 > nginx.50752: Flags [P.], seq 121:241, ack 155, win 509, options [nop,nop,TS val 3471986994 ecr 2146972567], length 120: HTTP: HTTP/1.1 200 
10:06:33.593857 IP nginx.50752 > springapp.docker_test-net.8080: Flags [.], ack 241, win 502, options [nop,nop,TS val 2146973629 ecr 3471986994], length 0

... 8번째 요청 응답 전 커넥션이 끊겨 실패 ...
10:06:34.552353 IP nginx.50752 > springapp.docker_test-net.8080: Flags [P.], seq 155:232, ack 241, win 502, options [nop,nop,TS val 2146974587 ecr 3471986994], length 77: HTTP: GET /test/1000/1 HTTP/1.1

... 4번째 Keepalive Connection 해제 ...
10:06:34.552367 IP springapp.docker_test-net.8080 > nginx.50752: Flags [F.], seq 241, ack 155, win 509, options [nop,nop,TS val 3471988001 ecr 2146973629], length 0
10:06:34.552394 IP springapp.docker_test-net.8080 > nginx.50752: Flags [.], ack 232, win 509, options [nop,nop,TS val 3471988002 ecr 2146974587], length 0
10:06:34.552554 IP nginx.50752 > springapp.docker_test-net.8080: Flags [F.], seq 232, ack 242, win 502, options [nop,nop,TS val 2146974588 ecr 3471988001], length 0
10:06:34.552580 IP springapp.docker_test-net.8080 > nginx.50752: Flags [.], ack 233, win 509, options [nop,nop,TS val 3471988002 ecr 2146974588], length 0
```

```bash
$ tcpdump -r dump | grep 50758

... 5번째 Keepalive Connection 연결 ...
10:06:33.542625 IP nginx.50758 > springapp.docker_test-net.8080: Flags [S], seq 573566322, win 64240, options [mss 1460,sackOK,TS val 2146973578 ecr 0,nop,wscale 7], length 0
10:06:33.542734 IP springapp.docker_test-net.8080 > nginx.50758: Flags [S.], seq 2169468269, ack 573566323, win 65160, options [mss 1460,sackOK,TS val 3471986992 ecr 2146973578,nop,wscale 7], length 0
10:06:33.542753 IP nginx.50758 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146973578 ecr 3471986992], length 0

... 9번째 요청 ...
10:06:33.542838 IP nginx.50758 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146973578 ecr 3471986992], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:33.542849 IP springapp.docker_test-net.8080 > nginx.50758: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471986992 ecr 2146973578], length 0
10:06:34.554867 IP springapp.docker_test-net.8080 > nginx.50758: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471988004 ecr 2146973578], length 120: HTTP: HTTP/1.1 200 
10:06:34.554945 IP nginx.50758 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146974590 ecr 3471988004], length 0

... 5번째 Keepalive Connection 해제 ...
10:06:35.566180 IP springapp.docker_test-net.8080 > nginx.50758: Flags [F.], seq 121, ack 78, win 509, options [nop,nop,TS val 3471989015 ecr 2146974590], length 0
10:06:35.584484 IP nginx.50758 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 122, win 502, options [nop,nop,TS val 2146975620 ecr 3471989015], length 0
10:06:35.584537 IP springapp.docker_test-net.8080 > nginx.50758: Flags [.], ack 79, win 509, options [nop,nop,TS val 3471989034 ecr 2146975620], length 0
```

```bash
$ tcpdump -r dump | grep 50766

... 6번째 Keepalive Connection 연결 ...
10:06:34.552668 IP nginx.50766 > springapp.docker_test-net.8080: Flags [S], seq 226606064, win 64240, options [mss 1460,sackOK,TS val 2146974588 ecr 0,nop,wscale 7], length 0
10:06:34.552710 IP springapp.docker_test-net.8080 > nginx.50766: Flags [S.], seq 139710190, ack 226606065, win 65160, options [mss 1460,sackOK,TS val 3471988002 ecr 2146974588,nop,wscale 7], length 0
10:06:34.552721 IP nginx.50766 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146974588 ecr 3471988002], length 0

... 8번째 요청 재시도 성공 ...
10:06:34.552782 IP nginx.50766 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146974588 ecr 3471988002], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:34.552796 IP springapp.docker_test-net.8080 > nginx.50766: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471988002 ecr 2146974588], length 0
10:06:35.571215 IP springapp.docker_test-net.8080 > nginx.50766: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471989020 ecr 2146974588], length 120: HTTP: HTTP/1.1 200 
10:06:35.571256 IP nginx.50766 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146975606 ecr 3471989020], length 0

... 6번째 Keepalive Connection 해제 ...
10:06:36.592991 IP springapp.docker_test-net.8080 > nginx.50766: Flags [F.], seq 121, ack 78, win 509, options [nop,nop,TS val 3471990042 ecr 2146975606], length 0
10:06:36.593947 IP nginx.50766 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 122, win 502, options [nop,nop,TS val 2146976629 ecr 3471990042], length 0
10:06:36.594026 IP springapp.docker_test-net.8080 > nginx.50766: Flags [.], ack 79, win 509, options [nop,nop,TS val 3471990043 ecr 2146976629], length 0
```

```bash
$ tcpdump -r dump | grep 50780

... 7번째 Keepalive Connection 연결 ...
10:06:35.585913 IP nginx.50780 > springapp.docker_test-net.8080: Flags [S], seq 2897084479, win 64240, options [mss 1460,sackOK,TS val 2146975621 ecr 0,nop,wscale 7], length 0
10:06:35.585964 IP springapp.docker_test-net.8080 > nginx.50780: Flags [S.], seq 1666003227, ack 2897084480, win 65160, options [mss 1460,sackOK,TS val 3471989035 ecr 2146975621,nop,wscale 7], length 0
10:06:35.585974 IP nginx.50780 > springapp.docker_test-net.8080: Flags [.], ack 1, win 502, options [nop,nop,TS val 2146975621 ecr 3471989035], length 0

... 10번째 요청 ...
10:06:35.586026 IP nginx.50780 > springapp.docker_test-net.8080: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 2146975621 ecr 3471989035], length 77: HTTP: GET /test/1000/1 HTTP/1.1
10:06:35.586032 IP springapp.docker_test-net.8080 > nginx.50780: Flags [.], ack 78, win 509, options [nop,nop,TS val 3471989035 ecr 2146975621], length 0
10:06:36.601586 IP springapp.docker_test-net.8080 > nginx.50780: Flags [P.], seq 1:121, ack 78, win 509, options [nop,nop,TS val 3471990051 ecr 2146975621], length 120: HTTP: HTTP/1.1 200 
10:06:36.601676 IP nginx.50780 > springapp.docker_test-net.8080: Flags [.], ack 121, win 502, options [nop,nop,TS val 2146976637 ecr 3471990051], length 0

... 7번째 Keepalive Connection 해제 ...
10:06:37.607576 IP springapp.docker_test-net.8080 > nginx.50780: Flags [F.], seq 121, ack 78, win 509, options [nop,nop,TS val 3471991057 ecr 2146976637], length 0
10:06:37.609732 IP nginx.50780 > springapp.docker_test-net.8080: Flags [F.], seq 78, ack 122, win 502, options [nop,nop,TS val 2146977645 ecr 3471991057], length 0
10:06:37.609774 IP springapp.docker_test-net.8080 > nginx.50780: Flags [.], ack 79, win 509, options [nop,nop,TS val 3471991059 ecr 2146977645], length 0
```  


```bash
$ cat logs/error.log 
2024/04/28 10:06:34 [error] 29#29: *16 upstream prematurely closed connection while reading response header from upstream, client: 172.22.0.1, server: , request: "GET /test/1000/1 HTTP/1.1", upstream: "http://172.22.0.2:8080/test/1000/1", host: "localhost"
```  

1, 4, 5, 6, 7번째 요청에서 `Spring Boot` 에서 먼저 `Nginx` 와의 커넥션을 끊는 증상이 있었다. 
하지만 비정상적인 커넥션 해제이지만 처리 중인 요청은 있지 않아 별다른 에러는 발생하지 않았다. 
하지만 이러한 연결 해제가 있다는 것은 추후 에러를 유발시킬 수 있다는 것을 기억해야 한다.  

`Spring Boot` 에서 먼저 커넥션을 해제한 것 중 4번째 커넥션이 처리할 요청 중 8번째 요청을 보면 
`Nginx` 가 `Spring Boot` 로 요청을 하고 연결이 끊긴 것을 볼 수 있다. 
주의가 필요한 경우가 바로 이런 경우이다. 
`Nginx` 는 아직 연결이 끊길 시간이 아니므로 `Spring Boot` 로 요청을 수행하지만, 
`Spring Boot` 입장에서는 지금 연결을 끊어야 하는 시점이므로 `Nginx` 의 요청과 상관없이 연결을 끊어버리는 것이다. 
그래서 `Nginx` 의 `error.log` 에서도 관련 에러가 남겨진 것을 확인 할 수 있다. 
하지만 `Nginx` 에서는 이러한 경우 자동으로 요청을 재시도하는 매커니즘이 있으므로 실제 사용자는 정상응답을 받게된다. 
실제로 6번째 연결에서 8번째 요청을 재시도하는 것을 확인 할 수 있다.  

