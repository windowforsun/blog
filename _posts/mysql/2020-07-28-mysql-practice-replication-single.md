--- 
layout: single
classes: wide
title: "[MySQL 개념] Replication(복제)"
header:
  overlay_image: /img/mysql-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
  - MySQL
  - Practice
  - Replication
  - Docker
toc: true
---  

## Replication
서버 `L4` 또는 `L7` 로드밸런싱을 사용해서 부하를 분산 시키는 방법으로 어느정도 해결 가능하다. 
하지만 서버 요청 증가에 따른 부하는 다시 `DB` 로 모이게 된다. 
이를 어느정도 해결 할 수 있는 방법이 바로 `Replication`(복제`) 이다. 

`Replication` 은 복제를 의미하는 것과 같이, 
하나의 `DBMS` 의 데이터를 여러 `DBMS` 에 복제해서 부하를 분산시키는 방법이다. 
또한 이렇게 복제로 구성한 여러대의 `DBMS` 를 사용해 성능을 항샹하는 기법을 `Query Off Loading` 이라고 한다. 

`Replication` 은 아래와 같은 구성을 갖는다. 
- `Master` : 원본 `DBMS` 의 역할이다. 
연결된 `Slave` 들에게 `BinaryLog` 를 전송해 변경사항을 동기화 시킨다. 
주로 `INSERT`, `UPDATE`, `DELETE` 연산을 수행한다. 
- `Slave` : 복제 `DBMS` 의 역할이다. 
`Master` 로 부터 전달받은 변경사항을 자신의 `DBMS` 에 적용해 `Master` 와 동기화를 수행한다. 
주로 `SELECT` 조회 연산을 수행한다. 

여기서 기억해야 할 점은 `Replication` 은 단방향이라는 것이다. 
앞서 설명한 것 처럼 `Replication` 을 해주는 `Master` 와 이를 받아 동기화 시키는 `Slave` 가 나뉜 것처럼, 
`Slave` 를 바탕으로 `Master` 를 동기화 할 수 없다. 
이러한 동작을 위해서는 반대로 `Replication` 설정을 해줘야 한다.  

`Replication` 을 통해 구성한 `DBMS` 도 관리 대상이다. 
부하 분산을 위해 많은 `DBMS` 를 구성했다면, 그만큼 관리대상이 늘어난다는 점을 인지해야 한다.  

지금까지는 `Replication` 에 대해 부하를 분산에 대해서만 언급했지만, 
데이터 백업 용으로도 사용할 수 있다. 
`DBMS` 의 데이터는 전체 서비스에서 중요한 정보이므로 이를 안정적이면서 지속적으로 서비스 가능하도록 구성이 필요하다. 
그 중 하나의 방법이 `Replication` 을 통해 복제본으로 백업 `DBMS` 를 두고 관리하는 방식이다. 

## Replication 구성하기
먼저 `MySQL` 에서 `Replication` 을 구성하는 방법에 대해 알아보고, 
이를 바탕으로 `Docker compose` 파일을 사용해서 전체적인 구성을 한번에 만들어 본다.  

예제에서 진행하는 `Replication` 은 1 `Master`, 1 `Slave` 로 간단하게 구성한다.  


### Docker container 로 구성하기
`MySQL` 컨테이너를 실행하기 전에 `Master`, `Slave` 에서 사용할 설정 파일을 아래와 같이 준비한다. 

#### MySQL Replication 설정파일

```
# master.cnf

[mysqld]
server-id=1
log-bin=mysql-bin
default-authentication-plugin=mysql_native_password
```  

```
# slave.cnf

[mysqld]
server-id=2
# replicate-do-db=test
default-authentication-plugin=mysql_native_password
```  

- `server-id` : `Replication` 을 구성할 때 `Replication Group` 에서 식별을 위한 고유 `ID` 값이다. 
- `log-gin` : `Master` 에서 수행하는 모든 업데이트 항목을 바이너리 파일로그로 님기는데, 
그 파일 이름을 설정한다. 
그리고 해당 로그는 이후 `Slave` 에게 전달된다. 
- `replicate-do-db` : 특정 `DB` 만 `Replication` 할 경우 해당 `DB` 이름을 설정한다. 
현재는 주석 처리 된 상태이다. 
- `default-authentication-plugin=master_native_password` : `MySQL` 에 비밀번호를 사용해서 로그인 하기 위해 설정한다. 
`Docker` 명령어, `Docker compose` 에서 `command` 항목에 `--default-authentication-plugin=master_native_password` 형식으로 추가할 수도 있다. 

위 설정 파일을 정상적으로 적용하기 위해 `chmod 755 <설정파일이름>.cnf` 명령을 실행해 준다.  

아래 `Docker` 명령어로 `master-db`, `slave-db` 컨테이너를 하나씩 실행 시킨다. 

#### MySQL Container 실행

```bash 
$ docker run \
> --rm \
> -d \
> --name master-db \
> -v master.cnf:/etc/mysql/conf.d/master.cnf \
> -e MYSQL_ROOT_PASSWORD=root \
> mysql:8
1cc8354b6476e32f737153fb2e6446d1551e7b356ae4f2ccc8d6faa44e68afcb
$ docker run \
> --rm \
> -d \
> --name slave-db \
> --hostname slave-db \
> -v slave.cnf:/etc/mysql/conf.d/slave.cnf \
MYSQ> -e MYSQL_ROOT_PASSWORD=root \
> mysql:8
64b739db1538a41af0c1aa9a8592d99d84db6eb4ea32782d492044c1713e8dd7
```  

- `-e MYSQL_ROOT_PASSWORD` : `root` 계정 비밀번호의 환경 변수이다. 
`root` 로 설정한다. 

2개의 쉘을 준비하면 빠르고 편한 `Replication` 구성이 가능하다. 
각 쉘에서 아래 명령어로 실행한 컨테이너에 접속하고 `MySQL` 에 로그인 한다. 

```bash
# master

$ docker exec -it master-db /bin/sh
# mysql -u root -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.21 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```  

```bash
# slave

$ docker exec -it slave-db /bin/sh
# mysql -u root -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.21 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```  

먼저 `Master` 에서 테스트로 사용할 데이터 베이스와 테이블 그리고 더미 데이터 몇개를 아래 쿼리로 생성한다. 

```bash
# master

mysql> create database test;
Query OK, 1 row affected (0.04 sec)

mysql> use test;
Database changed
mysql> CREATE TABLE `exam` (
    ->   `id` bigint(20) NOT NULL AUTO_INCREMENT,
    ->   `value` varchar(255) DEFAULT NULL,
    ->   `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
    ->   PRIMARY KEY (`id`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
Query OK, 0 rows affected, 1 warning (0.08 sec)

mysql> insert into exam(value) values('a');
Query OK, 1 row affected (0.02 sec)

mysql> insert into exam(value) values('b');
Query OK, 1 row affected (0.04 sec)

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-07-28 16:50:38 |
|  2 | b     | 2020-07-28 16:50:42 |
+----+-------+---------------------+
2 rows in set (0.00 sec)
```  

#### Replication 계정 및 권한

그리고 `Master` 에서 `Replication` 을 `Slave` 로 수행하는 `slaveuser` 라는 계정을 아래와 같이 생성하고, 
`Replication` 관련 권한을 부여해준다. 

```bash
mysql> create user 'slaveuser'@'%' identified by 'slavepasswd';
Query OK, 0 rows affected (0.05 sec)

mysql> grant replication slave on *.* to 'slaveuser'@'%';
Query OK, 0 rows affected (0.05 sec)
```  

이후 `Slave` 에서 `Replication` 설정을 할때 생성한 `slaveuser` 계정 이름과 `slavepasswd` 계정 비밀번호를 사용하게 된다.  

#### DB dump 및 초기 동기화
`Master` 는 데이터베이스, 테이블, 더미 데이터를 생성한 상태이다. 
`Slave` 와 `Replication` 을 위해서는 초기에 `Master` 와 같은 데이터 상태를 `Slave` 에 구성해주어야 한다.  

`Master` 에서 `dump` 파일을 만든 후, 
이 파일을 `Slave` 에 적용하는 방법을 사용한다. 
여기서 중요한 점은 `Master` 에서 `dump` 파일을 만든 후 변경이 일어나서는 안된다는 점이다. 
변경을 막기 위해 먼저 `Master` 에서 아래 명령으로 테이블 락을 걸어 `INSERT`, `UPDATE`, `DELETE` 연산을 막는다. 
서비스를 잠시 중단하는 것도 다른 방법이 될 수 있다.  

```bash
mysql> flush tables with read lock;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into exam(value) values('c');
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
mysql> update exam set value = 'ddd';
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
mysql> delete from exam;
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
```  








---
## Reference
