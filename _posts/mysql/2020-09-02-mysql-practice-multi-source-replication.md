--- 
layout: single
classes: wide
title: "[MySQL 개념] Multi Source Replication"
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
먼저 `Master` 1개 `Slave` 1개를 사용해서 초기상태부터 구성하는 예제를 진행한다. 

`master-1` 서비스를 구성하는 템플릿 파일인 `docker-compose-master-1.yaml` 의 내용은 아래와 같다. 

```yaml
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
    ports:
      - 33000:3306

networks:
  multi-source-net:
    external: true
```  

`master-1` 서비스에서 실행되는 `MySQL` 설정 파일인 `master-1.cnf` 파일 내용은 아래와 같다. 

```
[mysql]
server-id=1
default-authentication-plugin=mysql_native_password
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=on
```  

`row-based replication` 을 사용하고, `gtid` 을 활성화 하는 설정으로 구성돼있다. 
`docker stack deploy -c docker-compose-master-1.yaml master-1` 명령으로 서비스를 클러스터에 적용하고, 
`MySQL` 정상 동작 여부를 확인하면 아래와 같다. 


master-1.cnf 권한 변경하고 실행하고 slave 템플릿 및 설정 설명 하고 클러스터 적용하고 연동해서 결과 
```bash
docker stack deploy -c docker-compose-master-1.yaml master-1
Creating service master-1_db
docker exec -it `docker ps -q -f name=master-1` mysql -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] World-writable config file '/etc/mysql/conf.d/custom.cnf' is ignored.
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```


### 다른 데이터베이스를 사용하는 Master 추가하기


### 같은 데이터베이스를 사용하는 Master 추가하기

### 같은 데이터베이스, 같은 테이블을 사용하는 Master 추가하기


---
## Reference
[17.1.5 MySQL Multi-Source Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-multi-source.html)  
[MySQL 5.7 Multi-Source Replication – Automatically Combining Data From Multiple Databases Into One](http://mysqlhighavailability.com/mysql-5-7-multi-source-replication-automatically-combining-data-from-multiple-databases-into-one/)  
[MySQL 5.7 multi-source replication](https://www.percona.com/blog/2013/10/02/mysql-5-7-multi-source-replication/)  
[MySQL Multi-Source Replication 기본 사용법](https://mysqldba.tistory.com/155)  
[Mysql 8 multi-source-replication 개선점](https://sarc.io/index.php/mariadb/1782-mysql-8-multi-source-replication-2)  
[MySQL 8.0 - Replication By Channel](http://minsql.com/mysql8/B-4-A-replicationByChannel/)  
[GTID(Global Transaction Identifiers)](https://minsugamin.tistory.com/entry/MySQL-57-GTID)  
