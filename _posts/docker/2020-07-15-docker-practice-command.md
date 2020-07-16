--- 
layout: single
classes: wide
title: "[Docker 실습] Docker 명령 사용하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 명령에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Docker Command
toc: true
use_math: true
---  

## Docker Command
`Docker` 를 실제로 조작하는 명령은 `docker` 로 시작한다. 
`root` 유저일 경우 기본으로 사용 할 수 있고, 
`root` 유저가 아닐 때는 추가 설정이 필요하다.  

`docker` 명령어를 사용해서 `Docker Image` 를 조작하거나, 
`Docker Container` 를 컨트롤 할 수 있다. 
기본적으로 `Docker Image` 는 로컬 이미지 또는 `Docker Daemon` 에 등록된 이미지 저장소(`Docker Hub`) 를 사용할 수 있다.  

`docker` 명령에 대한 설명은 `docker -h` 으로 구성과 명령어 종류를 확인해 볼 수 있다. 

```bash
$ docker -h
Flag shorthand -h has been deprecated, please use --help

Usage:  docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/root/.docker")
  -c, --context string     Name of the context to use to connect to the daemon (overrides DOCKER_HOST env var and default
                           context set with "docker context use")

.. 생략 ..

      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  builder     Manage builds
  config      Manage Docker configs

.. 생략 ..

  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile

.. 생략 ..

  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

Run 'docker COMMAND --help' for more information on a command.
```  

### search
`serach` 는 `Docker Daemon` 에 등록된 이미지 저장소에서 이미지를 검색할 수 있는 명령이다. 
기본으로 `Docker Hub` 라는 이미지 저장소에서 원하는 이미지를 검색해 볼 수 있다.  

```bash
$ docker search <OPTIONS> <검색 이름>
```  

`CentOS` 이미지를 검색한다면 명령을 사용해서 가능하다. 

```bash
$ docker search CentOS
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
centos                             The official build of CentOS.                   6089                [OK]
ansible/centos7-ansible            Ansible on Centos7                              132                                     [OK]
consol/centos-xfce-vnc             Centos container with "headless" VNC session…   117                                     [OK]

.. 생략 ..
```  

명령 출력에서 `OFFICIAL` 가 `[OK]` 인 이미지는 `CentOS` 의 공식 이미지를 의미한다. 
각 OS, 언어, 애플리케이션, 툴 등 다양한 공식 이미지가 구성돼 있다. 
그 외 이미지는 사용자들이 별도로 구성한 이미지이다.  

`Docker Hub` 가 아닌 `Private registry` 를 구성한 경우 `search` 명령어의 대상이 되지 않는 것 같다. 
그리고 `search` 명령은 인자로 전달한 것과 매칭되는 이미지 이름만 검색해주고, 태그는 별도의 `curl` 명령을 사용해야 될 것같다. 

### pull
`pull` 은 원격 저장소에 있는 이미지를 로컬로 받을 수 있는 명령이다. 

```bash
$ docker pull <OPTIONS> <이미지이름>:<TAG>|@DIGEST]
``` 
















































---
## Reference
	