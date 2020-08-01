--- 
layout: single
classes: wide
title: "[Docker 실습] MySQL Replication 템플릿"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Compose 를 사용해서 MySQL Replication 템플릿을 만들어보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Practice
  - MySQL
  - Replication
toc: true
use_math: true
---  

## Docker Compose MySQL Replication
`Replication` 에 대한 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %}) 
에서 확인 할 수 있다. 
위 글에서는 독립 컨테이너로 `MySQL` 을 띄워 직접 하나씩 설정을 했다. 
이를 `Docker Compose` 를 사용해서 템플릿화 시키고, 스크립트로 가능한 부분은 자동화를 해서 구성해본다. 
이 구성은 실환경에서 사용하는 목적이라기 보다는 테스트에 사용하기 위해 구성했음을 미리 알린다.  
`Docker Compose` 템플릿을 사용해서 `Replication` 을 구성하는 것이 목적이기 때문에, 
별도의 `MySQL` 관련 설정은 모두 배제된 상태이다. 

디렉토리 구조는 아래와 같다.  

```bash
.
├── docker-compose-master.yaml
├── docker-compose-slave.yaml
├── master
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       └── _init.sql
└── slave
    ├── conf.d
    │   └── custom.cnf
    └── init
        └── replication.sh
```  

템플릿 사용을 위해서는 `Docker swarm` 이 구성돼있어야 한다. 

## Docker Compose 구성
`Docker Compose` 파일은 `Master`, `Slave` 2개로 구성돼있다. 
- `Master` : `docker-compose-master.yaml`
- `Slave` : `docker-compose-slave.yaml`

`Master` 서비스를 먼저 올리고, 이후 `Slave` 서비스를 올리는 방식으로 확장해 나갈 수 있는 구조이다. 

## Master
`Master` 의 템플릿을 정의하는 `docker-compose-master.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  master-db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - repl-net
    volumes:
      - ./master/conf.d/:/etc/mysql/conf.d
      - ./master/init-sql/:/docker-entrypoint-initdb.d/
    ports:
      - 33000:3306


networks:
  repl-net:
    driver: overlay
```  

- `.services.master-db.image` : `MySQL` 공식 이미지 중 8버전을 사용한다. 
- `.services.master-db.environment` : `root` 계정 비밀번호를 설정한다. 
- `.services.master-db.volumes` : `MySQL` 설정 파일과 초기화 `SQL` 을 마운트 한다. 
- `.services.master-db.ports` : `Master` 에서 사용할 포트를 포워딩 설정한다. 
- `.networks.master-net` : `Docker swarm` 에서 사용할 `overlay` 네트워크를 생성한다. 
이후 `Slave` 들은 이 네트워크를 사용해서 `Master` 와 연결된다. 

`MySQL` 설정 내용이 있는 `master/conf.d/custom.cnf` 의 파일 내용은 아래와 같다. 

```
[mysqld]
server-id=1
log-bin=mysql-bin
default-authentication-plugin=mysql_native_password
```  

`server-id` 를 설정하고, `log-bin` 파일 이름을 설정한다. 

초기 설정을 하는 `master/init/_init.sql` 의 파일 내용은 아래와 같다. 

```sql

create database test;

use test;

CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam(value) values('a');
insert into exam(value) values('b');
insert into exam(value) values('c');

create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;
```  

`Database`, `Table` 및 `Replication` 에서 사용할 계정을 생성한다.  

`Master` 템플릿은 `docker stack deploy -c docker-compose-master.yaml master` 명령으로 아래와 같이 실행할 수 있다. 

```bash
$ docker stack deploy -c docker-compose-master.yaml master
Creating network master_repl-net
Creating service master_master-db
```  

`Master` 스택을 `master` 라는 이름으로 배포하게 되면 `master_master-db` 서비스와 `master_repl-net` 네트워크가 생성된다.  

삭제는 `docker stack rm master` 로 가능하다. 

```bash
docker stack rm master
Removing service master_master-db
Removing network master_repl-net
```  


## Slave
`Slave` 템플릿을 정의하는 `docker-compose-slave.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  slave-db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
    networks:
      - master_repl-net
    volumes:
      - ./slave/conf.d/:/etc/mysql/conf.d
      - ./slave/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34000:3306

networks:
  master_repl-net:
    external: true
```  

- `.services.slave-db.environment` : `root` 계정 비밀 번호 뿐만아니라, `Replication` 설정에 필요한 환경 변수를 설정한다. 
- `.services.slave-db.environment.MASTER_HOST` : `Master` 의 호스트 정보를 설정한다. 
`Docker swarm` 클러스터에서 같은 `overlay` 네트워크를 사용하기 때문에 `Master` 의 서비스 이름을 설정한다. 
- `.services.slave-db.environment.MASTER_PORT` : `Master` 의 `MySQL` 포트 정보를 설정한다. 
- `.services.slave-db.environment.MASTER_USER` : `Replication` 설정 정보를 조회 할 수 있는 `Master` 계정 이름을 설정한다.  
- `.services.slave-db.environment.MASTER_PASSWD` : 위 계정의 비밀번호를 설정한다. 
- `.services.slave-db.environment.MASTER_REPL_USER` : `Master` 에서 생성한 `Replication` 권한이 있는 계정 이름을 설정한다. 
- `.services.slave-db.environment.MASTER_REPL_PASSWD` : `Master` 에서 생성한 `Replication` 권한이 있는 계정의 비밀번호를 설정한다.
- `.services.slave-db.networks` : `Master` 에서 생성한 `repl-net` 을 설정한다. 
- `.services.slave-db.volumes` : `MySQL` 설정 파일과 초기화 스크립트를 마운트한다.
- `.services.slave-db.ports` : `Slave` 에서 사용할 포트를 포워딩 설정한다. 
- `.services.networks` : `Master` 에서 생성한 네트워크를 템플릿에서 사용할 수 있도록 바인딩 한다. 
`Master` 스택 이름이 `master` 일 경우 `master_repl-net` 의 이름을 `external` 을 `true` 로 설정한다. 

`MySQL` 설정 내용이 있는 `slave/conf.d/custom.cnf` 의 파일 내용은 아래와 같다. 

```
[mysqld]
server-id=2
#replicate-do-db=test
default-authentication-plugin=mysql_native_password
```  

`server-id` 를 설정하고, 특정 `Database` 만 `Replication` 이 필요할 경우 `replicate-do-db` 에 데이터베이스 이름을 설정한다. 

초기 설정과 `Replication` 을 자동으로 구성하는 스크립트인  하는 `slave/init/replication.sh` 의 파일 내용은 아래와 같다. 

```shell script
# /bin/bash

# function for query to master
function doMasterQuery() {
  local query="$1"
  local grepStr="$2"
  local command="mysql -h${MASTER_HOST} -u ${MASTER_USER} -p${MASTER_PASSWD} -e \"${query}\""

  if [ ! -z "${grepStr}" ]; then
    command="${command} | grep ${grepStr}"
  fi

  echo `eval "$command"`
}

# function for query to slave
function doSlaveQuery() {
  local query="$1"
  local grepStr="$2"
  local command="mysql -u root -p${MYSQL_ROOT_PASSWORD} -e \"${query}\""

  if [ ! -z "${grepStr}" ]; then
    command="${command} | grep ${grepStr}"
  fi

  echo `eval "$command"`
}

# waiting for master connection
check=null
while [[ ${check} != *"Database"* ]]; do
  check=$(doMasterQuery "show databases;")
  sleep 3
done

# dump master databases
mysqldump -h ${MASTER_HOST} -u ${MASTER_USER} -p${MASTER_PASSWD} --master-data=2 --single-transaction --flush-privileges --routines --triggers --all-databases > /tmp/master-dump.sql

# parsing and make change master query
change_master=$(head -100 /tmp/master-dump.sql | grep 'CHANGE MASTER')
change_master=${change_master#-- }
change_master=${change_master%;}
change_master="${change_master},
MASTER_HOST='${MASTER_HOST}',
MASTER_USER='${MASTER_REPL_USER}',
MASTER_PASSWORD='${MASTER_REPL_PASSWD}'"

# apply dump databases
mysql -uroot -p${MYSQL_ROOT_PASSWORD} < /tmp/master-dump.sql

# start slave
echo $(doSlaveQuery "${change_master}")
echo $(doSlaveQuery "start slave")
```  

스크립트의 순서를 나열하면 아래와 같다. 
1. `Master` 의 `MySQL` 연결이 가능한 상태일 때까지 대기한다. 
1. `Master` 의 모든 데이터베이스와 `change master` 정보를 포함해서 덤프 뜬다. 
1. 덤프 파일 중 `change master` 구문을 파싱하고 필요한 정보를 추가한다. 
1. 덤프 파일을 `Slave` 에 적용한다. 
1. `change master` 내용을 `Slave` 에 적용하고 `Replication` 을 시작한다. 

스크립트 내용을 보면 `mysqlump` 명령으로 `Master` 의 데이터를 덤프 하고 있다. 
여기서 명령의 옵션을 확인해 보면 `--master-data=2` 옵션을 사용해서 현재 `Master` 의 로그 파일 이름과 로그 포지션 정보를 `change master` 구문으로 덤프파일에 포함하도록 한다. 
그리고 `--single-transaction` 을 사용해서 로그 파일 정보와 덤프 파일의 시점을 같도록 한다. 
해당 옵션을 사용하면 덤프 시점에 잠시 `transaction isolation level` 을 `REPEATABLE READ` 로 수정하고, 
`START TRANSACTION` 을 실행해서 `InnoDB` 엔진에서 테이블을 덤프를 원활하게 할 수 있도록 한다. 
주의 할점은 덤프 중 `DDL` 작업이나 `--lock-tables` 옵션과는 함께 사용해서는 안된다. 
만약 데이터의 크기가 클 경우 `--quick` 옵션을 사용할 수도 있다.  

위와 같은 설정을 통해 현재 덤프 과정에서 `Master` 에 데이터가 추가되거나 변경되더라고 `Slave` 에서 지속적인 데이터 동기화가 가능하다.  

`Slave` 템플릿은 `Master` 템플릿을 실행한 상태에서 
`docker stack deploy -c docker-compose-slave.yaml <Slave 스택 이름>` 명령으로 아래와 같이 실행할 수 있다. 

```bash
$ docker stack deploy -c docker-compose-slave.yaml slave-1
Creating service slave-1_slave-db
```  

진행 상태는 `docker service logs -f <Slave 스택 이름_slave-db>` 으로 로그를 확인하는 방식으로 가능하다. 

```bash
docker service logs -f slave-1_sl
ave-db
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01 09:13:25+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.21-1debian10 started.
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01 09:13:25+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01 09:13:25+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.21-1debian10 started.
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01 09:13:25+00:00 [Note] [Entrypoint]: Initializing database files
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01T09:13:25.812088Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.21) initializing of server in progress as process 44

.. 생략 ..

slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01T09:13:59.210935Z 0 [Warning] [MY-010604] [Repl] Neither --relay-log nor --relay-log-index were used; so replication may break when this MySQL server acts as a slave and has his hostname changed!! Please use '--relay-log=6f2ff784d882-relay-bin' to avoid this problem.
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01T09:13:59.342507Z 5 [Warning] [MY-010897] [Repl] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01T09:13:59.345406Z 5 [System] [MY-010562] [Repl] Slave I/O thread for channel '': connected to master 'slaveuser@master-db:3306',replication started in log 'mysql-bin.000003' at position 156
slave-1_slave-db.1.7bfislorbuoj@docker-desktop    | 2020-08-01T09:13:59.346724Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.21'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
```  

만약 `Slave` 를 추가해야 한다면 아래 정보 수정이 필요하다. 
- `docker-compose.slave.yaml` 파일에서 `.service.slave-db.prots` 외부 포트 번호
- `slave/conf.d/custom.cnf` 파일에서 `server-id` 번호
- `docker stack deploy -c docker-compose-slave.yaml <Slave 스택이름>` 에서 기존 `Slave` 와 다른 스택 이름

삭제는 `docker stack rm <Slave 스택 이름>` 로 가능하다. 

```bash
$ docker stack rm slave-1
Removing service slave-1_slave-db
```  

## 테스트
먼저 동시에 `Master` 와 `Slave` 하나를 `Swarm` 클러스터에 배포한다. 

```bash
$ docker stack deploy -c docker-compose-master.yaml master
Creating network master_repl-net
Creating service master_master-db
$ docker stack deploy -c docker-compose-slave.yaml slave-1
Creating service slave-1_slave-db
```  

서비스에서 실행하는 컨테이너에 접속해야 하기 때문에 각 컨테이너 아이디를 조회하면 아래와 같다. 

```bash
$ docker ps | grep master
a1bc867e74c5        mysql:8                      "docker-entrypoint.s…"   3 minutes ago       Up 3 minutes        3306/tcp, 33060/tcp   master_master-db.1.vavutuujn5tzi0unzjux4v3tw
$ docker ps | grep slave-1
bd495adb33d2        mysql:8                      "docker-entrypoint.s…"   3 minutes ago       Up 3 minutes        3306/tcp, 33060/tcp   slave-1_slave-db.1.zt0cphx4cnv9h3ogfik9fqaa1
```  

`Master` 에 데이터의 `exam` 테이블에 데이터를 추가하고, 
`slave-1` 에서 확인하면 아래와 같이 동기화가 진행 중인 것을 확인 할 수 있다. 

```bash
.. Master 컨테이너에 접속해서 데이터 추가 ..
$ docker exec -it a1bc867e74c5 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> insert into exam(value) values('new1');
Query OK, 1 row affected (0.04 sec)

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```  

```bash
.. slave-1 컨테이너에 접속해서 데이터 조회 ..
$ docker exec -it bd495adb33d2 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```  

`slave-2` 라는 스택 이름으로 새로운 `Slave` 를 추가한다. 
외부 포트 번호는 `34001` 이고, `server-id` 는 3으로 수정하고 배포 명령을 수행한다.  

```bash
$ docker stack deploy -c docker-compose-slave.yaml slave-2
Creating service slave-2_slave-db
```  

`slave-2` 의 컨테이너 정보를 확인하고, 
접속해서 `exam` 테이블의 데이터를 확인하면 아래와 같이 데이터 동기화가 진행 됨을 확인할 수 있다. 

```bash
$ docker ps | grep slave-2
1568dd96a780        mysql:8                      "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 33060/tcp   slave-2_slave-db.1.t0106tp0ilrwaj7enxes23dv5
$ docker exec -it 1568dd96a780 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```  

`Master` 하나에 `slave-1`, `slave-2` 가 `Replication` 으로 구성된 상태이다. 
다시 `Master` 에 데이터를 추가하고 `slave-1` 과 `slave-2` 에서 조회하면 모두 동기회가 진행 중인 것을 확인 할 수 있다.  

```bash
.. Master 에 새로운 데이터 추가 ..
$ docker exec -it a1bc867e74c5 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> insert into exam(value) values('new2');
Query OK, 1 row affected (0.01 sec)

mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
|  5 | new2  | 2020-08-01 09:48:37 |
+----+-------+---------------------+
5 rows in set (0.00 sec)
```  

```bash
.. slave-1 에서 데이터 조회 ..
docker exec -it bd495adb33d2 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
|  5 | new2  | 2020-08-01 09:48:37 |
+----+-------+---------------------+
5 rows in set (0.01 sec)
```  

```bash
.. slave-2 에서 데이터 조회 ..
$ docker exec -it 1568dd96a780 /bin/sh
# mysql -uroot -proot
.. 생략 ..

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from exam;
+----+-------+---------------------+
| id | value | datetime            |
+----+-------+---------------------+
|  1 | a     | 2020-08-01 09:34:28 |
|  2 | b     | 2020-08-01 09:34:28 |
|  3 | c     | 2020-08-01 09:34:28 |
|  4 | new1  | 2020-08-01 09:39:37 |
|  5 | new2  | 2020-08-01 09:48:37 |
+----+-------+---------------------+
5 rows in set (0.00 sec)
```  

`Docker Compose` 템플릿과 스크립트를 이용해서 `Replication` 구성을 어느정도 자동화를 통해 확장성이 있는 구조로 만들어 보았다. 
하지만 `Slave` 확장에 있어서 아직 불편하고 추가 설정이나, 스크립트를 수정해야 하는 부분들이 존재한다. 
특정 테이블만 `Replication` 을 해야 할 경우, 설정 파일과 스크립트를 모두 수정해야 한다는 등의 문제점과 
안정적인 실제 서비스 중이라면 `Replication` 에 대한 안정성에 대한 고려가 필요하다. 
그리고 `Slave` 로드 밸런싱하는 구조, 설정에 대한 고려도 필요해 보인다.  

---
## Reference
[Create a MySQL slave from another slave, but point it at the master](https://serverfault.com/questions/257394/create-a-mysql-slave-from-another-slave-but-point-it-at-the-master/257426#257426)  
[MySQL Replication without stopping master](https://dba.stackexchange.com/questions/35977/mysql-replication-without-stopping-master)  
[mysqldump 옵션중 –dump-slave 활용](https://idchowto.com/?p=15391)  
[복제 필터링 - 명령문 및 적용방법](https://myinfrabox.tistory.com/27)  
