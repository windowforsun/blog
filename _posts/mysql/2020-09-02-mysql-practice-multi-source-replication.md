--- 
layout: single
classes: wide
title: "[MySQL 실습] Multi Source Replication 구성과 테스트"
header:
  overlay_image: /img/mysql-bg.png
excerpt: 'MySQL 의 Multi Source Replication 구성과 테스트를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
  - MySQL
  - Practice
  - Replication
  - Multi Source Replication
  - Docker
toc: true
use_math: true
---  

## Multi Source Replication
`MySQL` 에서 `Multi Source Replication` 은 [여기]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %})
에서 알아본 `Replication` 구조에서 확장된 개념이다. 
`Master` 와 연결되는 `Channel` 을 사용해서 `Replication` 을 수행하기 때문에, 하나의 `Slave` 에 여러 `Channel` 별로 `Master` 를 설정 할 수 있다. 
찾아본 결과 `MySQL 5.7` 버전 부터 추가된 기능인 것 같다. 

`Multi Source Replication` 구조를 간단하게 그려보면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/mysql/practice_multi_source_replication_1_plantuml.png)

`Shard` 로 인해 테이블이 분리 돼 있는 경우, `Multi Srouce Replication` 을 사용해서 `Slave` 하나를 구성해서 통계를 위한 용도로 사용할 수 있을 것이다. 
그리고 잘 활용하면 전체 데이터베이스와 테이블에 대하 `Merge` 를 수행하건, `Split` 을 수행하는 동작이 가능해 보인다.  


## Multi Source Replication 구성하기
예제를 진행하기 앞서 `Replication` 을 수행할때 로그 파일의 정보를 사용하지 않고 
[GTID](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-concepts.html)
를 사용해서 진행한다. 
`GTID` 는 `Global Transaction Identifier` 의 약자로 각각의 트랙잭션이 고유한 전역 식별자를 갖는 것을 의미한다.  
 
진행되는 예제는 `Docker` 와 `Docker Swarm` 을 사용해서 진행한다. 
다수개의 `Master` 와 `Slave` 에서 공통으로 사용하는 `Overlay Network` 를 아래 명령어로 생성해준다. 

```bash
docker network create --driver overlay multi-source-net
kqfde1nswrssza9k04qdo6zpe
```  

### 데이터가 없는 상태에서 구성하기

![그림 1]({{site.baseurl}}/img/mysql/practice_multi_source_replication_2_plantuml.png)


먼저 `Master` 1개 `Slave` 1개를 사용해서 초기상태부터 구성하는 예제를 진행한다. 

`master-1` 서비스를 구성하는 템플릿 파일인 `docker-compose-master-1.yaml` 의 내용은 아래와 같다. 

```yaml
# docker-compose-master-1.yaml

version: '3.7'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - multi-source-net
    volumes:
      - ./master-1.cnf/:/etc/mysql/conf.d/custom.cnf
      - ./master-1-init.sql/:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 33000:3306

networks:
  multi-source-net:
    external: true
```  

`master-1` 서비스에서 실행되는 `MySQL` 설정 파일인 `master-1.cnf` 파일 내용은 아래와 같다. 

```
# master-1.cnf

[mysql]
server-id=1
default-authentication-plugin=mysql_native_password
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=on
```  

`row-based replication` 을 사용하고, `gtid` 을 활성화 하는 설정으로 구성돼있다.  

`master-1` 의 초기화 스크립트인 `master-1-init.sql` 내용은 아래와 같다. 

```sql
-- master-1-init.sql

create user 'slaveuser1'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser1'@'%';

flush privileges;

create database test;

use test;

CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```  

`slave` 에서 `Replication` 으로 사용할 `slaveuser` 유저를 생성하고, `exam` 테이블을 생성한다.   

`docker stack deploy -c docker-compose-master-1.yaml master-1` 명령으로 서비스를 클러스터에 적용하고, 
`MySQL` 정상 동작 여부를 확인하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose-master-1.yaml master-1
Creating service master-1_db
$ docker exec -it `docker ps -q -f name=master-1` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```  

아래는 `slave` 서비스를 구성하는 `docker-compose-slave.yaml` 템플릿의 내용이다. 

```yaml
# docker-compose-slave.yaml

version: '3.7'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - multi-source-net
    volumes:
      - ./slave.cnf:/etc/mysql/conf.d/custom.cnf

networks:
  multi-source-net:
    external: true
```  

`slave` 서비스의 `MySQL` 설정파일인 `slave.cnf` 의 내용은 아래와 같다. 

```
# slave.cnf

[mysqld]
server-id=100
default-authentication-plugin=mysql_native_password
gtid-mode=on
enforce-gtid-consistency=on
master-info-repository=TABLE
relay-log-info-repository=TABLE
```  

`master-1` 과 동일하게 `gtid` 를 활성화 하고, 
`gtid` 를 사용해서 `Replication` 을 수행할 수 있도록 `repository` 설정을 `TABLE` 로 한다.  

`docker stack deploy -c docker-compose-slave.yaml slave` 명령으로 클러스터에 서비스를 등록하고, 
`MySQL` 동작 여부를 확인하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose-slave.yaml slave
Creating service slave_db
$ docker exec -it `docker ps -q -f name=slave` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 100   |
+---------------+-------+
```  

`master-1` 서비스와 `slave` 서비스에서 각각 실행 중인 `MySQL` 컨테이너에 접속은 아래 명령어를 사용해서 수행한다. 

```bash
.. 컨테이너 접속 ..
$ docker exec -it `docker ps -q -f name=<서비스 및 컨테이너 식별 이름>` /bin/bash

.. 컨테이너에 실행 중인 MySQL 접속 ..
$ docker exec -it `docker ps -q -f name=<서비스 및 컨테이너 식별 이름>` mysql -uroot -proot
```  

먼저 `master-1` 에서 마스터의 상태와 `gtid` 를 확인하면 아래와 같다.  

```bash
.. master-1 ..
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 196
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 19dbb6ff-ecd8-11ea-8fad-02420a000058:1-10
1 row in set (0.00 sec)

mysql> select * from mysql.gtid_executed;
+--------------------------------------+----------------+--------------+
| source_uuid                          | interval_start | interval_end |
+--------------------------------------+----------------+--------------+
| 19dbb6ff-ecd8-11ea-8fad-02420a000058 |              1 |           10 |
+--------------------------------------+----------------+--------------+
1 row in set (0.00 sec)
```  

현재 `master-1` 의 `gtid` 는 `19dbb6ff-ecd8-11ea-8fad-02420a000058` 인 것을 확인 할 수 있다.  

이제 `slave` 서비스에 접속해서 아래 명령어로 `master-1` 과 `Replication` 설정을 한다. 
아래 명령을 보면 `for channel <채널 이름>` 을 사용해서 채널 기반으로 `Replication` 을 구성하는 것을 확인 할 수 있다.  

```bash
.. slave ..
mysql> change master to master_host='master-1_db', master_user='slaveuser', master_password='slavepasswd', master_auto_position=1 for channel 'master-1';
Query OK, 0 rows affected, 2 warnings (0.04 sec)

mysql> start slave for channel 'master-1';
Query OK, 0 rows affected (0.01 sec)
```  

슬레이브의 상태를 확인하면 아래와 같이 `master-1` 의 `gtid` 를 사용해서 `Replication` 이 수행된 것을 확인 할 수 있다. 

```bash
.. slave ..
mysql> show slave status for channel 'master-1'\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master-1_db
                  Master_User: slaveuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 196
.. 생략 ..
             Master_Server_Id: 1
                  Master_UUID: 19dbb6ff-ecd8-11ea-8fad-02420a000058
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
.. 생략 ..
           Retrieved_Gtid_Set: 19dbb6ff-ecd8-11ea-8fad-02420a000058:1-10
            Executed_Gtid_Set: 133a4799-ecd5-11ea-8ab0-02420a00018d:1-5,
19dbb6ff-ecd8-11ea-8fad-02420a000058:1-10
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name: master-1
.. 생략 ..
1 row in set (0.00 sec)
```  

`master-1` 에서 `test` 데이터베이스 조회, `exam` 테이블을 조회한 결과와 `slave` 에서 조회한 결과가 같다. 

```bash
.. master-1 ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.01 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| exam           |
+----------------+
1 row in set (0.00 sec)

mysql> select * from exam;
Empty set (0.00 sec)

.. slave ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.01 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| exam           |
+----------------+
1 row in set (0.00 sec)

mysql> select * from exam;
Empty set (0.00 sec)
```  


`master-1` 에서 `exam` 테이블에 데이터를 추가하게 되면 `slave` 에서도 동일하게 조회되는 것을 확인 할 수 있다. 

```bash
.. master-1 ..
mysql> insert into exam(value) values('111');
Query OK, 1 row affected (0.00 sec)

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | 111   | 2020-09-02 05:03:20 |
+----+-------+---------------------+
1 row in set (0.00 sec)

.. slave ..
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | 111   | 2020-09-02 05:03:20 |
+----+-------+---------------------+
1 row in set (0.00 sec)
```  



### 다른 데이터베이스를 사용하는 Master 추가하기

![그림 1]({{site.baseurl}}/img/mysql/practice_multi_source_replication_3_plantuml.png)

구성된 `master-1`, `slave` 에서 `master-1` 과 다른 데이터베이스 이름을 사용하는 `master-2` 를 추가해본다.  

`master-2` 서비스를 구성하는 `docker-compose-master-2.yaml` 템플릿 파일은 아래와 같다. 

```yaml
# docker-compose-master-2.yaml

version: '3.7'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - multi-source-net
    volumes:
      - ./master-2.cnf:/etc/mysql/conf.d/custom.cnf
      - ./master-2-init.sql/:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 33001:3306

networks:
  multi-source-net:
    external: true
```  

템플릿의 대부분 내용이 `master-1` 과 동일하다. 
다음으로 `master-2` 서비스의 `MySQL` 설정 파일인 `master-2.cnf` 내용은 아래와 같다. 

```
# master-2.cnf

[mysqld]
server-id=2
default-authentication-plugin=mysql_native_password
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=on
```  

`master-2` 의 초기화 스크립트인 `master-2-init.sql` 내용은 아래와 같다. 

```sql
-- master-2-init.sql

create user 'slaveuser2'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser2'@'%';

flush privileges;

create database test2;

use test2;

CREATE TABLE `exam22` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam22(value) values('a');
insert into exam22(value) values('b');
insert into exam22(value) values('c');
```  

`master-2` 는 `test2` 데이터베이스를 사용하고, `exam22` 테이블에 3개의 데이터를 추가한 상태로 진행한다. 
그리고 슬레이브에서 `master-2` 를 복제하기 위해 사용할 계정은 `slaveuser2` 이다. 
이렇게 마스터 마다 슬레이브 계정을 따로 사용할 수도 있고, 같은 계정을 사용할 수도 있다. 
같은 계정을 사용하는 것에 대한 내용은 추후에 다루도록 한다. 

`docker stack deploy -c docker-compose-master-2.yaml master-2` 로 클러스터에 적용하고, 
`MySQL` 동작 여부를 확인하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose-master-2.yaml master-2
Creating service master-2_db
$ docker exec -it `docker ps -q -f name=master-2` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
```  

`master-2` 에 접속해서 `gtid` 정보를 확인하면 아래와 같다. 

```bash
.. master-2 ..
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 196
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 209f0525-ecee-11ea-9210-02420a00005c:1-13
1 row in set (0.00 sec)

mysql> select * from mysql.gtid_executed;
+--------------------------------------+----------------+--------------+
| source_uuid                          | interval_start | interval_end |
+--------------------------------------+----------------+--------------+
| 209f0525-ecee-11ea-9210-02420a00005c |              1 |           13 |
+--------------------------------------+----------------+--------------+
1 row in set (0.00 sec)
```  

`slave` 서비스에서 이미 복제중인 `master-1` 과 `master-2` 의 유저나 데이터 중에서 겹치는 부분이 존재하지 않는다. 
이런 경우 `master-1` 에서 진행했던 것과 동일하게 `master-2` 도 `slave` 에서 복제를 수행해주면 된다. 
복제를 수행한 후 `slave` 에서 `master-2` 채널에 대한 복제 상태를 확인하면 아래와 같다. 

```bash
.. slave ..
mysql> change master to master_host='master-2_db', master_user='slaveuser2', master_password='slavepasswd', master_auto_position=1 for channel 'master-2';
Query OK, 0 rows affected, 2 warnings (0.06 sec)

mysql> start slave for channel 'master-2';
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status for channel 'master-2'\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master-2_db
                  Master_User: slaveuser2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 196
               Relay_Log_File: a5ffa0a360c9-relay-bin-master@002d2.000003
                Relay_Log_Pos: 411
.. 생략 ..
             Master_Server_Id: 2
                  Master_UUID: 209f0525-ecee-11ea-9210-02420a00005c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
.. 생략 ..
           Retrieved_Gtid_Set: 209f0525-ecee-11ea-9210-02420a00005c:1-13
            Executed_Gtid_Set: 133a4799-ecd5-11ea-8ab0-02420a00018d:1-5,
19dbb6ff-ecd8-11ea-8fad-02420a000058:1-11,
209f0525-ecee-11ea-9210-02420a00005c:1-13
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name: master-2
.. 생략 ..
1 row in set (0.00 sec)
```  

그리고 복제 상태를 `master-2` 와 `slave` 에서 각각 확인하면 아래와 같다. 

```bash
.. master-2 ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test2              |
+--------------------+
5 rows in set (0.01 sec)

mysql> use test2;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------+
| Tables_in_test2 |
+-----------------+
| exam22          |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from exam22;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 07:30:16 |
|  2 | b     | 2020-09-02 07:30:16 |
|  3 | c     | 2020-09-02 07:30:16 |
+----+-------+---------------------+
3 rows in set (0.00 sec)


.. slave ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
| test2              |
+--------------------+
6 rows in set (0.00 sec)

mysql> use test2;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------+
| Tables_in_test2 |
+-----------------+
| exam22          |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from exam22;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 07:30:16 |
|  2 | b     | 2020-09-02 07:30:16 |
|  3 | c     | 2020-09-02 07:30:16 |
+----+-------+---------------------+
3 rows in set (0.00 sec)
```  

`master-2` 에서 새로운 데이터를 추가하면 `slave` 에도 정상적으로 적용 되는 것도 확인 할 수 있다. 

```bash
.. master-2 ..
mysql> insert into exam22(value) values('222');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam22;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 07:30:16 |
|  2 | b     | 2020-09-02 07:30:16 |
|  3 | c     | 2020-09-02 07:30:16 |
|  4 | 222   | 2020-09-02 07:50:09 |
+----+-------+---------------------+
4 rows in set (0.00 sec)


.. slave ..
mysql> select * from exam22;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 07:30:16 |
|  2 | b     | 2020-09-02 07:30:16 |
|  3 | c     | 2020-09-02 07:30:16 |
|  4 | 222   | 2020-09-02 07:50:09 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```  


### 같은 데이터베이스를 사용하는 Master 추가하기

![그림 1]({{site.baseurl}}/img/mysql/practice_multi_source_replication_4_plantuml.png)

`master-3` 서비스는 `master-1` 과 같은 데이터베이스, 다른 테이블을 사용한다. 
`slave` 가 `master-3` 를 복제하게 되면 `test` 데이터베이스는 `master-1` 와 `master-3` 에서 복제를 수행하는 구조가 된다.  

`master-3` 서비스를 구성하는 `docker-compose-master-3.yaml` 템플릿과 관련 설정 파일은 아래와 같다. 

```yaml
# docker-compose-master-2.yaml

version: '3.7'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - multi-source-net
    volumes:
      - ./master-3.cnf:/etc/mysql/conf.d/custom.cnf
      - ./master-3-init.sql/:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 33002:3306

networks:
  multi-source-net:
    external: true
```  

```
# master-3.cnf

[mysqld]
server-id=3
default-authentication-plugin=mysql_native_password
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=on
```  

```sql
-- master-3-init.sql

create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;

create database test;

use test;

CREATE TABLE `exam11` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam11(value) values('a');
insert into exam11(value) values('b');
insert into exam11(value) values('c');
```  

`master-3` 는 `master-1` 에서 사용하는 복제용 계정 이름과 동일한 계정이름을 사용한다. 
그리고 `master-1` 과 동일한 `test` 데이터베이스와 `exam11` 이라는 테이블에 3개의 데이터가 추가된 상태로 진행한다.  

`dockers stack deploy -c docker-compose-master-3.yaml master-3` 명령으로 클러스터에 적용하고, 
`MySQL` 정상 동작여부를 확인하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose-master-3.yaml master-3
Creating service master-3_db
$ docker exec -it `docker ps -q -f name=master-3` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
```  

이전 까지는 복제를 진행할 때, `change master to` 명령을 바로 수행해서 진행했다. 
하지만 `master-3` 에서 사용하는 `test` 데이터베이스가 이미 `slave` 에 존재하기 때문에 기존 방법대로 진행은 불가하다. 
가능한 방법으로는 `mysqldump` 명령을 사용해서 `master-3` 의 데이터를 덤프 뜬 후, 이를 `slave` 에 적용하는 방법으로 진행 한다.  

복제구조에서 마스터에 해당하는 노드에서 데이터 변경과 관련된 쿼리를 수행한다면 먼저 아래 명령어로 데이터 읽기만 가능하도록 해야한다.  

```bash
mysql> flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)

mysql> set global read_only=1;
Query OK, 0 rows affected (0.00 sec)

.. 복제 수행 ..

mysql> set global read_only=0;
Query OK, 0 rows affected (0.00 sec)

mysql> unlock tables;
Query OK, 0 rows affected (0.00 sec)
```  

`master-3` 의 초기 `gtid` 를 확인하면 아래와 같다. 

```bash
.. master-3 ..
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 196
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: a51764c3-ecf9-11ea-8cea-02420a000062:1-13
1 row in set (0.01 sec)

mysql> select * from mysql.gtid_executed;
+--------------------------------------+----------------+--------------+
| source_uuid                          | interval_start | interval_end |
+--------------------------------------+----------------+--------------+
| a51764c3-ecf9-11ea-8cea-02420a000062 |              1 |           13 |
+--------------------------------------+----------------+--------------+
1 row in set (0.00 sec)
```  

현재 `gtid` 를 기준으로 복제를 하게 되면 `test` 테이블 생성 쿼리부터 수행하기 때문에 복제가 정상적으로 수행되지 않는다. 
데이터 변경관련 동작이 일어나지 않는 상태에서 `reset master` 명령으로 상태를 초기화 해준다. 

```bash
.. master-3 ..
mysql> reset master;
Query OK, 0 rows affected (0.02 sec)

mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 156
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```  

이제 `slave` 컨테이너에 접속해서 `master-3` 데이터를 덤프 뜨고 적용한다. 
그리고 기존과 동일하게 복제를 수행한 다음 상태를 확인하면 아래와 같다.  

```bash
.. slave ..
$ docker exec -it `docker ps -q -f name=slave` /bin/bash
root@a5ffa0a360c9:/# mysqldump --all-databases -flush-privileges --single-transaction --flush-logs --triggers --routines --events -hex-blob --host=master-3_db --port=3306 --user=root  --password=root > mysqlbackup_dump.sql
mysqldump: [Warning] Using a password on the command line interface can be insecure.
root@a5ffa0a360c9:/# mysql -uroot -proot < mysqlbackup_dump.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
root@a5ffa0a360c9:/# mysql -uroot -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
mysql> change master to master_host='master-3_db', master_user='slaveuser', master_password='slavepasswd', master_auto_position=1 for channel 'master-3';
Query OK, 0 rows affected, 2 warnings (0.06 sec)

mysql> start slave for channel 'master-3';
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status for channel 'master-3'\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master-3_db
                  Master_User: slaveuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 196
.. 생략 ..
             Master_Server_Id: 3
                  Master_UUID: a51764c3-ecf9-11ea-8cea-02420a000062
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
.. 생략 ..
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: 133a4799-ecd5-11ea-8ab0-02420a00018d:1-5,
19dbb6ff-ecd8-11ea-8fad-02420a000058:1-11,
209f0525-ecee-11ea-9210-02420a00005c:1-14,
a51764c3-ecf9-11ea-8cea-02420a000062:1
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name: master-3
.. 생략 ..
1 row in set (0.00 sec)
```  

복제 상태를 `master-3` 와 `slave` 에서 확인하고, 데이터가 추가 될때의 상황까지 확인하면 아래와 같다. 

```bash
.. master-3 ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| exam11         |
+----------------+
1 row in set (0.00 sec)

mysql> select * from exam11;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 08:52:43 |
|  2 | b     | 2020-09-02 08:52:43 |
|  3 | c     | 2020-09-02 08:52:43 |
+----+-------+---------------------+
3 rows in set (0.00 sec)


.. slave ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
| test2              |
+--------------------+
6 rows in set (0.00 sec)

mysql> use test;
Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| exam           |
| exam11         |
+----------------+
2 rows in set (0.00 sec)

mysql> select * from exam11;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 08:52:43 |
|  2 | b     | 2020-09-02 08:52:43 |
|  3 | c     | 2020-09-02 08:52:43 |
+----+-------+---------------------+
3 rows in set (0.00 sec)


.. master-3 ..
mysql> insert into exam11(value) values('333');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam11;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 08:52:43 |
|  2 | b     | 2020-09-02 08:52:43 |
|  3 | c     | 2020-09-02 08:52:43 |
|  4 | 333   | 2020-09-02 09:23:32 |
+----+-------+---------------------+
4 rows in set (0.00 sec)


.. slave ..
mysql> select * from exam11;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 08:52:43 |
|  2 | b     | 2020-09-02 08:52:43 |
|  3 | c     | 2020-09-02 08:52:43 |
|  4 | 333   | 2020-09-02 09:23:32 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```  

### 같은 데이터베이스, 같은 테이블을 사용하는 Master 추가하기

![그림 1]({{site.baseurl}}/img/mysql/practice_multi_source_replication_5_plantuml.png)

`master-4` 는 `master-2` 와 같은 데이터베이스, 같은 테이블을 사용한다. 
대부분 내용이 앞서 수행한 사항들과 비슷하기 때문에 필요한 부분만 설명하고 그외 부분은 관련 코드만 나열하는 식으로 진행한다.  

아래는 `master-4` 서비스 구성에 필요한 파일들의 내용이다. 

```yaml
# docker-compose-master-4.yaml

version: '3.7'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - multi-source-net
    volumes:
      - ./master-4.cnf:/etc/mysql/conf.d/custom.cnf
      - ./master-4-init.sql/:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 33003:3306

networks:
  multi-source-net:
    external: true
```  

```
# master-4.cnf

[mysqld]
server-id=4
default-authentication-plugin=mysql_native_password
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=on
```  

```sql
-- master-4-init.sql

create user 'slaveuser2'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser2'@'%';

flush privileges;

create database test2;

use test2;

CREATE TABLE `exam22` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci AUTO_INCREMENT=100000;

insert into exam22(value) values('aa');
insert into exam22(value) values('bb');
insert into exam22(value) values('cc');
```  

`master-4` 에서 사용하는 데이터베이스와 테이블은 `master-2` 에서 사용하는 것과 동일하다. 
여기서 `AUTO_INCREMENT` 값이 `100000` 부터 시작하도록 해서 `master-2` 와 `PRIMARY KEY` 가 겹치지 않도록 설정해준다.  

`docker stack deploy -c docker-compose-master-4.yaml master-4` 명령으로 클러스터에 적용하고 `MySQL` 정상동작 및 `gtid` 를 확인한다. 
그리고 `master-4` 에 추가돼있는 데이터들은 복제 대상이 되지 않도록 `reset master` 를 수행해 준다.  

```bash
$ docker stack deploy -c docker-compose-master-4.yaml master-4
Creating service master-4_db
$ docker exec -it `docker ps -q -f name=master-4` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+




.. master-4 ..
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 196
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 6f6c4bdc-ed8b-11ea-b42d-02420a00000e:1-13
1 row in set (0.00 sec)

mysql> select * from mysql.gtid_executed;
+--------------------------------------+----------------+--------------+
| source_uuid                          | interval_start | interval_end |
+--------------------------------------+----------------+--------------+
| 6f6c4bdc-ed8b-11ea-b42d-02420a00000e |              1 |           13 |
+--------------------------------------+----------------+--------------+
1 row in set (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.05 sec)

mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 156
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```  

이제 `slave` 에 접속해서 `mysqldump` 명령을 사용해서 `master-4` 의 `test2` 데이터베이스에 대한 덤프 파일을 만든다. 
여기서 주의할 점은 DDL 관련 쿼리는 제외하고 데이터만 덤프 파일로 만들어야 한다. 
그래서 `mysqldump` 명령에 `--database test2` 로 데이터베이스를 명시하고, `--no-create-info` 와 `--no-create-db` 를 통해 스키마 관련 생성 쿼리를 포함시키지 않도록 한다. 
그리고 `slave` 의 `gtid_purged` 값이 변경되는 부분을 방지하기 위해 `--set-gtid-purged=OFF` 옵션도 추가해 준다.  

```bash
.. slave ..
root@0923f17161bb:/# mysqldump --databases test2 --set-gtid-purged=OFF -flush-privileges --single-transaction --flush-logs --triggers --routines --events -hex-blob --host=master-4_db --port=3306 --user=root  --password=root --no-create-info --no-create-db > master-4-dump.sql
mysqldump: [Warning] Using a password on the command line interface can be insecure.
root@0923f17161bb:/# mysql -uroot -proot < master-4-dump.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
root@0923f17161bb:/# mysql -uroot -proot
mysql: [Warning] Using a password on the command line interface can be insecure.

mysql> change master to master_host='master-4_db', master_user='slaveuser2', master_password='slavepasswd', master_auto_position=1 for channel 'master-4';
Query OK, 0 rows affected, 2 warnings (0.15 sec)

mysql> start slave for channel 'master-4';
Query OK, 0 rows affected (0.03 sec)

mysql> show slave status for channel 'master-4'\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master-4_db
                  Master_User: slaveuser2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 196
.. 생략 ..
             Master_Server_Id: 4
                  Master_UUID: 6f6c4bdc-ed8b-11ea-b42d-02420a00000e
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
.. 생략 ..
           Retrieved_Gtid_Set: 6f6c4bdc-ed8b-11ea-b42d-02420a00000e:1
            Executed_Gtid_Set: 59d4f62b-ed8a-11ea-a48d-02420a000310:1-8,
6f6c4bdc-ed8b-11ea-b42d-02420a00000e:1,
bfae6f7e-ed88-11ea-9d41-02420a00000c:1-16
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name: master-4
.. 생략 ..
1 row in set (0.00 sec)
```  

`master-4` 복제 까지 정상적으로 완료 됐다. 
`test2.exam22` 테이블은 `master-2`, `master-4` 의 데이터를 복제하고 있는 상태이다. 
`master-2`, `master-4`, `slave` 를 사용해서 정상적으로 동작하고 있는지 테스트를 수행하면 아래와 같다.  

```bash
.. master-4 ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test2              |
+--------------------+
5 rows in set (0.00 sec)

mysql> use test2;
Database changed
mysql> show tables;
+-----------------+
| Tables_in_test2 |
+-----------------+
| exam22          |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from exam22;
+--------+-------+---------------------+
| id     | value | datetime            |
+--------+-------+---------------------+
| 100000 | aa    | 2020-09-03 02:16:29 |
| 100001 | bb    | 2020-09-03 02:16:29 |
| 100002 | cc    | 2020-09-03 02:16:29 |
+--------+-------+---------------------+
3 rows in set (0.00 sec)


.. slave ..
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
| test2              |
+--------------------+
6 rows in set (0.00 sec)

mysql> use test2;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------+
| Tables_in_test2 |
+-----------------+
| exam22          |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from exam22;
+--------+-------+---------------------+
| id     | value | datetime            |
+--------+-------+---------------------+
|      1 | a     | 2020-09-02 07:50:09 |
|      2 | b     | 2020-09-02 07:50:09 |
|      3 | c     | 2020-09-02 07:50:09 |
|      4 | 222   | 2020-09-02 07:50:09 |
| 100000 | aa    | 2020-09-03 02:16:29 |
| 100001 | bb    | 2020-09-03 02:16:29 |
| 100002 | cc    | 2020-09-03 02:16:29 |
+--------+-------+---------------------+
7 rows in set (0.00 sec)


.. master-4 ..
mysql> insert into exam22(value) values('444');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam22;
+--------+-------+---------------------+
| id     | value | datetime            |
+--------+-------+---------------------+
| 100000 | aa    | 2020-09-03 02:16:29 |
| 100001 | bb    | 2020-09-03 02:16:29 |
| 100002 | cc    | 2020-09-03 02:16:29 |
| 100003 | 444   | 2020-09-03 02:26:47 |
+--------+-------+---------------------+
4 rows in set (0.00 sec)


.. slave ..
mysql> select * from exam22;
+--------+-------+---------------------+
| id     | value | datetime            |
+--------+-------+---------------------+
|      1 | a     | 2020-09-02 07:50:09 |
|      2 | b     | 2020-09-02 07:50:09 |
|      3 | c     | 2020-09-02 07:50:09 |
|      4 | 222   | 2020-09-02 07:50:09 |
| 100000 | aa    | 2020-09-03 02:16:29 |
| 100001 | bb    | 2020-09-03 02:16:29 |
| 100002 | cc    | 2020-09-03 02:16:29 |
| 100003 | 444   | 2020-09-03 02:26:47 |
+--------+-------+---------------------+
8 rows in set (0.00 sec)


.. master-2 ..
mysql> insert into exam22(value) values('2222');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam22;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-09-02 07:50:09 |
|  2 | b     | 2020-09-02 07:50:09 |
|  3 | c     | 2020-09-02 07:50:09 |
|  4 | 222   | 2020-09-02 07:50:09 |
|  5 | 2222  | 2020-09-03 02:28:11 |
+----+-------+---------------------+
5 rows in set (0.00 sec)


.. slave ..
mysql> select * from exam22;
+--------+-------+---------------------+
| id     | value | datetime            |
+--------+-------+---------------------+
|      1 | a     | 2020-09-02 07:50:09 |
|      2 | b     | 2020-09-02 07:50:09 |
|      3 | c     | 2020-09-02 07:50:09 |
|      4 | 222   | 2020-09-02 07:50:09 |
|      5 | 2222  | 2020-09-03 02:28:11 |
| 100000 | aa    | 2020-09-03 02:16:29 |
| 100001 | bb    | 2020-09-03 02:16:29 |
| 100002 | cc    | 2020-09-03 02:16:29 |
| 100003 | 444   | 2020-09-03 02:26:47 |
+--------+-------+---------------------+
9 rows in set (0.00 sec)
```  


---
## Reference
[17.1.5 MySQL Multi-Source Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-multi-source.html)  
[MySQL 5.7 Multi-Source Replication – Automatically Combining Data From Multiple Databases Into One](http://mysqlhighavailability.com/mysql-5-7-multi-source-replication-automatically-combining-data-from-multiple-databases-into-one/)  
[MySQL 5.7 multi-source replication](https://www.percona.com/blog/2013/10/02/mysql-5-7-multi-source-replication/)  
[MySQL Multi-Source Replication 기본 사용법](https://mysqldba.tistory.com/155)  
[Mysql 8 multi-source-replication 개선점](https://sarc.io/index.php/mariadb/1782-mysql-8-multi-source-replication-2)  
[MySQL 8.0 - Replication By Channel](http://minsql.com/mysql8/B-4-A-replicationByChannel/)  
[GTID(Global Transaction Identifiers)](https://minsugamin.tistory.com/entry/MySQL-57-GTID)  
