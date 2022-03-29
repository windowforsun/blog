--- 
layout: single
classes: wide
title: "Keepalive"
header:
  overlay_image: /img/server-bg.jpg
excerpt: '연결을 확인하거나 연결을 유지하는 Keepalive 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - HTTP
  - TCP
  - Keepalive
toc: true
use_math: true
---  

## Keepalive
`Keepalive` 는 의미 그대로 커넥션에 대해서 연결유지에 대한 설정이라고 할 수 있다. 
커넥션이 정상적으로 연결중인지 확인 하는 용도로도 사용 가능하고, 
특정 시간 혹은 조건동안 커넥션을 끊지 않고 유지하는 용도로도 사용할 수 있다. 

`Keepalive` 는 `TCP` 관점에서 `Keepalive` 가 있고 `HTTP` 관점에서 `Keepalive` 가 있다. 
둘다 모두 `TCP` 세션(커넥션)을 유지하는 목적을 위해 사용할 수 있다는 공통점이 있지만, 
아래와 같은 차이점이 있다.  


차이점|TCP Keepalive|HTTP Keepalive
---|---|---
`Keepalive` 설정|커널 설정에 따름|웹서버 설정 등 별도 옵션에 따름
`Keepalive` 유지/관리|커널이 담당|웹서버 등 별도 관리 
세션 유지 방식|주기적인 연결확인|연결확인 없이 타임아웃 시간까지만 유지


### TCP Keepalive 
`TCP Keepalive` 는 소켓의 연결을 더 호율적으로 관리하는 방식으로 사용될 수 있는 설정이다. 
선택적인 기능이고 기본 설정은 사용하지 않는 것이기 때문에, 별도로 활성화가 필요하다. 
주기적으로 소켓의 연결이 정상적인지 확인하기 때문에, 
비정상적으로 끊겨 계속 연결된 상태로 남아있게 된(좀비 커넥션, 데드 커넥션) 커넥션에 주기적으로 패킷을 보내 설정된 횟수 동안 응답이 없으면 커넥션을 끊는 방식으로 정리 할 수 있다.  

`TCP Keepalive` 의 설정은 운영체제 커널에서 관리한다. 
`Linux` 를 기준으로 설정할 수 있는 옵션은 아래와 같다. 

```bash
$ sysctl -a | grep keepalive
net.ipv4.tcp_keepalive_intvl = 75 # 연결확인 패킷 주기
net.ipv4.tcp_keepalive_probes = 9 # 연결확인이 될떄까지 최대 몇번 패킷을 보낼지
net.ipv4.tcp_keepalive_time = 7200 # keepalive 소켓 유지시간
```  

### HTTP Keepalive
`HTTP` 통신은 기본적으로 `Conneciton less` 이기 때문에 요청-응답 한번을 위해 연결을 맺고 끊는 것을 반복한다. 
하지만 연결에 대한 비용은 `3-way handshake` 뿐만 아니라 많은 리소스가 들어가는 작업이기 때문에, 
이러한 비효율을 `Keepalive` 설정을 통해 보완할 수 있다. 
즉 `HTTP Keepalive` 를 사용해서 한번 연결된 커넥션을 재사용해서 리소스적인 이점을 얻을 수 있다.  

`HTTP 1.0` 에서는 별도로 헤더에 `keep-alive` 옵션을 추가해야 동작하지만, 
`HTTP 1.1` 부터는 서버가 `Keepalive` 를 제공한다면 별도 설정없이 바로 사용할 수 있다. 

`HTTP Keepalive` 는 웹서버가 자체적으로 제공하는 `Keepalive` 라고도 할 수 있다. 
이러한 이유로 `TCP Keeaplive` 와 확연한 차이가 있고 설정값 또한 웹서버의 설정을 따르게 된다.
`TCP Keepalive` 와 비교했을 때 연결을 유지한다는 목적은 같지만, 
`HTTP Keepalive` 는 연결이후 부터 해제까지 주기적인 패킷을 통한 연결확인 없이 
설정된 타임아웃 시간까지만 연결을 유지하고 연결을 끊는(`FIN`) 방식이다.  

대표적인 웹서버 2가지 `HTTP Keepalive` 차이는 아래와 같다. 

웹서버|특징
---|---
Apache|`Event MPM` 방식이 아니라면 소켓 커넥션마다 하나의 스레드(프로세스)가 할당된다. 커넥션 증가에 따른 부하가 크다.
Nginx|`event-driven` 방식이므로 소켓 커넥션마다 스레드 할당이 없고 커넥션 증가에 따른 부하가 적다.

`HTTP Keepalive` 는 연결을 유지하는 목적이므로 서버의 리소스와 요청량 레이턴시 등을 잘 고려해서 적절한 설정이 필요하다. 
그렇지 않으면 과도한 커넥션이 유지될 수 있고, 이미 끊긴 연결을 재사용하는 증상이 발생할 수 있기 때문이다.  



---
## Reference
[Keepalive](https://en.wikipedia.org/wiki/Keepalive)  
[What is Keep-Alive?](https://blog.stackpath.com/glossary-keep-alive/)  
[Apache 2,4 와 Nginx 특징 및 비교](https://victorydntmd.tistory.com/231)  
	