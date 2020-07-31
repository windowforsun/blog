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

디렉토리 구조는 아래와 같다. 

```bash
.
├── docker-compose.yaml
├── master-1
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       └── _init.sql
├── master-2
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       └── _init.sql
├── slave-1
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       ├── _init.sql
│       └── replication.sh
└── slave-2
    ├── conf.d
    │   └── custom.cnf
    └── init
        ├── _init.sql
        └── replication.sh
```  

## Docker Compose 구성















































































## Docker 기반 Database 구성
테스트를 위해 `Database` 구성은 `Docker` 를 사용해서 구성한다. 
`Replication` 구조로 되어있고, `Replication` 관련 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %}) 
에서 확인 할 수 있다.  



전체적인 컨테이너 템플릿을 정의하는 `docker-compose.yaml` 파일내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  master-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master-1/conf.d/:/etc/mysql/conf.d
      - ./master-1/init/:/docker-entrypoint-initdb.d/
    ports:
      - 33000:3306

  slave-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-1
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
      MASTER_LOG_BIN: mysql-bin
    networks:
      - db-net
    volumes:
      - ./slave-1/conf.d/:/etc/mysql/conf.d
      - ./slave-1/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34000:3306

  master-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master-2/conf.d/:/etc/mysql/conf.d
      - ./master-2/init/:/docker-entrypoint-initdb.d/
    ports:
      - 33001:3306

  slave-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-2
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
      MASTER_LOG_BIN: mysql-bin
    networks:
      - db-net
    volumes:
      - ./slave-2/conf.d/:/etc/mysql/conf.d
      - ./slave-2/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34001:3306

  adminer:
    image: adminer
    restart: always
    ports:
      - 8888:8080
    networks:
      - db-net

networks:
  db-net:
```  

- 총 4개의 `Databas` 로 구성돼있고, `Master-Slave` 한쌍씩 되어있다. 
- 실제로 구성한다면 `Database` 는 개별 호스트에 독립 컨테이너 형식이기 때문에 별도의 서비스로 만들었다. 
- `Master` 와 `Slave` 설정에 약간의 차이점이있고, 
`Master1`, `Master2` 와 `Slave1`, `Slave2` 는 포트나 참조하는 `Database` 의 값만 다르다. 
- `Master1-Slave1`, `Master2-Slave2` 의 구조로 구성돼 있다. 
- `Slave` 에서는 `Master` 관련 정보를 환경변수로 받아 설정한다. 

`Master` 의 `MySQL` 설정 파일인 `master-*/conf.d/custom.cnf` 파일의 내용은 아래와 같다. 

```
[mysqld]
server-id=1
log-bin=mysql-bin
default-authentication-plugin=mysql_native_password
```  

`Master` 의 초기화 `SQL` 파일은 `master-*/init/_init.sql` 파일은 아래와 같다.

```sql
create database test;

use test;

-- Master1 --
CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----

-- Master2 --
CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----


create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;
```  

- `Master1` 은 `exam` 테이블을 가지고, `Master2` 는 `exam2` 테이블을 갖는다. 

다음으로 `Slave` 의 `MySQL` 설정 파일인 `slave-*/conf.d/custom.cnf` 파일 내용은 아래와 같다. 

```
[mysqld]
server-id=2
default-authentication-plugin=mysql_native_password
```  

`Slave` 의 초기화 `SQL` 파일인 `slave-*/init/_init.sql` 파일 내용은 아래와 같다. 

```sql
create database test;

use test;

-- Slave1 --
CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----

-- Slave2 --
CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----
```  

- `Master` 와 동일하게 `Slave1` 은 `exam` 테이블을 `Slave2` 은 `exam2` 테이블을 갖는다. 

`Slave` 가 구동되면서 환경변수의 `Master` 관련 정보를 사용해서 `Replication` 을 구성하는 `slave-*/init/replication.sh` 파일 내용은 아래와 같다. 

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

master_log_file=''

# waiting master connection and get master log path
while [[ ${master_log_file} != *"${MASTER_LOG_BIN}"* ]]; do
  master_log_file=$(doMasterQuery "show master status\\G" "${MASTER_LOG_BIN}")
  sleep 3
done

# get master log position
master_log_pos=$(doMasterQuery "show master status\\G", "Position")

# parsing log file
re="${MASTER_LOG_BIN}.[0-9]*"
if [[ ${master_log_file} =~ $re ]]; then
  master_log_file=${BASH_REMATCH[0]}
fi

# parsing log position
re="[0-9]+"
if [[ ${master_log_pos} =~ $re ]]; then
  master_log_pos=${BASH_REMATCH[0]}
fi

# set master info
replication_query="change master to
master_host='${MASTER_HOST}',
master_user='${MASTER_REPL_USER}',
master_password='${MASTER_REPL_PASSWD}',
master_log_file='${master_log_file}',
master_log_pos=${master_log_pos}"
echo $(doSlaveQuery "${replication_query}")

echo $(doSlaveQuery "start slave")
```  

- `Master` 가 완전히 올라갈때 까지 대기한다.
- `show master status` 에서 현재 로그 파일 명을 파싱한다. 
- `show master status` 에서 현재 로그 포지션을 파싱한다. 
- `change master to` 명령으로 마스터 설정을 한다. 


---
## Reference
