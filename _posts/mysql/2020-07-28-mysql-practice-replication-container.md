--- 
layout: single
classes: wide
title: "[MySQL 개념] Replication(복제)"
header:
  overlay_image: /img/mysql-bg.png
excerpt: 'MySQL Replication 과 Docker container 를 사용해서 이를 간단하게 구성해 보자'
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
use_math: true
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
먼저 `MySQL` 에서 `Replication` 을 구성하는 방법에 대해 알아본다. 
예제에서 진행하는 `Replication` 은 1 `Master`, 1 `Slave` 로 간단하게 구성한다.  

### MySQL Replication 설정파일
`MySQL` 컨테이너를 실행하기 전에 `Master`, `Slave` 에서 사용할 설정 파일을 아래와 같이 준비한다. 

```
# /master/custom.cnf

[mysqld]
server-id=1
log-bin=mysql-bin
default-authentication-plugin=mysql_native_password
```  

```
# /slave/custom.cnf

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

### MySQL Container 실행
`Replication` 구성을 위해서는 `Master` 와 `Slave` 간의 통신이 이뤄져야 한다. 
`Master`, `Slave` 는 모두 `Docker container` 이기 때문에 별도의 `Docker network` 를 사용해서 통신이 가능하도록 한다. 
아래 명령어로 `replication-net` 이라는 네트워크를 생성하고, 
컨테이너를 실행할 때 네트워크 옵션으로 지정해 준다. 


```bash 
... host ...
$ docker network create replication-net
0509bbbd889f8c059ea68940fb5116d5a49ba538dc8bdb4039d183ca1e3c7bd3
$ docker run \
--rm \
-d \
--network replication-net \
--name master-db \
-v /master:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
mysql:8
1cc8354b6476e32f737153fb2e6446d1551e7b356ae4f2ccc8d6faa44e68afcb
$ docker run \
--rm \
--network replication-net \
-d \
--name slave-db \
-v /slave:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
mysql:8
64b739db1538a41af0c1aa9a8592d99d84db6eb4ea32782d492044c1713e8dd7
```  

- `-e MYSQL_ROOT_PASSWORD` : `root` 계정 비밀번호의 환경 변수이다. 
`root` 로 설정한다. 

2개의 쉘을 준비하면 빠르고 편한 `Replication` 구성이 가능하다. 
각 쉘에서 아래 명령어로 실행한 컨테이너에 접속하고 `MySQL` 에 로그인 한다. 

```bash
... master ...

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
... slave ...

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
... master ...

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

### Replication 계정 및 권한

그리고 `Master` 에서 `Replication` 을 `Slave` 로 수행하는 `slaveuser` 라는 계정을 아래와 같이 생성하고, 
`Replication` 관련 권한을 부여해준다. 

```bash
... master ...

mysql> create user 'slaveuser'@'%' identified by 'slavepasswd';
Query OK, 0 rows affected (0.05 sec)

mysql> grant replication slave on *.* to 'slaveuser'@'%';
Query OK, 0 rows affected (0.05 sec)
```  

이후 `Slave` 에서 `Replication` 설정을 할때 생성한 `slaveuser` 계정 이름과 `slavepasswd` 계정 비밀번호를 사용하게 된다.  

### DB dump 및 초기 동기화

`Master` 는 데이터베이스, 테이블, 더미 데이터를 생성한 상태이다. 
`Slave` 와 `Replication` 을 위해서는 초기에 `Master` 와 같은 데이터 상태를 `Slave` 에 구성해주어야 한다.  

`Master` 에서 `dump` 파일을 만든 후, 
이 파일을 `Slave` 에 적용하는 방법을 사용한다. 
여기서 중요한 점은 `Master` 에서 `dump` 파일을 만든 후 변경이 일어나서는 안된다는 점이다. 
변경을 막기 위해 먼저 `Master` 에서 아래 명령으로 테이블 락을 걸어 `INSERT`, `UPDATE`, `DELETE` 연산을 막는다. 
서비스를 잠시 중단하는 것도 하나의 방법이 될 수 있다.  

```bash
... master ...

mysql> flush tables with read lock;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into exam(value) values('c');
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
mysql> update exam set value = 'ddd';
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
mysql> delete from exam;
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
```  

수행한 락은 `InnoDB` 엔진을 기준으로 해당 세션을 유지한 동안에는 데이터 변경관련 쿼리를 수행할 수 없다. 
세션이 종료되면 락도 종료되기 때문에, 
세션을 유지 한채 새로운 쉘을 열어 `master-db` 에 접속하고 아래 명령어로 덤프를 뜬다.  

```bash
... master ...

$ docker exec -it master-db /bin/sh
# mysqldump -u root -proot --all-databases > master-dump.db
# ls | grep master-dump
master-dump.db
```  

`mysqldump` 명령에서 `-master-data` 옵션을 사용하면,
`Slave` 에서 필요한 바이너리 로그 정보까지 `change master` 구문으로 작성된다. 
예제 진행을 위해 직접 명령을 수행해 봐야 해서 해당 옵션은 사용하지 않는다.  
특정 테이블만 덤프를 뜨고 싶다면 `mysqldump -u root -proot <테이블이름> > master-dump.db` 와 같이 입력한다. 

그리고 `Slave` 설정에 필요한 바이너리 로그 정보는 아래 명령으로 조회 할 수 있다.

```bash
... master ...

mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 1877
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```  

필요한 정보는 `File` 필드의 현재 로그파일의 이름과 
`Position` 필드의 현재 로그 의 위치 정보이다. 
그리고 락을 풀어준다. 

### Slave Replication 설정
앞서 `Master` 상태를 덤프파일로 남겼고, 
그 시점 데이터에 대한 바이너리 로그 정보도 알고 있는 상태이다.  

이제 `Slave` 에 이 데이터들을 모두 알맞게 적용해주면 `Replication` 이 구성된다. 
먼저 `Master` 의 덤프파일을 호스트로 복사하고,
다시 `Slave` 로 복사한다. 

```bash
... host ...
$ docker cp master-db:master-dump.db /
$ ls | grep master-dump
master-dump.db
$ docker cp master-dump.db slave-db:.

... slave ...
$ docker exec -it slave-db /bin/sh
# ls | grep master-dump
master-dump.db
```  

이제 `Slave` 에서 `Master` 덤프파일을 적용하고 잘 적용 됐는지 확인하면 아래와 같다. 

```bash
... slave ...
# mysql -u root -proot < master-dump.db
# mysql -u root -proot
mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-07-29 03:14:34 |
|  2 | b     | 2020-07-29 03:14:36 |
+----+-------+---------------------+
2 rows in set (0.00 sec)
```  

특정 테이블에 대한 덤프파일일 경우, 
`mysql -u root -proot <테이블이름> < master-dump.db` 처럼 입력한다.  

이제 `change master` 명령을 사용해서 `Master` 설정을 통해 `Replication` 을 수행하는 작업만 남은 상태이다. 
`Master` 설정은 아래와 같이 수행할 수 있다. 

```bash
... slave ...

mysql> change master to
    -> master_host='master-db',
    -> master_port=3306,
    -> master_user='slaveuser',
    -> master_password='slavepassword',
    -> master_log_file='mysql-bin.000003',
    -> master_log_pos=1877;
Query OK, 0 rows affected, 2 warnings (0.04 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```  

`change master` 명령의 각 필드에 대한 설명은 아래와 같다. 
- `master_host` : `Master` 의 호스트 혹은 아이피를 입력한다. 
현재 `Master` 와 `Slave` 는 `replication-net` 네트워크를 사용하고 있기 때문에, 
`Docker container` 이름인 `master-db` 로 입력하면 `Docker network` 에서 해당 컨테이너의 아이피를 찾아 통신을 할 수 있다. 
- `master_port` : `Master` 의 포트를 입력한다. 
- `master_user` : `Master` 에 생성하고 `Replication` 권한을 부여한 `slaveuser` 를 설정한다. 
- `master_password` : `slaveuser` 의 비밀번호를 설정한다. 
- `master_log_file` : `Master` 에서 `show master status\G` 명령 결과 중 `File` 필드의 파일 이름을 설정한다. 
- `master_log_pos` : `show master status\G` 명령 결과 중 `Position` 필드의 값을 설정한다. 

마지막으로 `start slave;` 를 통해 `Replication` 동작을 수행 시킬 수 있다. 
정상동작 여부를 판별하기 위해 우선 `Slave` 에서 `show slave status\G` 명령을 수행하면 아래와 같다. 

```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Connecting to master
                  Master_Host: master-db
                  Master_User: slaveuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 1877

.. 생략 ..

                   Last_Errno: 0
                   Last_Error:

.. 생략 ..

Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 1045
                Last_IO_Error: error connecting to master 'slaveuser@master-db:3306' - retry-time: 60 retries: 7 message: Access denied for user 'slaveuser'@'slave-db.replication-net' (using password: YES)
               Last_SQL_Errno: 0
               Last_SQL_Error:

.. 생략 ..

          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400

.. 생략 ..
1 row in set (0.00 sec)
```  

결과를 확인하면 에러도 발생하지 않고 상태 값과 `Master` 에 대한 설정이 정상적으로 적용된 것을 확인 할 수 있다.  

다음으로 `Master` 와 `Slave` 에서 show processlist\G` 명령으로 현재 실행 중인 프로세스를 확인하면 아래와 같다. 

```bash
.. master ..
mysql> show processlist\G

.. 생략 ..

*************************** 3. row ***************************
     Id: 26
   User: slaveuser
   Host: slave-db.replication-net:52194
     db: NULL
Command: Binlog Dump
   Time: 78
  State: Master has sent all binlog to slave; waiting for more updates
   Info: NULL
3 rows in set (0.00 sec)

.. slave ..
mysql> show processlist\G

.. 생략 ..

*************************** 3. row ***************************
     Id: 12
   User: system user
   Host: connecting host
     db: NULL
Command: Connect
   Time: 56
  State: Waiting for master to send event
   Info: NULL
*************************** 4. row ***************************
     Id: 13
   User: system user
   Host:
     db: NULL
Command: Query
   Time: 44
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
4 rows in set (0.00 sec)
```  

`Master` 프로세스에는 `slaveuser` 가 사용중인 프로세스가 있는 것을 확인 할 수 있다. 
그리고 해당 프로세스는 `slave-db` 와 연결된 것도 확인 가능하다. 
`Slave` 프로세스에는 상태가 `Waiting for master to send event` 인 프로세스와 
상태가 `Slave has read all relay log; waiting for more updates` 인 프로세스가 동작 중인 것을 확인 할 수 있다.  

실제로 `Replication` 이 잘 동작하는지 테스트를 수행하면 아래와 같다. 

```bash
.. master ..
mysql> update exam set value = 'aa' where id = 1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | aa    | 2020-07-29 03:14:34 |
|  2 | b     | 2020-07-29 03:14:36 |
+----+-------+---------------------+
2 rows in set (0.00 sec)

.. slave ..
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | aa    | 2020-07-29 03:14:34 |
|  2 | b     | 2020-07-29 03:14:36 |
+----+-------+---------------------+
2 rows in set (0.00 sec)

.. master ..
mysql> insert into exam(value) values('c');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | aa    | 2020-07-29 03:14:34 |
|  2 | b     | 2020-07-29 03:14:36 |
|  3 | c     | 2020-07-29 03:43:50 |
+----+-------+---------------------+
3 rows in set (0.00 sec)

.. slave ..
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | aa    | 2020-07-29 03:14:34 |
|  2 | b     | 2020-07-29 03:14:36 |
|  3 | c     | 2020-07-29 03:43:50 |
+----+-------+---------------------+
3 rows in set (0.00 sec)
```  

이렇게 `Docker Container` 기반으로 `Replication` 구성을 완성 했다. 
하지만 항상 그렇듯 구성은 구성일 뿐, 관리는 또 다른 문제이다. 


---
## Reference
