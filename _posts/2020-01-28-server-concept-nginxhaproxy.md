--- 
layout: single
classes: wide
title: "Nginx 이용한 Load Balancing "
header:
  overlay_image: /img/server-bg.jpg
excerpt: 'Nginx 를 사용해서 Load balancer 를 구성하는 설정에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Load Balancing
  - Nginx
---  

## Nginx
- Nginx 는 Apache 의 문제점(1000K) 를 해결하기 위해 만들어진 비동기 웹서버로 가볍고 빠른 오픈소스 애플리케이션이다.
- `http`, `reverse proxy` 외에도 `load balancer` 기능도 지원하고 강력하다.
- Nginx 에서 `load balancer` 는 `upstream` 블럭을 사용해서 설정한다.

```
upstream myapp {
	least_conn
	server first-server:8080 max_fails=2 fail_timeout=5s;
	server second-server:8080 max_fails=2 fail_timeout=5s;
	server tmp-server:8080 backup;
}
```  

- `myapp` 은 하나의 서버 클러스터에 대한 이름으로 2개의 서버에 대해 `load balancer` 를 수행한다.
- `least_conn` 은 요청을 분산 시킬때 사용하는 `load balancing method`(로드 밸런싱 규칙) 중 하나이다.
	- `round-robin` : 로드 밸런싱 규칙을 설정하지 않았을 때 기본값으로, 요청을 순서대로 처리한다.
		
		```
		upstream myapp {
			server first-server:8080 weight=5;
			server second-server:8080 weight=5;
		}
		```  
		
		- 위와 같이 `weight` 옵션을 줄 경우 first 에 5번 요청을 한 후 second 서버에 요청한다.
	- `least_conn` : 서버에 할당된 가중치를 고려하여 클라이언트 연결 수가 적은 서버로 요청한다.
	- `ip_hash` : 클라이언트 IP 를 hash 해서 특정 클라이언트는 특정 서버로 연겨하는 설정이다. (session clustering 이 구축되지 않았을 때 유용하다)
- `server` 클러스터에 등록할 서버의 아이피와 포트 및 옵션을 명시하는 부분이다.
	- `weight=n` : 위에서 설명한 것과 같이 해당 서버의 가중치에 대한 옵션이다.
	- `max_fails=n` : n 횟수 만큼 실패가 일어나면 서버가 죽은 것으로 판단한다.
	- `fail_timeout=n` : `max_fails` 가 지정된 상태에서 n 동안 응답하지 않으면 죽은 것으로 판단한다.
	- `down` : 해당 서버를 사용하지 않게 지정한다. `ip_hash` 가 설정된 상태에만 유효하다.
	- `backup` : 클러스터로 등록된 모든 서버가 동작하지 않을 때만 `backup` 으로 등록된 서버가 사용된다.
- Nginx 는 Nginx Plus 에만 `Health check` 기능이 있어, 서버의 상태를 지속적으로 체크하기 위해서는 별도의 API 를 구성하는 방식해야 한다.
	- 특정 상황에 따라 서버를 스위칭하는 `High Availability` 에는 적합하지 않을 수 있다.

### HTTP load balancing
- HTTP 통신에 대한 load balancing 설정을 하면 아래와 같다.

```
http {
	upstream myhttp {
		server first-server:8080;
		server second-server:8080;
	}
	
	server {
		listen 80;
		
		location / {
			proxy_set_header Host $host;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://myhttp;
		}
	}
}
```  

### TCP/UDP load balancing
- TCP/UDP 통신에 대한 load balancing 설정은 아래와 같다.

```
stream {
	upstream mytcp {
		server first-server:9909;
		server second-server:9909;
	}

	server {
		listen 9909[tcp];
		proxy_pass mytcp;
	}
}
```  

---
## Reference

	