--- 
layout: single
classes: wide
title: "[Docker 실습] Database Loadbalancing(Nginx)"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Nginx 를 사용해서 Database 를 Loadbalancing 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Nginx
  - MySQL
  - LoadBalancing
  - LoadBalancer
  - Replication
toc: true
use_math: true
---  

## DataBase Replication LoadBalancing
[DataBase Replication]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %})
을 하게 되면 부하 분산을 위해 `Slave` 개수를 늘리게 될 수 있다. 
이런 상황에서 애플리케이션은 여러개 `Slave` 커넥션을 관리해야 되고, 로드 밸런싱까지 수행해야 할 수 있다. 
하지만 애플리케이션에서 `Slave` 커넥션에 대한 로드밸런싱을 수행하게 될 경우, 
`Failover` 에 대한 처리가 돼있지 않다면 일부 요청이 처리 불가능한 상태가 될 수 있고 이는 서비스 불가 현상으로 확장 될 수 있다. 
그리고 `Slave` 가 추가되는 상황에서도 애플리케이션에 로드밸런싱 로직이 있다면 배포를 다시 수행해 줘야 한다.  

이런 상황에 대한 해결책으로 별도의 `LoadBalancer` 를 사용해서 해결해 보고자 한다. 
애플리케이션에서 구현한다면 예외 상황에 대한 처리를 보다 커스텀하게 구현할 수 있겠지만, 
이 또한 추가 구현사항이므로 기존 검증된 솔루션을 사용한 방법에 대해 기술한다.  

지금 부터 서술되는 내용은 실환경의 목적으로 구성을 했다기 보다 테스트 목적으로 구성했다는 점을 미리 알린다.  

## MySQL Replication
`Database` 는 `MySQL` 을 사용하고 자세한 `Replication` 구성은 [여기]({{site.baseurl}}{% link _posts/docker/2020-07-31-docker-practice-mysql-replication-template.md %})
에서 확인할 수 있다.  

전체 프로젝트 디렉토리 구조는 아래와 같다. 

```bash
.
├── db-replication
│   ├── docker-compose-master.yaml
│   ├── docker-compose-slave-1.yaml
│   ├── docker-compose-slave-2.yaml
│   ├── master
│   │   ├── conf.d
│   │   │   └── custom.cnf
│   │   └── init
│   │       └── _init.sql
│   └── slave
│       ├── conf.d
│       │   └── custom.cnf
│       └── init
│           └── replication.sh
├── lb-haproxy
|   .. 생략 ..
└── lb-nginx
    .. 생략 ..
```  

`db-replication` 디렉토리는 `Master` 서비스와 2개의 `Slave` 서비스로 구성 된다. 
먼저 `master` 서비스 템플릿인 `docker-compose-master.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  master-db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - lb-net
    volumes:
      - ./master/conf.d/:/etc/mysql/conf.d
      - ./master/init/:/docker-entrypoint-initdb.d/
    ports:
      - 33000:3306

networks:
  lb-net:
    external: true
```  

여기서 기억해야 할점은 `master` 서비스의 `MySQL` 의 `server_id` 는 `1` 이라는 점이다.  

`slave1` 템플릿 내용은 아래와 같다. 

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
    command: --server-id=3
    networks:
      - lb-net
    volumes:
      - ./slave/conf.d/:/etc/mysql/conf.d
      - ./slave/init/:/docker-entrypoint-initdb.d/

networks:
  lb-net:
    external: true
```  

여기서 기억해야 할점은 `slave1` 서비스의 `MySQL` 의 `server_id` 는 `3` 이라는 점이다.  

`slave1` 템플릿 내용은 아래와 같다. 

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
    command: --server-id=4
    networks:
      - lb-net
    volumes:
      - ./slave/conf.d/:/etc/mysql/conf.d
      - ./slave/init/:/docker-entrypoint-initdb.d/

networks:
  lb-net:
    external: true
```  

여기서 기억해야 할점은 `slave2` 서비스의 `MySQL` 의 `server_id` 는 `4` 이라는 점이다.  

위 템플릿과 추후의 설명에서 확인 할 수 있겠지만, 
테스트에서 `lb-net` 이라는 `overlay` 네트워크를 사용하기 때문에 서비스를 구성하기 전에 먼저 생성해 줘야 한다. 

```bash
$ docker network create --driver overlay lb-net
djs5a4x8d36wnnlpu3wtefuhb
```  

그리고 `master`, `slave` 순서로 `Docker swarm` 클러스터에 서비스를 실행 시킨다. 

```bash
$ docker stack deploy -c docker-compose-master.yaml master
Creating service master_master-db
$ docker stack deploy -c docker-compose-slave-1.yaml slave1
Creating service slave1_slave-db
$ docker stack deploy -c docker-compose-slave-2.yaml slave2
Creating service slave2_slave-db
$ docker stack ls
NAME                SERVICES            ORCHESTRATOR
master              1                   Swarm
slave1              1                   Swarm
slave2              1                   Swarm
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
263h9cgdfc4o        master_master-db    replicated          1/1                 mysql:8             *:33000->3306/tcp
rlanugnimnoh        slave1_slave-db     replicated          1/1                 mysql:8
okylfb9cft4o        slave2_slave-db     replicated          1/1                 mysql:8
```  

## HAProxy
[이전 글]()
에 이어 이번에는 `HAProxy` 를 사용해서 `Database Loadbalancing` 을 구성해 본다. 
디렉토리 구조는 아래와 같다. 

```

```  









































---
## Reference
[MySQL High Availability with NGINX Plus and Galera Cluster](https://www.nginx.com/blog/mysql-high-availability-with-nginx-plus-and-galera-cluster/)  