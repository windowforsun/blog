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

### Signal
`PID 1` 프로세스 즉 `init` 시스템이 수행해 주는 역할 중에는 프로세스들의 `시그널` 관리를 해주는 역할 도 있다. 
여기서 `시그널` 이란 프로세스 간 통신의 한 형태로, 운영체제나 다른 프로세스가 프로세스에 특정 이벤트를 알리는 매커니즘을 의미한다. 
주요 시그널의 종류로는 아래와 같은 것들이 있다. 

- `SIGTERM`(15) : 정상 종료 요청
- `SIGKILL`(9) : 상제 종료(무시 불가)
- `SIGINT`(2) : 인터럽트 (Ctrl + C)
- `SIGHUP`(1) : 터미널 연결 종료
- `SIGCHLD`(17) : 자식 프로세스 상태 변경

즉 `PId 1` 프로세스가 `Signal` 처리 및 관리의 역할이 부족하다면 자식 프로세스들은 정상적으로 시그널을 받지 못하게 될 수 있다. 


### Orphaned/Zombie Process
`PID 1` 프로세스는 부모 프로세스의 강제 종료로 남겨진 자식 프로세스들의 새로운 부모 프로세스가 된다. 
그리고 주기적으로 `wait()` 시스템 콜을 호출하여 자식 프로세스들의 종료 상태를 수집해 `Zombie` 프로세스를 정리하고, 
프로세스 테이블에서 제거해 좀비 프로세스가 되는 것을 막아준다.  

`Orphaned Process` 는 부모 프로세스가 자식 프로세스보다 먼저 종료되어 부모가 없는 상태가 된 프로세스를 의미하고, 
`Zombie Process` 는 실행이 종료되었지만 프로세스 테이블에 여전히 남아있는 프로세스를 의미한다.  

즉 `PID 1` 프로세스가 `Orphaned/Zombie` 프로세스 관리에 대한 역할이 부족하면 하위 프로세스 관리가 되지 않아, 
불필요한 리소스가 사용될 수 있고, 과도하게 누적되면 시스템의 프로세스 테이블을 포화시켜 새로운 프로세스 생성을 방해할 수 있다.  


### Container PID 1 Problem
`Container` 환경(`Docker`, `Kubernetes`, ..) 에서는 `ENTRYPOINT(CMD)` 로 명시된 프로세스를 `PID 1` 으로 실행한다. 
그리고 `Container` 에 전달하는 모든 `Signal` 은 해당 `PID 1` 프로세스에만 전달돼 종료를 시킬 수 있다. 
이러한 이유로 컨체이너는 경량화 이미지를 사용해 단일 프로세스만 실행하는 경우가 많다.  

하지만 몇가지 상황에서는 `PID 1` 프로세스가 정상적인 역할 수행을 하지 못해 문제가 발생할 수 있다.  

전체 예제 코드는 [여기](https://github.com/windowforsun/docker-pid-1-problem-exam)
에서 확인할 수 있다.  

