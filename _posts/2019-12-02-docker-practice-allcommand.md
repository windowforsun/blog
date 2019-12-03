--- 
layout: single
classes: wide
title: "[Docker 실습] 전체 수행 명령어"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 에서 쌓인 것들을 정리해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Practice
---  

## 모든 컨테이너 중지 (Stop all docker containers)

```
$ docker stop `docker ps -qa` #
$ docker stop $(docker ps -qa)
```  

## 모든 컨테이너 삭제 (Remove all docker containers)

```
$ docker rm `docker ps -qa`
$ docker rm $(docker ps -qa)
```  

## 모든 도커 이미지 삭제 (Remove all docker images)

```
$ docker rmi `docker images -qa`
$ docker rmi $(docker images -qa)
```  

## 모든 도커 볼륨 삭제 (Remove all docker volumes)

```
$ docker volume rm `docker volume ls -qf`
$ docker volume rm $(docker volume ls -qf)
```  

## 모든 도커 네트워크 삭제 (Remove all docker networks)

```
$ docker network rm `docker network ls -q`
$ docker network rm $(docker network ls -q)
$ docker network prune # 사용하지 않는 모든 network 삭제하기
```  
	
---
## Reference
