--- 
layout: single
classes: wide
title: "[Linux 실습] 일반 계정에 Root 권한 및 명령어 부여"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '일반 계정에 Root 권한을 부여하거나 sudo 권한을 부여하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - root
  - sudo
---  

## 환경
- CentOS 6

## 사용자 계정에 Root 권한 및 명령어 부여하기
### Root 권한 부여
- 권한 부여

```
[root@windowforsun ~]# vi /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin

생략 ...

test:x:500:500::/home/test:/bin/bash    // root 권한을 부여하려는 계정
```  

```
test:x:0:0::/home/test:/bin/bash    // UID 와 GID 를 0으로 변경해 준다.
```  

- root 그룹 포함

```
root:x:0:test   // root 그룹에 포함시켜 준다.
bin:x:1:bin,daemon
daemon:x:2:bin,daemon

생략 ...
```  

## Root 명령어 실행 시키기 (sudo)
- Nginx 혹은 Tomcat 과 같은 서버 계정에 Root 권한을 부여해서 Root 명령어를 실행시킨다.
- /etc/sudoers 파일을 이용한다.
	- sudoers 이란 일반 계정이 sudo 명령어를 이용해서, 임시로 root 권한을 얻을 수 있는 방법이다.
- /etc/sudoers 파일은 쓰기 불가능한 파일이기 때문에 권한을 변경해 준다.

```
[root@windowforsun ~]# ll /etc/sudoers
-r--r-----. 1 root root 3729 Dec  8  2015 /etc/sudoers
[root@windowforsun ~]# chmod u+w /etc/sudoers
[root@windowforsun ~]# ll /etc/sudoers
-rw-r-----. 1 root root 3729 Dec  8  2015 /etc/sudoers
```  

- 부여하고자 하는 계정에 sodu 권한을 부여한다.

	```
	[root@windowforsun ~]# vi /etc/sudoers
	```  

- 마지막 부분에 추가해 준다.
	- 패스워드를 필요하는 경우
	
		```
		# user1 사용자에게 sudo 권한 
        user1    ALL=(ALL)       ALL
         
        # wheel 그룹의 모든 사용자에게 sudo 권한을 부여하는 경우
        %wheel        ALL=(ALL)       ALL
		```  
	- 패스워드 확인이 필요 없는 경우
	
		```
		# 사용자의 경우
        user1        ALL=(ALL)       NOPASSWD: ALL
        
        # 그룹의 경우
        %wheel        ALL=(ALL)       NOPASSWD: ALL
		```  
	
	- 특정 명령어만 허용하는 경우
	
		```
		user1        ALL=(ALL)       /bin/ls, /usr/sbin/ifconfig    # 패스워드 입력후 ls, ifconfig 사용 가능
		```  
		
- Nginx, Apache Tomcat 등의 웹 요청상에서 Root 명령어를 사용하기 위해서 sudo 명령어 사용할 수 있도록 변경

```
[root@windowforsun ~]# vi /etc/sudoers
```  

```
#Defaults  requiretty
```  

- Default requiretty 부분을 주석처리해서 tty 가 아닌 경우에도 sudo 명령어를 수행할 수 있도록 변경한다.
- 변경 후에 /etc/sudoers 파일 권한을 원래대로 수정해준다.

```
[root@windowforsun ~]# chmod u-w /etc/sudoers
[root@windowforsun ~]# ll /etc/sudoers
-r--r-----. 1 root root 3729 Dec  8  2015 /etc/sudoers
```  

---
## Reference
[일반 계정에 root권한 부여](https://itgameworld.tistory.com/75)  
[php exec 에서 root 권한 sudo 설정을 통한 획득 / php 함수 몇개 설명](http://blog.naver.com/PostView.nhn?blogId=anducher&logNo=130033390734&widgetTypeCall=true)  
[centos - 일반 계정 sudoer 만들기](https://sseungshin.tistory.com/82)  