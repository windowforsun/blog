--- 
layout: single
classes: wide
title: "[Linux 실습] CentOS 7 방화벽 설정"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'CentOS 7 방화벽 설정해보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
  - CentOS 7
  - firewall
---  

## 환경
- CentOS 7

## 방화벽 포트 열기
- CentOS 7 방화벽에 포트를 관리할때 `firewall` 명령어를 사용한다.
- `firewall-cmd --zone=public --permanent --add-port=<포트번호>/<tcp,udp>` 명령어를 통해 포트를 열수 있다.

	```
	[root@windowforsun]# firewall-cmd --zone=public --permanent --add-port=6300/tcp
    success
	```  

	- TCP 의 6300 포트를 오픈하는 명령어 이다.
	
- `firewall-cmd --reload` 명령어를 통해 변경한 방화벽 설정을 적용시킬 수 있다.

	```
	[root@windowforsun]# firewall-cmd --reload
	success
	```  
	
## 방화벽 포트 삭제
- `firewall-cmd --permanent --zone=public --remove-port=<포트번호>/<tcp,udp>`
	
## 열린 포트 정보 확인하기
- `firewall-cmd --zone=public --list-all` 명령어를 통해 오픈된 모든 포트를 확인 할 수 있다.

	```
	[root@dontrise2 windowforsun_surface]# firewall-cmd --zone=public --list-all
	public
	  target: default
	  icmp-block-inversion: no
	  interfaces:
	  sources:
	  services: dhcpv6-client ssh
	  ports: 8080/tcp 2377/tcp 6300/tcp
	  protocols:
	  masquerade: no
	  forward-ports:
	  source-ports:
	  icmp-blocks:
	  rich rules:
	```  

---
## Reference
[]()  