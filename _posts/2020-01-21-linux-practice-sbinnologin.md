--- 
layout: single
classes: wide
title: "[Linux 실습] 쉘 권한 없는 사용자 계정"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '/sbin/nologin, /bin/false 와 같은 쉘 권한 없는 사용자 계정에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
  - CentOS 7
---  

## 환경
- CentOS 7


## 쉘 권이 없는 사용자
- SSH 접속은 되지않지만, FTP 접속은 가능한 계정을 만들 수 있다.
- Linux 에서 Bash 쉘 권한이 없는 계정을 만들기위해서는 `useradd` 의 `-s` 옵션에 `/sbin/nologin` 또는 /bin/false` 를 주면 된다.
- 시스템 계정, `apache`, `gocd` 등의 애플리케이션에서 사용하는 계정들이 보통 `nologin` 이다.
- `nologin` 계정을 확인하는 방법은 `grep nologin /etc/passwd` 명령어로 가능하다.

	```
	[root@localhost]# grep nologin /etc/passwd
	bin:x:1:1:bin:/bin:/sbin/nologin
	daemon:x:2:2:daemon:/sbin:/sbin/nologin
	
	# 생략 ..
	
	systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin

	```  
	
## 쉘 권한 없는 계정 만들기
### /sbin/nologin
- `useradd -s /sbin/nologin <사용자명>` 으로 계정을 추가할 수 있다.

	```
	[root@localhost]# useradd -s /sbin/nologin testnologin1
	[root@localhost]# grep testnologin1 /etc/passwd
	testnologin1:x:1001:1001::/home/testnologin1:/sbin/nologin
	```  
	
- 아래와 같이 `/sbin/nologin` 으로 설정된 계정으로 변경하려고 하면 메시지가 뜬다.

	```
	[root@localhost]# su - testnologin1
	This account is currently not available.
	```  
	
### /bin/false
- `useradd -s /bin/false <사용자명>` 으로도 계정 추가가 가능하다.

	```
	[root@localhost]# useradd -s /bin/false testfalse1
	[root@localhost]# grep testfalse1 /etc/passwd
	testfalse1:x:1002:1002::/home/testfalse1:/bin/false
	```  
	
- `/bin/false` 계정으로 사용자 전환을 하려고 하면 메시지는 뜨지 않지만, 동일하게 전환이 안된다.

	```
	[root@localhost]# su - testfalse1
	[root@localhost]# 
	```  

## 쉘 권한이 없는 계정의 쉘에 로그인하기
- `testnologin1`, `testfalse1` 이 두 계정은 위에서 생성한 쉘 권한이 없는 계정이다.
- 해당 계정에 관련된 설정을 하기 위해서 쉘로 접근해야 하는 경우가 있는데, 관리자 권한을 얻어 쉘로 접근할 수 있다.
- `sudo -u <쉘권한이없는계정명> /bin/bash` 을 통해 쉘 접근이 가능하다.

	```
	[root@localhost]# whoami
	root
	[root@localhost]# sudo -u testnologin1 /bin/bash
	[testnologin1@localhost]$ whoami
	testnologin1
	[testnologin1@localhost]$ exit
	exit
	[root@localhost]# sudo -u testfalse1 /bin/bash
	[testfalse1@localhost]$ whoami
	testfalse1
	[testfalse1@localhost]$ exit
	exit
	[root@localhost]#
	```  
	
	
---
## Reference
[Does /usr/sbin/nologin as a login shell serve a security purpose?](https://unix.stackexchange.com/questions/155139/does-usr-sbin-nologin-as-a-login-shell-serve-a-security-purpose)  