--- 
layout: single
classes: wide
title: "[Linux 실습] Tomcat 설치하기"
header:
  overlay_image: /img/linux-bg.png
excerpt: 'Linux 에 Java 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Tomcat
---  

## 환경
- CentOS 6
- Java 8(jdk 1.8)
- Tomcat 8.5

## 사전 작업
- [jdk 설치]({{site.baseurl}}{% link _posts/2019-04-01-linux-practice-javainstall.md %})

## 다운로드 및 설치

```
[root@windowforsun ~]# wget http://apache.mirror.cdnetworks.com/tomcat/tomcat-8/v8.5.39/bin/apache-tomcat-8.5.39.tar.gz
--2019-04-02 09:30:19--  http://apache.mirror.cdnetworks.com/tomcat/tomcat-8/v8.5.39/bin/apache-tomcat-8.5.39.tar.gz
Resolving apache.mirror.cdnetworks.com... 14.0.101.165
Connecting to apache.mirror.cdnetworks.com|14.0.101.165|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 9672485 (9.2M) [application/x-gzip]
Saving to: “apache-tomcat-8.5.39.tar.gz”

100%[===========================================================================>] 9,672,485    138K/s   in 94s

2019-04-02 09:31:54 (100 KB/s) - “apache-tomcat-8.5.39.tar.gz” saved [9672485/9672485]
```  






---
## Reference
[CentOS JDK 설치](https://zetawiki.com/wiki/CentOS_JDK_%EC%84%A4%EC%B9%98)  
