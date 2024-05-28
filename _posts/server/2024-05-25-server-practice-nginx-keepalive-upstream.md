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


