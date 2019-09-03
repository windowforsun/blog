--- 
layout: single
classes: wide
title: "[Docker 개념] Docker 설치 및 기본 명령어"
header:
  overlay_image: /img/docker-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Concept
    - Docker
---  

## 환경
- CentOS 7


## Docker 설치하기
- Docker 는 리눅스 기반이기 때문에 리눅스상에 설치를 한다.(Windows, Mac OS 에도 가능하다.)
- 간편하고 빠른 설치를 위해 자동 설치 스크립트를 이용한다.
- Docker 는 리눅스 Kernel 버전이 3.10.x 이상 일 경우에 설치 및 실행이 가능하다.(Centos 에서는 7 부터 해당된다.)
- 설치는 `curl -fsSL https://get.docker.com/ | sh` 명령어를 통해 가능하다.
	- root 사용자가 아닐 경우 sudo 를 붙여 사용한다. `curl -fsSL https://get.docker.com/ | sudo sh`
	
	```  
	[root@localhost vagrant]# curl -fsSL https://get.docker.com/ | sh
	# Executing docker install script, commit: 6bf300318ebaab958c4adc341a8c7bb9f3a54a1a
	Warning: the "docker" command appears to already exist on this system.
	
	If you already have Docker installed, this script can cause trouble, which is
	why we're displaying this warning and provide the opportunity to cancel the
	installation.
	
	If you installed the current Docker package using this script and are using it
	again to update Docker, you can safely ignore this message.
	
	You may press Ctrl+C now to abort this script.
	+ sleep 20
	+ sh -c 'yum install -y -q yum-utils'
	Package yum-utils-1.1.31-50.el7.noarch already installed and latest version
	+ sh -c 'yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo'
	Loaded plugins: fastestmirror
	adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
	grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
	repo saved to /etc/yum.repos.d/docker-ce.repo
	+ '[' stable '!=' stable ']'
	+ sh -c 'yum makecache'
	Loaded plugins: fastestmirror
	Loading mirror speeds from cached hostfile
	 * base: data.aonenetworks.kr
	 * extras: ftp.riken.jp
	 * updates: ftp.jaist.ac.jp
	base                                                                                          | 3.6 kB  00:00:00
	docker-ce-stable                                                                              | 3.5 kB  00:00:00
	extras                                                                                        | 3.4 kB  00:00:00
	updates                                                                                       | 3.4 kB  00:00:00
	Metadata Cache Created
	+ '[' -n '' ']'
	+ sh -c 'yum install -y -q docker-ce'
	Package 3:docker-ce-19.03.1-3.el7.x86_64 already installed and latest version
	If you would like to use Docker as a non-root user, you should now consider
	adding your user to the "docker" group with something like:
	
	  sudo usermod -aG docker your-user
	
	Remember that you will have to log out and back in for this to take effect!
	
	WARNING: Adding a user to the "docker" group will grant the ability to run
	         containers which can be used to obtain root privileges on the
	         docker host.
	         Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
	         for more information.
	```  

- 설치 및 버전 확인을 위해 `docker version` 을 하면 아래와 같은 메시지가 출력된다.

	```
	[root@localhost vagrant]# docker version
	Client: Docker Engine - Community
	 Version:           19.03.1
	 API version:       1.40
	 Go version:        go1.12.5
	 Git commit:        74b1e89
	 Built:             Thu Jul 25 21:21:07 2019
	 OS/Arch:           linux/amd64
	 Experimental:      false
	Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
	```  
	
- 데몬이 실행되지 않았기 때문에 `service docker start` 로 Docker daemon 을 실행해 준다.

	```
	[root@localhost vagrant]# service docker start
	Redirecting to /bin/systemctl start docker.service
	```  

- 다시 `docker version` 을 수행하면 아래와 같다.

	```
	[root@localhost vagrant]# docker version
	Client: Docker Engine - Community
	 Version:           19.03.1
	 API version:       1.40
	 Go version:        go1.12.5
	 Git commit:        74b1e89
	 Built:             Thu Jul 25 21:21:07 2019
	 OS/Arch:           linux/amd64
	 Experimental:      false
	
	Server: Docker Engine - Community
	 Engine:
	  Version:          19.03.1
	  API version:      1.40 (minimum version 1.12)
	  Go version:       go1.12.5
	  Git commit:       74b1e89
	  Built:            Thu Jul 25 21:19:36 2019
	  OS/Arch:          linux/amd64
	  Experimental:     false
	 containerd:
	  Version:          1.2.6
	  GitCommit:        894b81a4b802e4eb2a91d1ce216b8817763c29fb
	 runc:
	  Version:          1.0.0-rc8
	  GitCommit:        425e105d5a03fabd737a126ad93d62a9eeede87f
	 docker-init:
	  Version:          0.18.0
	  GitCommit:        fec3683
	```  
	
- `docker version` 의 출력 결과에서 알 수 있듯이 Docker 는 Client 와 Server 로 구성되어 있다.
	- Docker 커멘드를 입력하면 Docker Client 가 Docker Server 로 명령어를 전달하고 결과를 터미널에 출력해준다.
	- 이러한 메커니즘을 통해 Windows 와 Mac OS 에서도 리눅스와 동일하게 Docker 를 사용할 수 있다.
- Docker 는 항상 root 의 권한으로 명령어를 실행해야 한다. 편의를 위해 아래 커멘트를 통해 Docker 명령어를 사용하는 사용자를 Docker 그룹에 추가해 준다.

	```
	[root@localhost vagrant]# usermod -aG docker $USER # 현재 접속중이 사용자를 Docker 그룹에 추가
	[root@localhost vagrant]# usermod -aG docker user_name # user_name 사용자를 Docker 그룹에 추가
	```  

## Docker 기본 명령어
- Docker 의 몇가지 명령어에 대해서만 살펴보는데, 모든 명령어가 궁금할 경우 [docker 명령어](https://docs.docker.com/engine/reference/commandline/docker/) 에서 확인 가능하다.
- Docker 는 [Docker Hub](https://registry.hub.docker.com) 라는 거대한 이미지 공유 생태계를 사용한다.
- 다양한 종류의 리눅스 배포판 뿐만아니라, 오픈 소스 프로젝트 등이 이미지로 Hub 에 올라가 있다.
- Hub 를 통해 필요한 이미지를 검색하고, 내려받고(Pull), 나의 이미지를 올릴(Push) 수도 있다.

### `docker search` 
- Hub 에 있는 이미지들을 검색 할 수 있다.

```
[root@localhost vagrant]# docker search centos
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
centos                             The official build of CentOS.                   5535                [OK]
ansible/centos7-ansible            Ansible on Centos7                              122                                     [OK]
jdeathe/centos-ssh                 CentOS-6 6.10 x86_64 / CentOS-7 7.6.1810 x86…   111                                     [OK]
consol/centos-xfce-vnc             Centos container with "headless" VNC session…   99                                      [OK]
centos/mysql-57-centos7            MySQL 5.7 SQL database server                   62
imagine10255/centos6-lnmp-php56    centos6-lnmp-php56                              57                                      [OK]
tutum/centos                       Simple CentOS docker image with SSH access      45
centos/postgresql-96-centos7       PostgreSQL is an advanced Object-Relational …   39
kinogmt/centos-ssh                 CentOS with SSH                                 29                                      [OK]
pivotaldata/centos-gpdb-dev        CentOS image for GPDB development. Tag names…   10
drecom/centos-ruby                 centos ruby                                     6                                       [OK]
centos/tools                       Docker image that has systems administration…   4                                       [OK]
pivotaldata/centos                 Base centos, freshened up a little with a Do…   3
darksheer/centos                   Base Centos Image -- Updated hourly             3                                       [OK]
mamohr/centos-java                 Oracle Java 8 Docker image based on Centos 7    3                                       [OK]
ovirtguestagent/centos7-atomic     The oVirt Guest Agent for Centos 7 Atomic Ho…   2
miko2u/centos6                     CentOS6 日本語環境                                   2                                       [OK]
pivotaldata/centos-gcc-toolchain   CentOS with a toolchain, but unaffiliated wi…   2
pivotaldata/centos-mingw           Using the mingw toolchain to cross-compile t…   2
indigo/centos-maven                Vanilla CentOS 7 with Oracle Java Developmen…   1                                       [OK]
mcnaughton/centos-base             centos base image                               1                                       [OK]
blacklabelops/centos               CentOS Base Image! Built and Updates Daily!     1                                       [OK]
smartentry/centos                  centos with smartentry                          0                                       [OK]
pivotaldata/centos7-dev            CentosOS 7 image for GPDB development           0
pivotaldata/centos6.8-dev          CentosOS 6.8 image for GPDB development         0
```  

- 이미지의 이름이 오픈소스 혹은 OS 이름과 같다면 공식적인 이미지이다.

### `docker pull` 
- Docker Hub 에서 이미지를 받을 수 있다.
- `docker pull <이미지 이름orID>:<태그>`

```
[root@localhost vagrant]# docker pull centos:lastest
Error response from daemon: manifest for centos:lastest not found: manifest unknown: manifest unknown
[root@localhost vagrant]# docker pull centos:latest
latest: Pulling from library/centos
d8d02d457314: Pull complete
Digest: sha256:307835c385f656ec2e2fec602cf093224173c51119bbebd602c53c3653a3d6eb
Status: Downloaded newer image for centos:latest
docker.io/library/centos:latest
```  

- 위는 `latest` 가장 최신의 centos 이미지를 받은 모습이다.
- Docker 의 켄테이너는 하나의 프로세스 개념임을 잊으면 안된다, 그러기 때문에 호스트 운영체제가 Centos 인 상황에서 Ubuntu 리눅스 켄테이너 사용할 수 있다.
	
### `docker images` 
- 현재 받아져 있는 모든 이미지 목록을 확인 할 수 있다.
- `docker images <이미지 이름>` 으로 하면 이미지 이름이 같고 태그가 다른 모든 이미지가 출력된다.

```
[root@localhost vagrant]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              67fa590cfc1c        13 days ago         202MB
redis               latest              f7302e4ab3a8        2 weeks ago         98.2MB
ubuntu              16.04               5e13f8dd4c1a        5 weeks ago         120MB
```  
	
### `docker run` 
- 다운 받은 이미지로 컨테이너를 실행할 수 있다.
- `docker run <옵션> <이미지 이름orID> <실행할 파일>`

	옵션|Desc
	---|---
	-d|detached mode 로 백그라운드 모드
	-p|호스트와 컨테이너의 포트 연결(포트 포워딩)
	-v|호스트와 컨테이너의 디렉토리 연결(마운팅)
	-e|컨테이너 내에서 사용할 환경변수 설정
	-name|컨테이너 이름 설정
	-rm|프로세스 종료시 컨테이너 자동 제거
	-it|-i, -t 를 동시에 사용, 터미널 입력을 위한 옵션
	-link|컨테이너 연결[컨테이너:별칭]

```
[root@localhost vagrant]# docker run -i -t --name first-container centos /bin/bash
[root@8e70aed13d13 /]# ll
total 12
-rw-r--r--.   1 root root 12090 Aug  1 01:10 anaconda-post.log
lrwxrwxrwx.   1 root root     7 Aug  1 01:09 bin -> usr/bin
drwxr-xr-x.   5 root root   360 Sep  3 02:19 dev
drwxr-xr-x.   1 root root    66 Sep  3 02:19 etc
drwxr-xr-x.   2 root root     6 Apr 11  2018 home
lrwxrwxrwx.   1 root root     7 Aug  1 01:09 lib -> usr/lib
lrwxrwxrwx.   1 root root     9 Aug  1 01:09 lib64 -> usr/lib64
drwxr-xr-x.   2 root root     6 Apr 11  2018 media
drwxr-xr-x.   2 root root     6 Apr 11  2018 mnt
drwxr-xr-x.   2 root root     6 Apr 11  2018 opt
dr-xr-xr-x. 109 root root     0 Sep  3 02:19 proc
dr-xr-x---.   2 root root   114 Aug  1 01:10 root
drwxr-xr-x.  11 root root   148 Aug  1 01:10 run
lrwxrwxrwx.   1 root root     8 Aug  1 01:09 sbin -> usr/sbin
drwxr-xr-x.   2 root root     6 Apr 11  2018 srv
dr-xr-xr-x.  13 root root     0 Sep  2 09:58 sys
drwxrwxrwt.   7 root root   132 Aug  1 01:10 tmp
drwxr-xr-x.  13 root root   155 Aug  1 01:09 usr
drwxr-xr-x.  18 root root   238 Aug  1 01:09 var
[root@8e70aed13d13 bin]# exit
exit
[root@localhost vagrant]#
```  

- 위는 centos 이미지를 컨터이너로 실행하고 Bash 쉘을 실행한 상황이다.
- `-i`, 와 `-t` 옵션을 사용해서 실행된 컨테이너의 Bash 쉘에 입출력을 할 수 있도록 한다.
- `--name` 을 통해 컨테이너의 이름을 지정할 수 있다. 사용자가 별도로 지정하지 않으면 Docker 에서 자동으로 생성한다.
	
### `docker ps` 
- 컨테이너의 목록을 확인 할 수 있다.
- `-a` 옵션을 사용할 경우 정지된 컨테이너까지 모드 출력한다.

```
[root@localhost vagrant]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                       PORTS               NAMES
8e70aed13d13        centos              "/bin/bash"              6 minutes ago       Exited (127) 2 minutes ago                       first-container
0103a9f06750        ubuntu:16.04        "/bin/bash"              16 hours ago        Exited (0) 16 hours ago                          nice_volhard
```  
	
### `docker start` 
- 정지한 컨테이너를 다시 시작시킬 수 있다.
- `docker start <컨테이너 이름orID>
- Docker 명령어에서 ID 를 사용할때 `ab1cd`, `ab2cd` 아이디가 있을때 `ab1` 혹은 `ab2` 까지만 입력해도 식별 가능하다.

```
[root@localhost vagrant]# docker start first-container
first-container
[root@localhost vagrant]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8e70aed13d13        centos              "/bin/bash"         8 minutes ago       Up 5 seconds                            first-container
```  
	
### `docker restart` 
- 컨테이너를 재시작 시킬 수 있다.
- `docker restart <컨테이너 이름orID>

```
[root@localhost vagrant]# docker restart first-container
first-container
```  
	
### `docker attach` 
- 실행 중인 컨테이너에 다시 접속할 수 있다.
- `docker attach <컨테이너 이름orID>

```
[root@localhost vagrant]# docker attach first-container
[root@8e70aed13d13 /]# exit
exit
[root@localhost vagrant]#
```  

### `docker exec` 
- 실행중인 안의 명령어를 실행시킬 수 있다.
- `docker exec <컨테이너 이름orID> <명렁어> <매개변수>`

```
[root@localhost vagrant]# docker exec first-container echo "First Container"
First Container
```  
	

### `docker logs`
- 실행중인 컨테이너의 로그를 확인 할 수 있다.
- `docker logs <옵션> <컨테이너 이름orID>`

```
[root@bbb86eeb3bcc /]# exit
exit
[root@localhost vagrant]# docker logs first-container
[root@bbb86eeb3bcc /]# exit
exit
``` 

- 해당 컨테이너에서 실행된 모든 명령어 및 프로그램 로그가 출력된다. 
- Docker 는 표준 스크림 중 `stdout`, `stderr` 를 수집하여 `docker logs` 명령어의 결과로 출력해준다.
	
### `docker stop`
- 실행 중인 컨테이너를 중지 시킬 수 있다.
- `docker stop <컨테이너 이름orID>`

```
[root@localhost vagrant]# docker stop first-container
first-container
[root@localhost vagrant]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```  
	
### `docker rm`
- 만들어진 컨테이너를 삭제할 수 있다.
- `docker rm <컨테이너 이름orID>`

```
[root@localhost vagrant]# docker rm first-container
first-container
[root@localhost vagrant]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS               NAMES
0103a9f06750        ubuntu:16.04        "/bin/bash"              17 hours ago        Exited (0) 17 hours ago                       nice_volhard
```  
	
### `docker rmi` 
- 받은 이미지를 삭제할 수 있다.
- `docker rmi <이미지 이름orID>:<태그>
- 태그를 붙이지 않을 경우 같은 이미지 이름은 모두 삭제 된다.

```
[root@localhost vagrant]# docker rmi centos
Untagged: centos:latest
Untagged: centos@sha256:307835c385f656ec2e2fec602cf093224173c51119bbebd602c53c3653a3d6eb
Deleted: sha256:67fa590cfc1c207c30b837528373f819f6262c884b7e69118d060a0c04d70ab8
Deleted: sha256:877b494a9f30e74e61b441ed84bb74b14e66fb9cc321d83f3a8a19c60d078654
[root@localhost vagrant]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              f7302e4ab3a8        2 weeks ago         98.2MB
ubuntu              16.04               5e13f8dd4c1a        5 weeks ago         120MB
```  	
	
	

---
## Reference
[초보를 위한 도커 안내서 - 설치하고 컨테이너 실행하기](https://subicura.com/2017/01/19/docker-guide-for-beginners-2.html)  
