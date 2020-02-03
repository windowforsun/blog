--- 
layout: single
classes: wide
title: "[Docker 실습] "
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Overlay Network
---  

## Overlay Network 란
- Overlay Network 는 도커 데몬 호스트들 간의 분산 네트워크를 구성해 준다.
- 호스트 네트워크의 상단에 있어서 컨테이너 간 통신과 안전한 통신이 가능하도록 한다.

## Docker Swarm 의 기본 네트워크
- `Swarm` 을 통해 서비스를 구성하고 배포하면 `ingress`, `docker_gwbridge` 라는 네트워크가 자동으로 생성된다.

### ingress
- overlay 네트워크 드라이버이다.
- `Swarm` 에 구성된 서비스들의 네트워크 트래픽을 통제한다.
- `Swarm` 을 구성하는 기본 네트워크 드라이버이고, 서비스 생성시 별도로 네트워크 정보를 지정하지 않으면 자동으로 `ingress` 네트워크에 연결된다.

### docker_gwbridge
- `bridge` 네트워크 드라이버이다.
- 

---
## Reference

	