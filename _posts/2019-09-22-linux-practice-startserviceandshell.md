--- 
layout: single
classes: wide
title: "[Linux 실습] CentOS 7 부팅시 자동실행 서비스 또는 쉘, 명령어 관리"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'CentOS 7 부팅시 자동실행 서비스나 쉘, 명령어를 관라하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
  - CentOS 7
  - systemctl
---  

## 환경
- CentOS 7

## 자동실행 서비스 관리
- `systemctl enable <서비스이름>` 을 통해 자동실행 서비스 등록 가능하다.

	```
	[root@windowforsun]# systemctl enable docker
	Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
	```  
	
- `systemctl disable <서비스이름>` 으로 자동실행으로 등록되어 있던 서비스를 해제 시킬 수 있다.

	```
	[root@windowforsun]# systemctl disable docker
	Removed symlink /etc/systemd/system/multi-user.target.wants/docker.service.
	```  
	

## 자동실행 쉘 스크립트
- `/etc/rc.d/rc.local` 파일에 쉘 스크립트를 등록하면 부팅시 자동으로 실행 시켜 준다.

	```
	[root@windowforsun]# vi /etc/rc.d/rc.local
	
	# 파일 내용
	
	#!/bin/bash
    # THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
    #
    # It is highly advisable to create own systemd services or udev rules
    # to run scripts during boot instead of using this file.
    #
    # In contrast to previous versions due to parallel execution during boot
    # this script will NOT be run after all other services.
    #
    # Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
    # that this script will be executed during boot.
    
    touch /var/lock/subsys/local
    
    # 부팅시 실행시킬 쉘 추가
    /myshell/test.sh
	```  
	
- `chmod u+x /etc/rc.d/rc.local` 명령어로 실행 권한을 추가해준다.


## Reference
[]()  