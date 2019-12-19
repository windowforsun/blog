--- 
layout: single
classes: wide
title: "[DevOps] GoCD 설치 및 기본 사용"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring '
author: "window_for_sun"
header-style: text
categories :
  - DevOps
tags:
  - DevOps
  - GoCD
  - CD
---  


## GoCD

![그림 1]({{site.baseurl}}/img/devops/gocd-introandinstall-1.png)

- GoCd 는 크게 `Server` 와 `Agent` 로 구성된다.
- `Server` 는 Web UI 인터페이스를 제공하고, `Agent` 에게 명령을 내려 모든것을 컨트롤하는 역할을 수행한다.
- `Agent` 는 `Server` 의 명령을 받아 실질적으로 명령어를 실행해 작업을 하는 역할을 수행한다.
- `Server` 는 `CD` 관련 역할을 수행하지 않고, `CD` 관련 역할을 `Agent` 가 수행한다.















































	
---
## Reference
[INTRODUCTION TO GoCD](https://www.gocd.org/getting-started/part-1/)