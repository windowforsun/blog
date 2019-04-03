--- 
layout: single
classes: wide
title: "[Linux 실습] Jenkins 설치하기"
header:
  overlay_image: /img/linux-bg.png
excerpt: 'Linux 에 Jenkins 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Jenkins
---  

## 환경
- CentOS 6
- Jenkins 2.164

## 설치(LTS)

```
[root@windowforsun ~]# wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo
--2019-04-03 12:50:16--  http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo
Resolving pkg.jenkins-ci.org... 52.202.51.185
Connecting to pkg.jenkins-ci.org|52.202.51.185|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 85
Saving to: “/etc/yum.repos.d/jenkins.repo”

100%[===========================================================================>] 85          --.-K/s   in 0s

2019-04-03 12:50:16 (9.89 MB/s) - “/etc/yum.repos.d/jenkins.repo” saved [85/85]
```  

```
[root@windowforsun ~]# rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
```  

```
[root@windowforsun ~]# yum install jenkins

생략 ..

Dependencies Resolved

=====================================================================================================================
 Package                   Arch                     Version                          Repository                 Size
=====================================================================================================================
Installing:
 jenkins                   noarch                   2.164.1-1.1                      jenkins                    74 M

Transaction Summary
=====================================================================================================================
Install       1 Package(s)

Total download size: 74 M
Installed size: 74 M
Is this ok [y/N]: y
Downloading Packages:
jenkins-2.164.1-1.1.noarch.rpm                                                                |  74 MB     00:03
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : jenkins-2.164.1-1.1.noarch                                                                        1/1
  Verifying  : jenkins-2.164.1-1.1.noarch                                                                        1/1

Installed:
  jenkins.noarch 0:2.164.1-1.1

Complete!
```  

## 포트 변경
- 기본 포트 8080에서 9100으로 변경한다.

```
[root@windowforsun ~]# vi /etc/sysconfig/jenkins
```  

- 위 경로의 파일에서 **JENKINS_PORT="8080"** 부분을 **JENKINS_PORT="9100"** 으로 변경 해준다.

## 방화벽 설정

```
[root@windowforsun ~]# iptables -I INPUT 1 -p tcp --dport 9100 -j ACCEPT
[root@windowforsun ~]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
```  

- GCP, AWS 와 같은 클라우드 서버의 경우 해당 VM 인스턴스의 방화벽에 동일한 포트를 추가해 준다.

## Jenkins 서비스 시작

```
[root@windowforsun ~]# service jenkins start
Starting Jenkins                                           [  OK  ]
```  

## 접속 및 초기 설정

![jenkins 초기 접속 화면]({{site.baseurl}}/img/linux/linux-jenkins-firststart.png)

```
[root@windowforsun ~]# vi /var/lib/jenkins/secrets/initialAdminPassword
```  

- 위 파일에서 초기 비밀번호를 찾아 사용한다.

![jenkins 초기 로그인]({{site.baseurl}}/img/linux/linux-jenkins-firstlogin.png)

- **Install suggested plugins** 로 기본 플러그인을 설치한다.

![jenkins 초기 플러그인 설치]({{site.baseurl}}/img/linux/linux-jenkins-firstinstallplugins.png)

![jenkins 초기 사용자 생성]({{site.baseurl}}/img/linux/linux-jenkins-firstcreateaccount.png)

- 사용할 계정을 생성 한다.

![jenkins index 페이지]({{site.baseurl}}/img/linux/linux-jenkins-firstindex.png)

- 모든 설정을 완료하면 위와 같은 페이지가 보인다.
- 추가적인 플러그인은 **Jenkins 관리 -> 플러그인 관리 -> 설치가능** 에서 설치할 수 있다.

## 부팅시 자동 실행

```
[root@windowforsun ~]# chkconfig jenkins on
```  

---
## Reference
[Centos7 Jenkins 설치](https://goddaehee.tistory.com/82)  
