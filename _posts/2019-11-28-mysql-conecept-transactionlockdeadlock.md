--- 
layout: single
classes: wide
title: "[MySQL 개념] Transaction    , Lock, DeadLock"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'MySQL 의 Transaction 과 Lock, DeadLock 개념과 종류에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
    - MySQL
    - Concept
    - Transaction
    - Lock
    - DeadLock
---  








## 트랜잭션(Transaction)
- 논리적인 작업셋을 모두 완벽하게 처리하는 기능이다.
- 논리적인 작업셋을 모두 처리하지 못할 경우 원 상태로 복구해서 작업 일부만 적용되는 현상(Partial update)이 발생하지 않게 한다.
- 트랜잭션은 `ACID` 중 데이터의 정합성인 `Atomicity` 를 보장하기 위한 기능이다.
- 트랜잭선에는 레벨이 있는데 이를 `Transaction Isolation Levels` 이라고 하고 아래와 같은 레벨이 있다.
	- READ UNCOMMITED
	- READ COMMITED
	- REPEATABLE READ
	- SERIALIZABLE



































































































---
## Reference
[MySQL의 Transaction Isolation Levels](https://jupiny.com/2018/11/30/mysql-transaction-isolation-levels/)   
[MySQL 트랜잭션과 잠금 1](https://idea-sketch.tistory.com/45)   
[MySQL 트랜잭션과 잠금 2](https://idea-sketch.tistory.com/47?category=547413)   
[MySQL lock & deadlock 이해하기](https://www.letmecompile.com/mysql-innodb-lock-deadlock/)   
[동시성 문제를 해결하기 위한 MySQL 잠금 두가지](https://sangheon.com/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C%EB%A5%BC-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%9C-mysql-%EC%9E%A0%EA%B8%88-%EB%91%90%EA%B0%80%EC%A7%80/)   