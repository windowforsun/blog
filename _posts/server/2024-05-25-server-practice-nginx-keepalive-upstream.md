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

  