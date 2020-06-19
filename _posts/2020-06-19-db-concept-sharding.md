--- 
layout: single
classes: wide
title: ""
header:
  overlay_image: /img/db-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - DB
tags:
  - DB
  - Database
  - Sharding
---  

## DB Sharding
- `Sharding` 이란 단일 논리적인 데이터 집합을 여러 데이터베이스에 저장하는 것을 의미한다.
- 다수개의 시스템에 데이터를 분산시킴으로써 데이터베이스 시스템은 하나의 큰 클러스터를 구성하게 되고,
이를 통해 더 크고 많은 데이터 집합을 저장하고 더 많은 요청을 처리할 수 있게 된다.
- 단일 데이터베이스에 저장하기에 데이터 집합이 너무 크다면 `Sharding` 은 하나의 해결책이 될 수 있다.
- `Sharding` 에는 다양한 전략들이 존재하고 이를 통해 필요에 따라 시스템을 추가할 수 있다.
- 데이터 및 트래픽 증가에 따라 데이터베이스의 구성을 유연하게 대응할 수 있다.
- `Sharding` 은 하나의 테이블을 수평 분할(`Horizontal Partitioning`) 해서 여러 개의 작은 단위로 나눠 물리적으로 다른 위치에 분산해서 저장하는 것을 의미한다.
- 여기서 `Horizontal Partitioning` 과 `Vertical Partitioning` 의 차이는 아래와 같다.
	







































