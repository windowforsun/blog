--- 
layout: single
classes: wide
title: "[Linux 실습] CLI 로 성능 분석하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
---  

## 환경
- CentOS 7

## 개요
- 서버 성능이나 현재 상태를 파악이 필요할 때, 클라우드 서비스를 사용한다면 자체적으로 제공하는 모니터링 툴을 이용할 수 있다.
- 별도의 툴을 사용해서 성능 체크나 모니터링도 가능하고 대부분의 이슈 파악까지 해결할 수 있다.
- 특정 상황에서 직접 서버 인스턴스에 로그인해서 `CLI` 를 통해 리눅스 표준 성능 체크 툴이 필요할 경우에 대해 설명한다.

## uptime

```bash
$ uptime
 22:14:25 up 12 days, 11:51,  3 users,  load average: 8.48, 7.65, 6.37
```  

- `uptime` 은 현재 대기중인 프로세스가 얼마나 있는지를 의미하는 `load average` 를 확인하기 좋은 명령어이다.
- `load average` 는 대기 중인 프로세스, Disk I/O 와 같은 I/O 작업으로 부터 블럭된 프로세스까지 포함한다.
- 대략적으로 얼만큼의 리소스가 사용되고 있는지 확인 가능하다. 정확한 파악을 위해서는 이후에 나오는 `vmstat`, `mpstat` 등의 명령어가 필요하다.
- `8.48, 7.65, 6.37` 의 값은 순서대로 1분, 5분, 15분에 대한 `load average` 의 값이다.
- 장애 상황일 때 1분의 값이 5분이나 15분의 값보다 작다면, 확인이 너무 늦었다는 의미가 될 수 있고, 1분의 값이 다른 값들보다 현저하게 높다면 최근에 장애가 발생했다고 파악할 수 있다.

## dmesg | tail

```bash
$ dmesg | tail
[1078468.902282] device vethc09632a entered promiscuous mode
[1078468.903237] IPv6: ADDRCONF(NETDEV_UP): vethb1bb04d: link is not ready
[1078468.904539] IPv6: ADDRCONF(NETDEV_UP): vethc09632a: link is not ready
[1078468.904588] IPv6: ADDRCONF(NETDEV_CHANGE): vethc09632a: link becomes ready
[1078468.904622] docker0: port 1(vethc09632a) entered blocking state
[1078468.904626] docker0: port 1(vethc09632a) entered forwarding state
[1078468.904651] IPv6: ADDRCONF(NETDEV_CHANGE): vethb1bb04d: link becomes ready
[1078468.929065] docker0: port 1(vethc09632a) entered disabled state
[1078468.932709] docker0: port 1(vethc09632a) entered blocking state
[1078468.932712] docker0: port 1(vethc09632a) entered forwarding state
```  

- `dmesg` 는 시스템 메시지를 확인 할 수 있는 명령어이다.
- 인스턴스가 부팅한 시점으로 부터 커널 메시지를 확인 할 수 있다.
- 


	
---
## Reference
[Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)  