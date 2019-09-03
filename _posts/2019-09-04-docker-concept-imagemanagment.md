--- 
layout: single
classes: wide
title: "[Docker 개념] Docker 이미지 관리하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Concept
    - Dockerfile
    - Docker
---  

## 환경
- CentOS 7

## Dockerfile 이란
- DockerFile 은 이미지를 생성하기 위한 스크립트이다.
- DockerFile 을 빌드하면 완성된 이미지를 얻을 수 있다.
- DockerFile 을 통해 이미지가 어떻게 만들어 졌는지 기록 할 수 있다.
	- Docker 을 사용해서 개발환경을 구성할 때 다양한 환경설정이 있을 수 있다.
	- 이를 기록을 남기며 지속적으로 수정하며 개발환경을 완성시킬 수 있다.
- 배포시 큰 용량을 가지고 있는 이미지 자체를 배포하는 방식보다는 DockerFile 을 배포하고 DockerFile 에서 이미지를 얻는 방식이 더욱 편리할 수 있다.

## Dockerfile 작성하기
- 작성을 위해 임의의 새로운 디렉토리를 생성한다.

	```
	[root@localhost vagrant]# mkdir first-dockerfile
	[root@localhost vagrant]# ll
	total 0
	drwxr-xr-x. 2 root root 6 Sep  3 06:26 first-dockerfile
	[root@localhost vagrant]# cd first-dockerfile/
	[root@localhost first-dockerfile]#
	```  
	
- `Dockerfile` 의 이름으로 파일을 만들고 아래 내용을 추가한다.
	- Dockerfile 의 확장자는 따로 존재하지 않는다. 파일 이름을 `Dockerfile` 로만 해주면 된다.

	```
	[root@localhost first-dockerfile]# vi Dockerfile
	```  
	  
	```
	FROM centos:centos7
	MAINTAINER windowforsun <window_for_sun@naver.com>
	
	RUN yum -y update
	RUN rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
	RUN yum -y install nginx
	RUN echo "daemon off;" >> /etc/nginx/nginx.conf
	
	VOLUME ["/data", "/etc/nginx/conf.d", "/var/log/nginx"]
	
	WORKDIR /etc/nginx
	
	CMD ["/usr/sbin/nginx"]
	
	EXPOSE 80
	EXPOSE 433
	```  
	
## Dockerfile 빌드해서 이미지 만들기
- `docker build` 명령어를 사용해서 `Dockerfile` 에 작성한 스크립트를 기반으로 이미지를 생성한다.

	```
	[root@localhost first-dockerfile]# docker build --tag centos7-nginx:0.0.1 .
	Sending build context to Docker daemon  2.048kB
	Step 1/11 : FROM centos:centos7
	 ---> 67fa590cfc1c
	Step 2/11 : MAINTAINER windowforsun <window_for_sun@naver.com>
	 ---> Running in ac6b6d825206
	Removing intermediate container ac6b6d825206
	 ---> f6323da0e974
	Step 3/11 : RUN yum -y update
	 ---> Running in a225e590cb56
	Loaded plugins: fastestmirror, ovl
	Determining fastest mirrors
	 * base: ty1.mirror.newmediaexpress.com
	 * extras: mirror.kakao.com
	 * updates: mirrors.cat.net
	No packages marked for update
	Removing intermediate container a225e590cb56
	 ---> 7433fdedd79e
	Step 4/11 : RUN rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
	 ---> Running in a63cc8d09ad2
	
	// 생략 ...
	
	Removing intermediate container bdf5f954f3ee
	 ---> 2417750febf7
	Step 11/11 : EXPOSE 433
	 ---> Running in 85afac7f91a8
	Removing intermediate container 85afac7f91a8
	 ---> c6c5c8bb0d2f
	Successfully built c6c5c8bb0d2f
	Successfully tagged centos7-nginx:0.0.1
	```  
	
- `docker images` 로 확인해 보면 `centos7-web` 의 이름으로 생성된 이미지를 확인 할 수 있다.

	```
	[root@localhost first-dockerfile]# docker images
	REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
	centos7-nginx       0.0.1               c6c5c8bb0d2f        35 minutes ago      451MB
	centos              centos7             67fa590cfc1c        13 days ago         202MB
	redis               latest              f7302e4ab3a8        2 weeks ago         98.2MB
	```  
	
## Dockerfile 이미지 컨테이너로 실행하기

- `docker run` 명령어로 Dockerfile 로 생성된 이미지를 실행시킨다.

	```
	[root@localhost first-dockerfile]# docker run --name web -d -p 80:80 -v /data:/data centos7-nginx:0.0.1
	5c28a5194293509d52635531b5a6b24f3c4a41627e73b4483c75d5eade883709
	```  
	
	- `-d` 옵션은 백그라운드 실행 옵션이다.
	- `-p` 옵션을 통해 호스트와 컨테이너의 80 포트를 포워딩 하였다.
	- `-v` 옵션으로 호스트 `/data` 디렉토리를 컨테이너 `/data` 디렉토리와 연결했다.

- `docker ps` 로 실행 중인 컨테이너를 확인하면 web 이라는 이름으로 실행중인 컨테이너를 확인 할 수 있다.

	```
	[root@localhost first-dockerfile]# docker ps
	CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                         NAMES
	5c28a5194293        centos7-nginx:0.0.1   "/usr/sbin/nginx"   12 minutes ago      Up 11 minutes       0.0.0.0:80->80/tcp, 433/tcp   web
	bbb86eeb3bcc        67fa590cfc1c          "/bin/bash"         6 hours ago         Up 6 hours                                        first-container
	```  
	
- `http://<호스트IP>:80` 으로 접속하면 아래와 같은 Nginx 페이지를 확인할 수 있다.

	![그림 1]({{site.baseurl}}/img/docker/concept-imagemanagment-1.png)
	
- 실행 중인 `web` 컨테이너를 종료하고 동일한 URL 로 접속하면 접속이 되지 않는 것을 확인 할 수 있다.

## `docker history`


























































	

---
## Reference
[초보를 위한 도커 안내서 - 설치하고 컨테이너 실행하기](https://subicura.com/2017/01/19/docker-guide-for-beginners-2.html)  
