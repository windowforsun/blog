--- 
layout: single
classes: wide
title: "[Linux 실습] Maven 설치하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Linux 에 Maven 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Maven
---  

## 환경
- CentOS 6
- Maven 3.6.0

## 다운로드 및 설치

```
[root@windowforsun ~]# wget http://mirror.navercorp.com/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
--2019-04-05 05:43:20--  http://mirror.navercorp.com/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
Resolving mirror.navercorp.com... 125.209.216.167
Connecting to mirror.navercorp.com|125.209.216.167|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 9063587 (8.6M) [application/x-gzip]
Saving to: “apache-maven-3.6.0-bin.tar.gz”

100%[===========================================================================>] 9,063,587   2.08M/s   in 7.5s

2019-04-05 05:43:28 (1.15 MB/s) - “apache-maven-3.6.0-bin.tar.gz” saved [9063587/9063587]
```  

```
[root@windowforsun ~]# tar xvf apache-maven-3.6.0-bin.tar.gz
apache-maven-3.6.0/README.txt
apache-maven-3.6.0/LICENSE
apache-maven-3.6.0/NOTICE
apache-maven-3.6.0/lib/
apache-maven-3.6.0/lib/animal-sniffer-annotations.license
apache-maven-3.6.0/lib/checker-compat-qual.license
apache-maven-3.6.0/lib/jcl-over-slf4j.license

생략 ..

apache-maven-3.6.0/lib/maven-compat-3.6.0.jar
apache-maven-3.6.0/lib/wagon-provider-api-3.2.0.jar
apache-maven-3.6.0/lib/wagon-http-3.2.0-shaded.jar
apache-maven-3.6.0/lib/jcl-over-slf4j-1.7.25.jar
apache-maven-3.6.0/lib/wagon-file-3.2.0.jar
apache-maven-3.6.0/lib/maven-resolver-connector-basic-1.3.1.jar
apache-maven-3.6.0/lib/maven-resolver-transport-wagon-1.3.1.jar
apache-maven-3.6.0/lib/maven-slf4j-provider-3.6.0.jar
apache-maven-3.6.0/lib/jansi-1.17.1.jar
```  

```
[root@windowforsun ~]# mv apache-maven-3.6.0 /usr/local/maven
[root@windowforsun ~]# ln -s /usr/local/maven/bin/mvn /usr/bin/mvn
```  

## 환경 변수 설정

```
[root@windowforsun ~]# vi /etc/profile.d/maven.sh
```  

- /etc/profile.d/maven.sh 파일에 아래 내용을 추가한다.

```
#!/bin/bash 
MAVEN_HOME=/srv/maven 
PATH=$MAVEN_HOME/bin:$PATH 
export PATH MAVEN_HOME 
export CLASSPATH=.
```  

## 실행권한 부여

```
[root@windowforsun ~]# chmod +x /etc/profile.d/maven.sh
[root@windowforsun ~]# source /etc/profile.d/maven.sh
```  

## 설치 확인

```
[root@windowforsun ~]# mvn --version
Apache Maven 3.6.0 (97c98ec64a1fdfee7767ce5ffb20918da4f719f3; 2018-10-24T18:41:47Z)
Maven home: /usr/local/maven
Java version: 1.8.0_201, vendor: Oracle Corporation, runtime: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-2.el6_10.x86_64/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "2.6.32-754.11.1.el6.x86_64", arch: "amd64", family: "unix"
```  

---
## Reference
[리눅스 CentOS - 메이븐(Maven) 설치 및 설정](https://www.ihee.com/279)  
