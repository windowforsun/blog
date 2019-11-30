--- 
layout: single
classes: wide
title: "[MySQL 개념] Transaction 과 Isolation Level"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'MySQL 의 Transaction 과 Isolation Level 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
    - MySQL
    - Concept
    - Transaction
---  

## 환경
- MySQL 8.0
- InnoDB

## Isolation Level 에서 발생할 수 있는 문제점
### Dirty Read
- 한 트랜잭션에서 `COMMIT` 하지 않은 데이터를 다른 트랜잭션에서 읽어올 수 있다.
- T1 에서 A 값을 B 로 수정하고 아직 커밋하지 않은 상태이다.
- T2 에서 테이블을 조회 했을 때 B 라는 결과가 조회 될때, T1 이 `ROLLBACK` 을 한다면 데이터에 문제가 생긴다.

### Non-Repeatable Read
- 한 트랜잭션에서 동일한 `SELECT` 문의 결과가 다를 수 있다.
- T1 이 한 쿼리로 데이터를 조회하고 있다.
- T2 가 데이터를 수정 및 삭제 하고 `COMMIT` 을 한다.
- T1 이 동일한 쿼리로 데이터를 조회를 할 경우 첫번째 쿼리 결과와 데이터가 다를 수 있다.

### Phantom Read
- 이전 `SELECT` 쿼리 결과에 없던 `ROW` 가 조회 될 수 있다.
- T1 이 특정 조건으로 데이터를 조회 한 결과를 얻었다.
- T2 는 T1 과 같은 조건에 해당하는 데이터를 추가 및 삭제 하고 `COMMIT` 을 하지 않았다.
- T1 이 한번 더 같은 조건으로 데이터를 조회하게 되면 데이터가 추가/누락 된다.
- T2 가 `ROLLBACK` 을 할 경우 T1 의 데이터에 문제가 생기게 된다.

## 트랜잭션(Transaction)
- 논리적인 작업셋을 모두 완벽하게 처리하는 기능이다.
- 논리적인 작업셋을 모두 처리하지 못할 경우 원 상태로 복구해서 작업 일부만 적용되는 현상(Partial update)이 발생하지 않게 한다.
- 트랜잭션은 `ACID` 중 데이터의 정합성인 `Atomicity` 를 보장하기 위한 기능이다.
- 트랜잭선에는 레벨이 있는데 이를 `Transaction Isolation Levels` 이라고 하고 아래와 같은 레벨이 있다.
	- READ UNCOMMITTED
	- READ COMMITTED
	- REPEATABLE READ
	- SERIALIZABLE	

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
	
### READ UNCOMMITTED
- 다른 트랜잭션에서 `COMMIT` 되지 않은 데이터들을 읽어 올수 있는 level 이다.
- `COMMIT` 되지 않은 데이터를 읽일 수 있기 때문에 `ROLLBACK` 이 될 경우 문제가 될 수 있다.
	- 이렇게 존재해서는 안 될 데이터를 읽어오는 것을 `dirty read` 라고 한다.
	
1. `transaction1`
	
	```
	mysql> set session transaction isolation level read uncommitted;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> start transaction;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> select * from info;
	+----+-------+
	| id | name  |
	+----+-------+
	|  1 | name1 |
	|  2 | name2 |
	|  3 | name3 |
	|  4 | name4 |
	+----+-------+
	4 rows in set (0.00 sec)
	```  
	
1. `transaction2`

	```
	mysql> update info set name = 'nameupdated' where id = 1;
	Query OK, 1 row affected (0.00 sec)
	Rows matched: 1  Changed: 1  Warnings: 0
	
	mysql> insert into info(name) values('name5');
	Query OK, 1 row affected (0.00 sec)
	```  
	
1. `transaction1`

	```
	mysql> select * from info;
    +----+-------------+
    | id | name        |
    +----+-------------+
    |  1 | nameupdated |
    |  2 | name2       |
    |  3 | name3       |
    |  4 | name4       |
    |  5 | name5       |
    +----+-------------+
    5 rows in set (0.00 sec)
	```  
	
- `transaction2` 에서 아직 `COMMIT` 하지 않은 변경 사항들이 `transaction1` 에서도 확인 가능하다.
- 이렇게 `READ UNCOMMITTED` 는 아래와 같은 3가지 문제점이 발생할 수 있다.
	- `dirty read` : 아직 `COMMIT` 되지 않은 데이터를 읽어 올 수 있다.
	- `non-repeatable read` : 한 트랜잭션에서 동일한 `SELECT` 의 결과가 다를 수 있다.
	- `phantom read` : 이전의 `SELECT` 쿼리의 결과에 없던 `ROW` 가 생길 수 있다.

### READ COMMITTED
- 다른 트랜잭션에서 `COMMIT` 된 데이터만 읽을 수 있는 level 이다.

1. `transaction1`

	```
	mysql> set session transaction isolation level read committed;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> start transaction;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> select * from info;
    +----+-------+
    | id | name  |
    +----+-------+
    |  1 | name1 |
    |  2 | name2 |
    |  3 | name3 |
    |  4 | name4 |
    +----+-------+
    4 rows in set (0.01 sec)
	```  
	
1. `transaction2`

	```
	mysql> start transaction;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> update info set name = 'nameupdated' where id = 1;
	Query OK, 1 row affected (0.00 sec)
	Rows matched: 1  Changed: 1  Warnings: 0
	
	mysql> insert into info(name) values('name5');
	Query OK, 1 row affected (0.00 sec)
	```  
	
1. `transaction1`

	```
	mysql> select * from info;
	+----+-------+
	| id | name  |
	+----+-------+
	|  1 | name1 |
	|  2 | name2 |
	|  3 | name3 |
	|  4 | name4 |
	+----+-------+
	4 rows in set (0.00 sec)
	```  
	
	- `transaction2` 에서 아직 `COMMIT` 을 하지 않았기 때문에 `transaction1` 에서 데이터를 확인할 수 없다.
		- `COMMIT` 되지 않은 데이터는 조회 할 수 없기 때문에 `dirty read` 는 발생하지 않는다.
	
1. `transaction2`

	```
	mysql> commit;
    Query OK, 0 rows affected (0.03 sec)
	```  
	
1. `transaction1`

	```
	mysql> select * from info;
    +----+-------------+
    | id | name        |
    +----+-------------+
    |  1 | nameupdated |
    |  2 | name2       |
    |  3 | name3       |
    |  4 | name4       |
    |  5 | name5       |
    +----+-------------+
    5 rows in set (0.00 sec)
	```  
	
- 위 와 같은 결과를 통해 `READ COMMITTED` 에서는 2가지 문제점이 발생할 수 있다.
	- `non-repeatable read` : 한 트랜잭션에서 동일한 `SELECT` 의 결과가 다를 수 있다.
	- `phantom read` : 이전의 `SELECT` 쿼리의 결과에 없던 `ROW` 가 생길 수 있다.
	
### REPEATABLE READ
- MySQL InnoDB 의 기본 Isolation Level 이다.
- 기본 Isolation Level 로 사용되는 만큼 동시성과 안전성을 잘 갖춘 level 이다.
- 개념적으로 봤을 때 `REPEATABLE READ` 는 `READ COMMITED` 에서 `non-repeatable read` 문제점을 해결한 level 이다.
<!--- MySQL InnoDB 의 `REPEATABLE READ` 의 경우 약간 다른 점이 있기 때문에 예제를 통해 확인해 본다.-->

1. `transaction1`

	```
	mysql> set session transaction isolation level repeatable read;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> start transaction;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> select * from info;
    +----+-------+
    | id | name  |
    +----+-------+
    |  1 | name1 |
    |  2 | name2 |
    |  3 | name3 |
    |  4 | name4 |
    +----+-------+
    4 rows in set (0.00 sec)
	```  
	
1. `transaction2`

	```
	mysql> start transaction;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> update info set name = 'nameupdated' where id = 1;
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0
    
    mysql> insert into info(name) values('name5');
    Query OK, 1 row affected (0.00 sec)
	```  
	
1. `transaction1`

	```
	mysql> select * from info;
	+----+-------+
	| id | name  |
	+----+-------+
	|  1 | name1 |
	|  2 | name2 |
	|  3 | name3 |
	|  4 | name4 |
	+----+-------+
	4 rows in set (0.00 sec)
	```  
	
	- `COMMIT` 되지 않은 데이터는 조회되지 않으므로 `dirty read` 발생하지 않는다.
	
1. `transaction2`

	```
	mysql> commit;
	Query OK, 0 rows affected (0.02 sec)
	```  

1. `transaction1`

	```
	mysql> select * from info;
	+----+-------+
	| id | name  |
	+----+-------+
	|  1 | name1 |
	|  2 | name2 |
	|  3 | name3 |
	|  4 | name4 |
	+----+-------+
	4 rows in set (0.00 sec)
	```  
	
	- 한 트랜잭션에서 동일한 `SELECT` 쿼리의 결과가 같으므로 `non-repeatable read` 도 발생하지 않는다.
	- 이전 `SELECT` 쿼리 결과에 없던 `ROW` 가 생기지 않았으므로 `phantom read` 도 발생하지 않는것 같다.
	
- [MySQL Reference](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) 를 참고해 보면 아래와 같이 설명 되어있다.
	- `READ COMMITTED`
		> Each consistent read, even within the same transaction, sets and reads its own fresh snapshot. For information about consistent reads
	
	- `REPEATABLE READ`
		
		> This is the default isolation level for InnoDB. Consistent reads within the same transaction read the snapshot established by the first read. This means that if you issue several plain (nonlocking) SELECT statements within the same transaction, these SELECT statements are consistent also with respect to each other.
- MySQL 에서 `REPEATABLE READ` 와 `READ COMMITTED` level 에서 `SELECT` 쿼리로 데이터를 조회 할때 lock 을 걸지 않고, snapshot 을 만들어 데이터를 조회한다.
- `READ COMMITED` 는 쿼리 시점의 최신의 snapshot 으로 데이터를 조회하기 때문에, 한 트랜잭션에서 `SELECT` 쿼리의 결과 다른 것을 확인 할 수 있었다.
- `REPEATABLE READ` 는 한 트랜잭션에서 처음으로 데이터를 조회할 때의 snapshot 에서 계속 해서 데이터를 조회 하기때문에, 한 트랜잭션에서 `SELECT` 쿼리 결과가 같았고, `phantom read` 도 발생하지 않았다.
- 위 와 같은 상황에서는 `SELECT` 쿼리 결과가 항상 동일하지만, 다른 트랜잭션에서 변경한(UPDATE, DELETE, INSERT) 데이터를 동일하게 변경할 경우 `SELECT` 결과가 달라질 수 있다.

1. `transaction1`

	```
	mysql> update info set name = 'name5updated' where name = 'name5';
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0
    
    mysql> select * from info;
    +----+--------------+
    | id | name         |
    +----+--------------+
    |  1 | name1        |
    |  2 | name2        |
    |  3 | name3        |
    |  4 | name4        |
    |  5 | name5updated |
    +----+--------------+
    5 rows in set (0.00 sec)
	```  
	
	- `transaction2` 에서 `INSERT` 한 `name5` 를 `transaction1` 에서 `name5updated` 로 변경하자, `transaction1` 에서 조회 되지 않았던 `id = 5` 데이터가 조회 되는 것을 확인 할 수 있다.
		- 위 상황을 통해 `phantom read` 가 발행하는 것을 확인할 수 있다.
	
### SERIALIZABLE
- 동시성의 상당 부분을 포기하고 안전성에 큰 비중을 둔 Isolation Level 이다.
- 한 트랜잭션에서 `SELECT` 쿼리를 사용 할때 항상 `SELECT ... FOR SHARE` 로 변경된다.
- `SERIALIZABLE` 에서는 모든 `ROW` 에 `Share Lock(Read Lock)` 을 걸게 된다.

1. `transaction1`

	```
	mysql> set session transaction isolation level serializable;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> start transaction;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> select * from info;
    +----+-------+
    | id | name  |
    +----+-------+
    |  1 | name1 |
    |  2 | name2 |
    |  3 | name3 |
    |  4 | name4 |
    +----+-------+
    4 rows in set (0.00 sec)
	```  
	
1. `transaction2`

	```
	mysql> start transaction;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> select * from info;
	+----+-------+
	| id | name  |
	+----+-------+
	|  1 | name1 |
	|  2 | name2 |
	|  3 | name3 |
	|  4 | name4 |
	+----+-------+
	4 rows in set (0.00 sec)
	```  
	
	- `Share Lock` 이 걸려 있기 때문에 다른 트랜잭션에서 `SELECT` 쿼리를 통해 조회는 가능하다.
	
1. `transaction2`

	```
	mysql> update info set name = 'nameupdated' where id = 1;
    ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
	```  

	- 위와 같이 `transaction1` 에서 조회한, 즉 `Share Lock` 이 걸린 `ROW` 를 변경하려 하면 아무런 동작이 되지 않고 대기 상태에 빠지게 된다.
	- 대기 상태는 `transaction1` 이 `COMMIT` 을 수행하게 되면 쿼리가 변경 쿼리가 수행되며 빠져나올 수 있고, timeout 시간이 자니면 에러와 함께 데이터는 변경되지 않고 빠져나오게 된다.
	
- 이렇게 `SERIALIZABLE` 은 조회하는 데이터에 모두 `Share Lock` 걸어 수정, 추가, 삭제가 불가능하다.
- Isolation Level 중에서 가장 안전성이 강한 Level 이기 때문에 3가지 문제점 모두 발생하지 않는다.
	- `dirty read` 발생하지 않음
	- `non-repeatable read` 발생하지 않음
	- `phantom read` 발생하지 않음
	
## Isolation Level 에 따른 발생 문제 정리

. |Dirty Read|Non-Repeatable Read|Phantom Read
---|---|---|---
READ UNCOMMITTED|O|O|O
READ COMMITTED|X|O|O
REPEATABLE READ|X|X|O
SERIALIZABLE|X|X|X


---
## Reference
[MySQL의 Transaction Isolation Levels](https://jupiny.com/2018/11/30/mysql-transaction-isolation-levels/)   
[MySQL 트랜잭션과 잠금 1](https://idea-sketch.tistory.com/45)   
[MySQL 트랜잭션과 잠금 2](https://idea-sketch.tistory.com/47?category=547413)   
[MySQL lock & deadlock 이해하기](https://www.letmecompile.com/mysql-innodb-lock-deadlock/)   
[동시성 문제를 해결하기 위한 MySQL 잠금 두가지](https://sangheon.com/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C%EB%A5%BC-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%9C-mysql-%EC%9E%A0%EA%B8%88-%EB%91%90%EA%B0%80%EC%A7%80/)   