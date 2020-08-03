--- 
layout: single
classes: wide
title: "[Docker 실습] Healthcheck"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 에서 제공하는 Healthcheck 를 알아보고, Dockerfile과 Docker Compose 에서 사용해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
  - Docker Compose
  - Healthcheck
toc: true
use_math: true
---  

## Healthcheck
`Healthcheck` 를 사용하면 사용자의 정의에 따라 컨테이너의 상태를 체크할 수 있다. 
컨테이너 시작 후 애플리케이션 초기화 과정에서 시간이 소요되는 상황이나, 
특정 예외 상황을 감지하고 관리할 수 있다.  

컨테이너에 `Healthcheck` 가 설정된 상황에서 실행되면 처음에는 `starting` 상태가 된다. 
그리고 정의된 `Healthcheck` 를 수행해서 성공하게 되면 상태는 `healthy` 로 변경되고 서비스가 가능한 상태이다. 
만약 설정된 횟수만큼 실패하게 되면, 상태는 `unhealthy` 로 변경되고 이는 서비스가 불가능한 상태이다.  

만약 여러개 `Healthcheck` 가 정의된 상태인 경우, 
마지막에 선언된 `Healthcheck` 만 적용된다.  

`Healthcheck` 에서 사용할 수 있는 옵션은 아래와 같다. 

옵션|설명|기본값
---|---|---
interval|수행 간격|30s
timeout|타임 아웃 시간|30s
start-period|초기화 시간|0s
retries|실패 최대 횟수|3

`Healthcheck` 수행은 `interval` 간격만큼 컨테이너가 실행 중인 동안 계속해서 실행 된다. 
그리고 `Healthcheck` 동작은 `timeout` 의 임계치를 넘게되면 실패로 판별된다. 
그러므로 `start-period` 와 같은 `Healthcheck` 를 수행하기 전 초기화 시간을 애플리케이션 상황에 맞춰 설정 할 필요가 있다. 
또한 `retries` 의 횟수만큼 실패하게 되면 `Healthcheck` 상태는 `unhealthy` 가 되고 컨테이너는 재시작 된다.  

만약 `Healthcheck` 를 `Shell Script` 로 사용할 때, 
종료 코드에 따른 `Healthcheck` 의 동작은 아래와 같다. 
- 0 : `success` 의 의미로 컨테이너의 애플리케이션이 사용 가능 상태임을 의미한다.  
- 1 : `unhealthy` 인 상태로 컨테이너 혹은 애플리케이션 동작에 문제가 있는 상태이다. 
- 2 : `reserved` 는 `Healthcheck` 에서 예약해둔 종료 코드로 사용해서는 안된다. 

## Healthcheck 사용하기
`Healthcheck` 를 수행하는 동작은 사용자가 정의한 쉘 스크립트를 사용하거나, 
애플리케이션에 요청을 보내는 방식을 주로 사용한다. 

### Dockerfile 
`Dockerfile` 에서 `Healthcheck` 의 사용예시는 아래와 같다. 

```dockerfile
HEALTHCHECK --interval=<주기> --timeout=<실행타임아웃시간> --start-period=<시작주기> --regies=<실패반복횟수> \
    CMD  <명령>
```  

### Docker Compose
사용하는 이미지에 `Healthcheck` 가 미리 정의되지 않은 경우, 
`Docker Compose` 에서 서비스 단위로 `Healthcheck` 를 정의할 수 있다. 
그 예시는 아래와 같다. 

```yaml
services:
    service-1:
      healthcheck:
        test: ["CMD", "<명령>"]
        invertial: 주기
        timeout: 실행타임아웃시간
        retries: 실패반복횟수
```  

### 테스트






























































---
## Reference
[Docker Compose - healthcheck](https://docs.docker.com/compose/compose-file/#healthcheck)  
[Dockerfile - healthcheck](https://docs.docker.com/engine/reference/builder/#healthcheck)  