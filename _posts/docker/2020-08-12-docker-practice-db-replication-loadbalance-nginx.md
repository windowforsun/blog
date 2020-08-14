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
toc: true
use_math: true
---  

## DataBase Replication LoadBalancing
[DataBase Replication]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %})
을 하게 되면 부하 분산을 위해 `Slave` 개수를 늘리게 될 수 있다. 
이런 상황에서 애플리케이션은 여러개 `Slave` 커넥션을 관리해야 되고, 로드 밸런싱까지 수행해야 할 수 있다. 
하지만 애플리케이션에서 `Slave` 커넥션에 대한 로드밸런싱을 수행하게 될 경우, 
`Failover` 에 대한 처리가 돼있지 않다면 일부 요청이 처리 불가능한 상태가 될 수 있고 이는 서비스 불가 현상으로 확장 될 수 있다. 
그리고 `Slave` 가 추가되는 상황에서도 애플리케이션에 로드밸런싱 로직이 있다면 배포를 다시 수행해 줘야 한다.  

이런 상황에 대한 해결책으로 별도의 `LoadBalancer` 를 사용해서 해결해 보고자 한다. 
애플리케이션에서 구현한다면 예외 상황에 대한 처리를 보다 커스텀하게 구현할 수 있겠지만, 
이 또한 추가 구현사항이므로 기존 검증된 솔루션을 사용한 방법에 대해 기술한다.  

## 테스트 프로젝트
테스트에 사용하는 프로젝트의 전체 디렉토리 구조는 아래와 같다. 

```bash

```  

## 전체 프로젝트
`Database` 는 `MySQL` 을 사용하고 `Replication` 은 [여기]({{site.baseurl}}{% link _posts/docker/2020-07-31-docker-practice-mysql-replication-template.md %})
와 방법과 동일하게 구성한다. 

전체 프로젝트 디렉토리 구조는 아래와 같다. 

```bash
.
├── db-replication
│   ├── docker-compose-master.yaml
│   ├── docker-compose-slave-1.yaml
│   ├── docker-compose-slave-2.yaml
│   ├── master
│   │   ├── conf.d
│   │   │   └── custom.cnf
│   │   └── init
│   │       └── _init.sql
│   └── slave
│       ├── conf.d
│       │   └── custom.cnf
│       └── init
│           └── replication.sh
├── use-haproxy
|   .. 생략 ..
└── use-nginx
    .. 생략 ..
```  

`db-replication` 디렉토리는 `Master` 서비스와 2개의 `Slave` 서비스로 구성 된다. 




docker network create --driver overlay lb-net
o1q9v29su1nirmfndyn6zdfod











































## Nginx
`Nginx` 는 대중적인 웹 서버 중 하나로, 로드밸런서로도 많이 사용된다. 
하지만 `Nginx Plus` 에서만 `Healthcheck` 기능을 지원한다는 점을 인지해야 한다. 
별도로 [헬스체크 모듈](https://github.com/yaoweibin/nginx_upstream_check_module) 
을 지원하지만 도커 공식 이미지상에서 적요하는 방법에 대해 아직 찾지 못해 헬스체크 기능은 제외하고 구현한다. 























































---
## Reference
[]()  