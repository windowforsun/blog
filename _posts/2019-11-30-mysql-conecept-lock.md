--- 
layout: single
classes: wide
title: "[MySQL 개념] Lock 과 Dead Lock"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'MySQL 의 '
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
    - MySQL
    - Concept
    - Lock
    - Dead Lock
---  

## 환경
- MySQL 8.0
- InnoDB

## 테이블 및 데이터

```sql
create table info (
  id         bigint not null auto_increment,
  name     varchar(255),
  primary key (id)
) engine = InnoDB;

insert into info(name) values('name1');
insert into info(name) values('name2');
insert into info(name) values('name3');
insert into info(name) values('name4');
```  

---
## Reference
[MySQL의 Transaction Isolation Levels](https://jupiny.com/2018/11/30/mysql-transaction-isolation-levels/)   
[MySQL 트랜잭션과 잠금 1](https://idea-sketch.tistory.com/45)   
[MySQL 트랜잭션과 잠금 2](https://idea-sketch.tistory.com/47?category=547413)   
[MySQL lock & deadlock 이해하기](https://www.letmecompile.com/mysql-innodb-lock-deadlock/)   
[동시성 문제를 해결하기 위한 MySQL 잠금 두가지](https://sangheon.com/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C%EB%A5%BC-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%9C-mysql-%EC%9E%A0%EA%B8%88-%EB%91%90%EA%B0%80%EC%A7%80/)   