--- 
layout: single
classes: wide
title: "[MySQL 개념] Lock 과 Dead Lock"
header:
  overlay_image: /img/mysql-bg.png
excerpt: 'MySQL 의 Lock 의 종류와 개념 및 Dead Lock 에 대해 알아보자'
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


## 잠금(Lock)
- 여러 요청에 의해 같은 자원에 접근하려는 다중 트랜잭션에서 데이터베이스의 일관성과 무결성을 유지하는 직렬화(Serialization) 역할을 수행한다.
- 데이터베이스에서 공유자원에 대한 일관성과 무결성을 유지하는 메커니즘을 `Lock` 이라고 한다.
- `Lock` 의 기본적은 종류로는 `Shared Lock` 과 `Exclusive Lock` 이 있다.
- `Lock` 을 사용하다보면 `Blocking` 과 `Dead Lock` 상태에 빠질 수 있기 때문에 주의 해야 한다.

### Shared Lock
- 공유(Shared) Lock 은 데이터를 일고자 할 때 사용된다.
- `Shared Lock` 은 다른 `Shared Lock` 과는 동시에 접근 가능하지만, `Exclusive Lock` 과는 동시에 접근 할 수 없다.
- 공유자원에 대해 `Shared Lock` 은 동시에 읽는 작업(`Shared Lock`)은 가능하지만, 변경(`Exclusive Lock`) 은 불가능하다.
- 위와 반대로 공유자원에 대해 변경(`Exclusive Lock`) 중인 자원을 동시에 읽는 작업(`Shared Lock`) 은 불가능 하다.

### Exclusive Lock
- 배타적(Exclusive) Lock 은 데이터를 변경하고자 할 때 사용하고, 트랜잭션이 완료될 떄까지 유지된다.
- `Exclusive Lock` 걸려 있는 공유자원은 해당 `Lock` 이 해제 될 때까지 다른 트랜잭션에서 접근 할 수 없다. 접근 할 수 없다는 것은 변경 및 읽기도 불가능 한것을 뜻한다.
- 위와 반대로 공유자원에 어떤 락(`Shared Lock`, `Exclusive Lock`) 이 걸려 있다면, `Exclusive Lock` 은 동시에 걸 수 없다.

### Blocking
- 블로킹(Blocking) 은, `Lock` 경합이 발상해 특정 세션이 작업을 진행하지 못하고 멈춰있는 상태를 뜻한다.
- 서로 다른 `Shared Lock` 은 동시 접근이 가능하기 때문에 `Blocking` 상태에 빠지지 않는다.
- `Shared Lock` 과 `Exclusive Lock` 은 동시 접근이 불가능 하기 때문에 `Blocking` 상태에 빠지게 된다.
- `Blocking` 상태를 해결하는 방법은 경합이 이뤄진 특정 세션에서 `COMMIT` 또는 `ROLLBACK` 을 하는 방법 밖에 없다.
- `Blocking` 이 자주 발생하거나, 시간이 길다면 성능 저하로 인해 전체적인 서비스의 질이 떨어질 수 있다.
- `Lock` 에 의해 발생되는 성능 저하를 최소화 시키는 방법은 아래와 같다.
	1. 트랜잭션의 `Atomicity` 를 훼손하지 않는 선에서 트랜잭션을 최대한 짧게 정의 해야 한다.
	1. 공유 자원을 사용하는 트랜잭션이 동시에 수행되지 않도록 설계해야 한다.
	1. `Blocking` 으로 인해 무한정 대기하는 않는 프로그래밍 기법이 필요하다. (LOCK_TIMEOUT)
	
### Dead Lock
- 교착상태(Dead Lock) 은 두 세션이 각각의 `Lock` 이 설정된 서로의 자원에 접근을 위해 대기하는 상황을 뜻한다.
- 서로가 서로의 `Lock` 이 해제되기를 대기하는 상황이기 때문에 무한 대기에 빠진 생태이다.
- `Dead Lock` 을 방지하는 방법은 아래와 같다.
	- 여러 테이블에 접근하며 발생되는 `Dead Lock` 은 테이블 접근 순서를 같게 처리하면 회피 할 수 있다.
	
## MySQL InnoDB 의 Lock
- 사용자에 필요에 따라 쿼리에 `Lock` 을 설정할 수 있다.
	- `Shared Lock`
		
		```sql
		SELECT ... LOCK IN SHARE MODE
		SELECT ... FOR SHARE 	# MySQL 8.0 부터
		```  
		
	- `Exclusive Lock`
	
		```sql
		SELECT ... FOR UPDATE
		```  

### Shared Lock(S)
- Row-level Lock
- SELECT, INSERT 을 위한 Read Lock
- `S` 이 걸려 있는 동안 다른 트랜잭션에서 `Exclusive Lock` 은 획득 할 수 없지만, 다른 `S` 은 획득 가능하다.
- 한 `ROW` 에 대해 여러 `S` 락 획득이 가능하다.

### Exclusive Lock(x)
- Row-level Lock
- UPDATE, DELETE 를 위한 Write Lock
- `X` 이 걸려 있으면, 다른 트랜잭션에서 해당 `ROW` 에 대해 `S`, `X` 모두 획득하지 못하고 대기 상태에 빠진다.

### Intention Lock
- Table-level Lock
- 테이블에 포함된 `ROW` 에 대해 어떤 Row-level Lock 을 걸지에 대해 알려주는 용도로, 미리 Table-level Lock 을 걸어 테이블에 대한 동시성을 제어한다.
- `Intetion Lock` 은 아래와 같이 2가지로 구분된다.
	- `Intention Shared Lock`
	- `Intention Exclusive Lock`
	
#### Intention SharedLock(IS)
- `SELECT ... FOR SHARE` 실행 될 때 아래와 같은 흐름으로 `Lock` 이 걸리게 된다.
	1. Intention Shared Lock(IS) 이 Table-level 에 걸리게 된다.
	1. 이후 `S` 가 Row-Level 에 걸리게 된다.
	
#### Intention Exclusive Lock(IX)
- `SELECT ... FOR UPDATE` 실행 될 때 아래와 같은 흐름으로 `Lock` 이 걸린다.
	1. Intention Exclusive Lock(IX) 이 Table-level 에 걸린다.
	1. 이후 `X` 가 Row-level 에 걸린다.
	
#### IS, IX 의 동시성 
- `IS`, `IX` 는 여러 트랜잭션에서 동시에 접근 가능하다.
- `IS` 이 걸린 테이블에서 `S` 는 접근 가능하지만, `X`는 접근 할 수 없다.
- `IX` 이 걸린 테이블에서 `S`, `X` 모두 접근 할 수 없다.
- `LOCK TABLE`, `ALTER TABLE`, `DROP TABLE` 이 실행 될때, `IS`, `IX` 모두 접근 할 수 없는 `Lock` 이 걸리게 된다. 
- 위와 반대로 `IS`, `IX` 가 걸려 있는 테이블에 위 쿼리를 실행 할 경우 대기상태에 빠지게 된다.

### Lock 의 종류와 호환성 정리

. |Exclusive Lock(X)|Intention Exclusive Lock(IX)|Shared Lock(S)|Intention Shared Lock(IS)
---|---|---|---|---
Exclusive Lock(X)|Conflict|Conflict|Conflict|Conflict
Intention Exclusive Lock(IX)|Conflict|Compatible|Conflict|Compatible
Shared Lock(S)|Conflict|Conflict|Compatible|Compatible
Intention Shared Lock(IS)|Conflict|Compatible|Compatible|Compatible


### Record Lock
- `primary key`, `unique index` 로 조회할 때, 걸리는 Row-level Lock 이다.
- Row-level Lock 이란 인덱스 레코드에 락을 수행하는 것을 뜻한다.

### Gap Lock(Range Lock)
- `Record Lock` 과 비슷하지만, `WHERE` 범위가 명시되었을 때 해당 범위에 속하는 모든 `ROW` 에 대해 락을 수행한다.
- `negative infinity`(최초 레코드 이전), `positive infinity`(마지막 레코드 이후) 의 개념을 사용해서 범위에 대해 `Lock` 을 수행한다.
- Isolation Levels 가 `READ COMMITTED` 일 경우 `UPDATE`, `DELETE` 에도 락이 수행된다.
- `Gap Lock` 을 통해 `Phantom read` 를 방지한다.
- `Record Lock` 과 `Gap Lock` 의 상황 분류
	- `unique index` 가 걸려있는 컬럼 조건으로 조회할 경우 `Record Lock` 이 수행된다.
	- Index 가 걸려 있지만 `unique index` 가 아니면 조건 범위에 대해 `Gap Lock` 이 수행된다.
	- Index 가 걸려 있지 않다면, 전체 테이블을 스캔해야 하므로 모든 `ROW` 에 대해 `Lock` 이 수행된다.
	
## Next-Key Lock
- `Gap Lock` 과 비슷하고, `REPEATABLE READ` 에서 `Phantom read` 를 방지하기 위해 사용 되는 `Lock` 이다.
- `UPDATE`, `DELETE` 에서 모두 사용 된다.
- `num` 이라는 컬럼에 값이 `5, 10, 15, 20, 25, 30` 과 같이 있다고 가정했을 때 잠금 범위는 아래와 같다.
	- (negative infinity, 5] / (5, 10] / (10, 15] / (15, 20] / (20, 25] / (25, 30] / (30, positive infinity)



















































### Next-Key Lock

### Insert Intention Lock

### AUTO-INC Lock

## MySQL Dead Lock


---
## Reference
[MySQL 트랜잭션과 잠금 2](https://idea-sketch.tistory.com/47?category=547413)   
[MySQL lock & deadlock 이해하기](https://www.letmecompile.com/mysql-innodb-lock-deadlock/)   
[동시성 문제를 해결하기 위한 MySQL 잠금 두가지](https://sangheon.com/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C%EB%A5%BC-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%9C-mysql-%EC%9E%A0%EA%B8%88-%EB%91%90%EA%B0%80%EC%A7%80/)   
[Lock](http://www.dbguide.net/db.db?cmd=view&boardUid=148215&boardConfigUid=9&categoryUid=216&boardIdx=138&boardStep=1)   