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
`Keepalive` 설정|커널 설정에 따름|웹서버, `WAS` 설정 옵션에 따름
`Keepalive` 유지/관리|커널이 담당|웹서버, `WAS` 가 관리 
세션 유지 방식|주기적인 연결확인|연결확인 없이 타임아웃 시간까지만 유지 



### TCP Keepalive 

`TCP Keepalive` 의 설정은 운영체제 커널에서 관리한다. 
`Linux` 를 기준으로 설정할 수 있는 옵션은 아래와 같다. 

```bash
$ sysctl -a | grep keepalive
net.ipv4.tcp_keepalive_intvl = 75 # 연결확인 패킷 재전송 주기
net.ipv4.tcp_keepalive_probes = 9 # 연결확인이 될떄까지 최대 몇번 패킷을 보낼지
net.ipv4.tcp_keepalive_time = 7200 # keepalive 소켓 유지시간
```  






---
## Reference
[Keepalive](https://en.wikipedia.org/wiki/Keepalive)  
	