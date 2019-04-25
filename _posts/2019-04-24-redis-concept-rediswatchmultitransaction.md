--- 
layout: single
classes: wide
title: "[Redis 실습] Watch, Multi 로 Transaction 및 Synchronized"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 에서 Transaction 과 Synchronized 동작을 구현해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Transaction
  - Synchronized
---  

## 환경
- CentOS 6
- Redis 5.0

## Redis Transaction
- Redis 에서도 RDB 와 같이 Transaction 관련 수행을 할 수 있지만 방식에는 차이가 있다.
- RDB 와 같은 Lock 의 개념은 지원되지 않는다.

## Transaction, Synchronized 관련 명령어
### MULTI
- 트랜잭션 블록의 시작을 표시한다.
- 후속 명령어인 EXEC 를 사용하여 원자적으로 여러개의 명령어를 실행할 수 있다.

### EXEC
- MULTI 를 시작으로 대기중인 명령을 실행한다.
- WATCH 를 사용 할 경우 해당 해당 키에 대한 변경 여부를 체크 한후 실행한다.
- 명령실행이 실패 했을 경우 null 을 반환한다.

### WATCH
- 트랜잭션 수행 시에 키의 변경 여부를 검사한다.

### UNWATCH
- 트랜잭션 수행 시에 WATCH 로 감시 중인 키를 플러시 감시를 중단한다.

### DISCARD
- MULTI 시작으로 대기중인 명령을 플러시 한다.
- WATCH 를 사용 했을 경우 WATCH 중인 키를 UNWATCH 한다.

## MULTI + EXEC 을 이용한 Transaction

```
[root@windowforsun ~]# redis-cli
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set key1 1
QUEUED
127.0.0.1:6379> incr key1
QUEUED
127.0.0.1:6379> exec
1) OK
2) (integer) 2
127.0.0.1:6379> get key1
"2"
```  

- MULTI 로 시작해서 set, incr 명령을 Queue 에 넣고 EXEC 시점에 모두 한번에 수행 했다.
- 명령어의 결과로 2라는 값을 확인 할 수 있다.

## MULTI + DISCARD 을 이용한 Transaction 취소

```
[root@windowforsun ~]# redis-cli
127.0.0.1:6379> set key1 1
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incr key1
QUEUED
127.0.0.1:6379> incr key1
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> get key1
"1"
```  

- key1 에 먼저 1이라는 값을 설정한다.
- MULTI 로 시작해서 key1 에 incr 명령어를 두 번 수행해서 Queue 에 넣는다.
- DISCARD 를 통해 Queue 에 있는 명령어들을 플러시 한다.
- 결과적으로 아무 명령어도 수행되지 않은 1 값을 확인 할 수 있다.

## WATCH 을 이용한 Optimistic Locking 을 이용한 Synchronized
- Redis 의 Locking 은 RDB 의 Locking 의 개념과 다르다.
- RDB 의 경우 Lock 걸린 부분을 Transaction 이 잡고 있어 다른 Transaction 에서 해당 부분을 접근할 수 없다.
- Redis 는 Sequential 하게 요청을 수행하도록 보장해 주는 역할만 수행한다.
- WATCH 명령어를 사용해서 해당 키에 대한 변경 유무를 검사해 이를 Locking 처럼 사용하고 해당 키에 대한 Synchronized 연산을 수행할 수 있다.

```
[root@windowforsun ~]# redis-cli
127.0.0.1:6379> set key2 1
OK
127.0.0.1:6379> watch key2
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incr key2
QUEUED
127.0.0.1:6379> exec
1) (integer) 2
127.0.0.1:6379> get key2
"2"
```  

- 새로운 키 key2 에 먼저 1 값을 설정한다.
- WATCH + MULTI + EXEC 명령을 사용해서 1 증가를 수행한다.
- 위 명령어를 수행할때 key2 변경되지 않았기 때문에 정상적으로 명령어가 수행되어 2값을 확인할 수 있다. 
- Redis Client A, B 를 키고 WATCH 를 수행하면 아래와 같은 실패 할 경우를 테스트 할 수 있다.
- A 클라이언트에서 WATCH 를 key2 에 설정 한 상태에서 B 클라이언트가 key2 에 incr 명령어를 통해 값을 증가시켰다.
- A 클라이언트 또한 incr 명령어를 큐에 넣고 EXEC 하지만 이미 B 클라이언트에서 key2 를 수정했기 때문에 A 클라이언트의 명령어는 실행되지 않는다.

A|A결과|B|B결과
---|---|---|---
127.0.0.1:6379> set key3 1|OK| | 
127.0.0.1:6379> watch key3|OK| | 
 | |127.0.0.1:6379> incr key3|(integer) 2
127.0.0.1:6379> multi|OK| | 
127.0.0.1:6379> incr key3|QUEUED| | 
127.0.0.1:6379> exec|(nil)| | 
127.0.0.1:6379> get key3|"2"| | 
 | |127.0.0.1:6379> get key3|"2"
 
- Redis 는 RDB 와 다르게 Key 값의 변화에 따라 Sequential 한 요청을 보장하는 장치를 사용하여 Locking 동작이 수행된다.

---
## Reference
[Transactions](https://redis.io/topics/transactions)  
[[Redis] How to use Transaction](https://rocksea.tistory.com/319)  