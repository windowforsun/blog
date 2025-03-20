--- 
layout: single
classes: wide
title: "[Docker] Container PID 1 Problem"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Container 환경에서 PID 1 문제에 대해 알아보고, dumb-init 을 사용한 해결 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Kubernetes
  - Container
  - PID 1
  - Signal
  - Orphaned Process
  - Zombie Process
  - Dumb-init
  - Init System
  - tini
toc: true
use_math: true
---  

## PID 1
`PID 1` 은 `Linux` 시스템에서 가장 먼저 시작되는 사용자 모드 프로세스를 의미한다. 
일반적으로 `init` 프로세스라고(시스템) 불리며, 시스템이 종료될 떄까지 실행된다. 
그리고 `PID 1` 은 다른 서비스(데몬)들을 관리하고 프로세스 트리의 루트역할을 한다.  

대표적인 `init` 시스템으로는 `systemd` 와 `SysV` 등이 있는데, 
이러한 `init` 시스템은 일반적으로 `PID 1` 을 할당 받아 시스템 부팅 관리, 서비스 관리, 프로세스 관리, 시스템 상태 관리, 시스템 종료 관리 등을 수행한다. 

```bash
$ docker run -d --rm --name test --privileged centos:8 /sbin/init

$ $ docker exec -it test ps auxf  
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.0  22808  9916 ?        Ss   09:26   0:00 /sbin/init
root         126  0.0  0.0   8608  3028 pts/0    Rs+  09:27   0:00 ps auxf
root          25  0.0  0.0  28048 10164 ?        Ss   09:26   0:00 /usr/lib/systemd/systemd-journald
root          33  0.0  0.0  22396  9188 ?        Ss   09:26   0:00 /usr/lib/systemd/systemd-udevd
dbus          89  0.0  0.0  10020  3564 ?        Ss   09:26   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --system

```  

이번 포스팅에서는 이런 `PID 1` 의 역할과 그리고 `Container` 환경에서 `PID 1` 수행 방식에 따른 발생할 수 있는 문제점 들을 알아보고 
간단한 예제에 대해서도 살펴본다.  
