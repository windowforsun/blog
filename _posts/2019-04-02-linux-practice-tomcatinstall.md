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
- 파일 다운로드

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

-  http://apache.mirror.cdnetworks.com/tomcat 에서 다른 버전을 다운 받을 수 있다.
- 압축 풀기

```
[root@windowforsun ~]# tar xzf apache-tomcat-8.5.39.tar.gz -C /usr/local/
```  

- 파일 이동

```
[root@windowforsun ~]# mv /usr/local/apache-tomcat-8.5.39/ /usr/local/tomcat8
```  

## 서비스 구성
- /etc/init.d/tomcat8 파일을 만들어 service 로 등록한다.

```
[root@windowforsun ~]# ll /etc/init.d/tomcat8
ls: cannot access /etc/init.d/tomcat8: No such file or directory
[root@windowforsun ~]# touch /etc/init.d/tomcat8
[root@windowforsun ~]# chmod 755 /etc/init.d/tomcat8
[root@windowforsun ~]# vi /etc/init.d/tomcat8
```  

- /etc/init.d/tomcat8 파일의 내용

```
#!/bin/bash  
#JAVA_HOME=/usr/java/jdk
#export JAVA_HOME
#JRE_HOME=/usr/java/jre
#export JRE_HOME
#PATH=$JAVA_HOME/bin:$PATH  
#export PATH

# chkconfig: 345 90 90
# description: init file for tomcat
# processname: tomcat

CATALINA_HOME="/usr/local/tomcat8"
NAME="$(basename $0)"
case $1 in  
start)  
sh $CATALINA_HOME/bin/startup.sh  
;;   
stop)     
sh $CATALINA_HOME/bin/shutdown.sh  
;;   
status)
if [ -f "/var/run/${NAME}.pid" ]; then
	read kpid < /var/run/${NAME}.pid
	if [ -d "/proc/${kpid}" ]; then
		echo "${NAME} (pid ${kpid}) is running..."
	fi
else
	pid="$(/usr/bin/pgrep -d , java)"
	if [ -z "$pid" ]; then
		echo "${NAME} is stopped"
	else
		echo "${NAME} (pid $pid) is running..."
	fi
fi
;;
restart)  
sh $CATALINA_HOME/bin/shutdown.sh  
sh $CATALINA_HOME/bin/startup.sh  
;;   
version)  
sh $CATALINA_HOME/bin/version.sh  
;;
*)
echo "Usage: $0 {start|stop|restart|status|version}"
;;
esac      
exit 0
```  

### 환경변수 설정

```
[root@windowforsun ~]# vi /etc/profile
```  

- /etc/profile 하단에 아래 내용을 추가한다.

```
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-2.el6_10.x86_64
CATALINA_HOME=/usr/local/tomcat8
CLASSPATH=$JAVA_HOME/lib:$CATALINA_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$CALTALINA_HOME

export JAVA_HOME CLASSPATH PATH CATALINA_HOME
```                                                               

- tomcat8 service 등록 확인

```
[root@windowforsun ~]# service tomcat8
Usage: /etc/init.d/tomcat8 {start|stop|restart|status|version}
```  

## 시작 및 확인

```
[root@windowforsun ~]# service tomcat8 start
Using CATALINA_BASE:   /usr/local/tomcat8
Using CATALINA_HOME:   /usr/local/tomcat8
Using CATALINA_TMPDIR: /usr/local/tomcat8/temp
Using JRE_HOME:        /usr
Using CLASSPATH:       /usr/local/tomcat8/bin/bootstrap.jar:/usr/local/tomcat8/bin/tomcat-juli.jar
Tomcat started.

```  

```
[root@windowforsun ~]# netstat -nlpt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 127.0.0.1:25                0.0.0.0:*                   LISTEN      1572/master
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      1482/sshd
tcp        0      0 ::1:25                      :::*                        LISTEN      1572/master
tcp        0      0 :::8009                     :::*                        LISTEN      7948/java
tcp        0      0 :::8080                     :::*                        LISTEN      7948/java
tcp        0      0 :::22                       :::*                        LISTEN      1482/sshd
```  

```
[root@windowforsun ~]# netstat -anp | grep :8080
tcp        0      0 :::8080                     :::*                        LISTEN      7948/java
```  

```
[root@windowforsun ~]# ps -ef | grep tomcat
root      7948     1  3 14:08 pts/0    00:00:04 /usr/bin/java -Djava.util.logging.config.file=/usr/local/tomcat8/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Dorg.apache.catalina.security.SecurityListener.UMASK=0027 -Dignore.endorsed.dirs= -classpath /usr/local/tomcat8/bin/bootstrap.jar:/usr/local/tomcat8/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat8 -Dcatalina.home=/usr/local/tomcat8 -Djava.io.tmpdir=/usr/local/tomcat8/temp org.apache.catalina.startup.Bootstrap start
root      8030  7397  0 14:10 pts/0    00:00:00 grep tomcat

```  

- http://<서버IP>:8080 접속
	- http://localhost:8080

![톰캣 설치 완료]({{site.baseurl}}/img/linux/linux-tomcat-install-success.png)

## 방화벽 열기

```
[root@windowforsun ~]# iptables -I INPUT 1 -p tcp --dport 8080 -j ACCEPT
[root@windowforsun ~]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
```  

```
[root@windowforsun ~]# service iptables restart
iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
iptables: Flushing firewall rules:                         [  OK  ]
iptables: Unloading modules:                               [  OK  ]
iptables: Applying firewall rules:                         [  OK  ]
```  

- GCP, AWS 등의 클라우드 서버의 경우 VM 인스턴스의 방화벽에 tcp:8080 포트를 추가해준다.

## 부팅시 자동 실행

```
[root@windowforsun ~]# chkconfig --add tomcat8
[root@windowforsun ~]# chkconfig tomcat8 on
```  

---
## Reference
[리눅스 톰캣7 컴파일 설치](https://zetawiki.com/wiki/%EB%A6%AC%EB%88%85%EC%8A%A4_%ED%86%B0%EC%BA%A37_%EC%BB%B4%ED%8C%8C%EC%9D%BC_%EC%84%A4%EC%B9%98)  
[JDK Tomcat 설치](https://nayha.tistory.com/292)  
[CentOS 6 tomcat 설치, 설정](https://m.blog.naver.com/dawning160723/220977208322)  