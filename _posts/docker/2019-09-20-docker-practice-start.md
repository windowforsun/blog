--- 
layout: single
classes: wide
title: "[Docker 개념] Docker 시작하기"
header:
  overlay_image: /img/docker-bg-2.jpg
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


## Intro
- Docker 공식 홈페이지의 가이드가 잘되어 있어서 이를 정리한 글이다.
- Docker 로 Application 을 구성할 때, 사용하는 개념을 계층 적으로 나열하면 아래와 같다.
	- Stack
	- Services
	- Container
- Container 들이 모여서 Services 를 구성하고, Services 들 모아서 하나의 Stack 을 구성한다.




## Container
Docker 에서 Container 는 정적인 Image 가 메모리가 올라가 실행된 상태와 같다.  
간단한 Docker Image 를 만들고 이를 Container 로 실행 시켜보며 Container 에 대해 알아본다.

- Dockerfile 은 하나의 Container 를 정의하는 파일이다.
- 

































	

---
## Reference
[초보를 위한 도커 안내서 - 설치하고 컨테이너 실행하기](https://subicura.com/2017/01/19/docker-guide-for-beginners-2.html)  
