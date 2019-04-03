--- 
layout: single
classes: wide
title: "[Linux 실습] Redis 설치하기"
header:
  overlay_image: /img/linux-bg.png
excerpt: 'Linux 에 Redis 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Redis
---  

## 환경
- CentOS 6
- Redis 5

## 사전 작업
- Remi 저장소에서 Redis 최신 버전을 배포하며 Redis 는 EPEL 저장소에 있는 jemalloc 패키지를 필요로 하므로 EPEL, Remi 저장소를 설치한다.

```
[root@windowforsun ~]# rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm Retrieving http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
warning: /var/tmp/rpm-tmp.ytOBUl: Header V3 RSA/SHA256 Signature, key ID 0608b895: NOKEY
Preparing...                ########################################### [100%]
        package epel-release-6-8.noarch is already installed
```  

```
[root@windowforsun ~]# rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
Retrieving http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
warning: /var/tmp/rpm-tmp.jFOlQe: Header V4 DSA/SHA1 Signature, key ID 00f97f56: NOKEY
Preparing...                ########################################### [100%]
   1:remi-release           ########################################### [100%]

```  

## 설치
- EPEL, Remi 저장소를 활성화 시키기 위해 --enablerepo 옵션을 준다.
	- 기본적으로 활성화 시키기 위해서는 /etc/yum.repos.d/{remi,epel}.repo 파일내 enabled 항목을 1로 설정한다.
	
```
[root@windowforsun ~]# yum --enablerepo=epel,remi install redis

생략 ..

Dependencies Resolved

=====================================================================================================================
 Package                 Arch                     Version                               Repository              Size
=====================================================================================================================
Installing:
 redis                   x86_64                   5.0.4-1.el6.remi                      remi                   1.1 M

Transaction Summary
=====================================================================================================================
Install       1 Package(s)

Total download size: 1.1 M
Installed size: 2.7 M
Is this ok [y/N]: y
Downloading Packages:
redis-5.0.4-1.el6.remi.x86_64.rpm                                                             | 1.1 MB     00:00
warning: rpmts_HdrFromFdno: Header V4 DSA/SHA1 Signature, key ID 00f97f56: NOKEY
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-remi
Importing GPG key 0x00F97F56:
 Userid : Remi Collet <RPMS@FamilleCollet.com>
 Package: remi-release-6.10-1.el6.remi.noarch (installed)
 From   : /etc/pki/rpm-gpg/RPM-GPG-KEY-remi
Is this ok [y/N]: y
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
Warning: RPMDB altered outside of yum.
  Installing : redis-5.0.4-1.el6.remi.x86_64                                                                     1/1
  Verifying  : redis-5.0.4-1.el6.remi.x86_64                                                                     1/1

Installed:
  redis.x86_64 0:5.0.4-1.el6.remi

Complete!
```  

## Redis 구동

```
[root@windowforsun ~]# service redis start
Starting redis-server:                                     [  OK  ]
```  

## 설치 확인

```
[root@windowforsun ~]# redis-server -v
Redis server v=5.0.4 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=65c0b40c98c41bc1

```  

```
[root@windowforsun ~]# redis-cli
127.0.0.1:6379> set install_test testValue
OK
127.0.0.1:6379> get install_test
"testValue"

```  

## 부팅시 자동 실행

```
[root@windowforsun ~]# chkconfig redis on
```  

---
## Reference
[리눅스 톰캣7 컴파일 설치](https://zetawiki.com/wiki/%EB%A6%AC%EB%88%85%EC%8A%A4_%ED%86%B0%EC%BA%A37_%EC%BB%B4%ED%8C%8C%EC%9D%BC_%EC%84%A4%EC%B9%98)  
