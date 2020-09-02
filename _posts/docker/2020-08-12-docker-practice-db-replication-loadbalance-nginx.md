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


## Nginx
먼저 `Nginx` 를 사용해서 `Slave` 를 로드밸런생 해본다. 
`Nginx` 는 `Plus` 버전이 아니므로, `Healthcheck` 기능은 제외한다. 

![그림 1]({{site.baseurl}}/img/docker/practice_db_replication_loadbalance_nginx_1.png)


### 구성 소개
디렉토리 구조는 아래와 같다. 

```bash
lb-nginx
├── docker-compose.yaml
└── nginx
    └── nginx.conf
```  

핵심이라고 할 수 있는 `nginx.conf` 내용은 아래와 같다. 

```
events {
    worker_connections 1024;
}

stream {
    # master node
    upstream master {
        server master_master-db:3306;
    }

    # slave node
    upstream slave {
        server slave1_slave-db:3306;
        server slave2_slave-db:3306;
    }

    # log format
    log_format smtplog '$remote_addr $remote_port -> $server_port '
                    '[$time_local] $bytes_sent $bytes_received '
                    '$session_time ==> $upstream_addr';

    # master server listen
    server {
        listen 3306;
        proxy_pass master;
        proxy_connect_timeout 1s;
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log smtplog;
    }

    # slave server listen
    server {
        listen 3307;
        proxy_pass slave;
        proxy_connect_timeout 1s;
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log smtplog;
    }
}
```  

기존 `HTTP` 서버를 로드밸런싱 하는 것과 크게 다르지 않다. 
`upstream` 필드로 해당하는 노드의 이름과 호스트 및 포트를 알맞게 설정해주고, 
`server` 필드에 `Nginx` 에서 사용할 포트와 적절한 `upstream` 이름을 `proxy_pass` 에 설정해 주면 된다. 
설정파일이 바뀐다면 기존과 동일하게 `nginx -s reload` 명령으로 재시작 할 수 있다.  

다음으로 `Nginx` 서비스를 구성하는 템플릿 파일인 `docker-compose.yaml` 파일은 아래와 같다. 

```yaml
version: '3.7'

services:
  lb:
    image: nginx:latest
    ports:
      - 30006:3306
      - 30007:3307
    networks:
      - lb-net
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf

  netshoot:
    image: nicolaka/netshoot
    networks:
      - lb-net
    stdin_open: true
    tty: true

networks:
  lb-net:
    external: true
```  

- `.services.lb` : `Nginx` 의 서비스 이름은 `lb` 이고 관련 구성요소를 설정한다. 
- `.services.lb.ports[]` : 설정파일과 매칭 시키면 호스트의 `30006` 포트는 `master` 와 연결되고, `30007` 포트는 `slave` 와 연결된다. 
- `.services.lb.networks[]` : 앞서 생성한 `lb-net` 을 네트워크로 설정해 준다. 
- `.services.lb.volumes[]` : 앞서 설명한 `nginx.conf` 를 컨테이너로 마운트 시킨다. 
- `.services.netshoot` : 네트워크 테스트용으로 사용할 서비스이다. 
- `.networks.lb-net` : 외부에서 생성한 네트워크를 템플릿에서 사용할 수 있도록 설정해 준다. 없어도 무관하다.

### 실행
이제 템플릿을 `Docker Swarm` 클러스터에 적용한다. 

```bash
$ docker stack deploy -c docker-compose.yaml nginx
Creating service nginx_netshoot
Creating service nginx_lb
$ docker service ls -f name=nginx
ID                  NAME                MODE                REPLICAS            IMAGE                      PORTS
vcn8rpdxr4c5        nginx_lb            replicated          1/1                 nginx:latest               *:30006->3306/tcp
m3cqg03tzqns        nginx_netshoot      replicated          1/1                 nicolaka/netshoot:latest
```  

`nginx` 라는 스택에 `Nginx` 가 구동되는 `lb` 서비스가 등록 되었고, 이름은 `nginx_lb` 인 것을 확인 할 수 있다.  

테스트는 `master` 서비스의 컨테이너에 접속한다음 `mysql -h<호스트> -P<포트> -u<계정명> -p<비밀번호> -e "show variables like 'server_id'` 명령을 사용해서 진행한다. 
`master` 서비스는 `lb-net` 네트워크에 참여하고 있기 때문에 호스트를 `nginx_lb` 로 하고 포트가 `3306` 일 경우엔 `master` 에 접속되어서 `1` 이라는 값이 출력되고, 
`3307` 포트로 접속하면 `3` 또는 `4` 가 반복해서 나오게 된다. 
먼저 아래 명령어로 `master` 서비스의 컨테이너에 접속한다. 

```bash
$ docker exec -it `docker ps -q -f name=master` /bin/bash
root@4553d02dc9ba:/#
```  

그리고 `3306` 포트로 명령을 몇번 수행해보면 아래와 같이 `server_id` 값으로 `1` 이 출력되는 것을 확인 할 수 있다. 

```bash
root@4553d02dc9ba:/# mysql -hnginx_lb -P3306 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3306 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```  

이번에는 `3307` 포트로 명령을 몇번 수행해 보면 아래와 같이 `server_id` 값으로 `3` 과 `4` 가 번갈아 출력되는 것을 확인 할 수 있다. 

```bash
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
```  

컨테이너에서 빠져나와 `nginx_lb` 서비스의 로그를 확인하면 실제로 연결된 로그를 확인 할 수 있다. 

```bash
docker service logs --tail 6 nginx_lb
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 35598 -> 3306 [14/Aug/2020:15:38:47 +0000] 3200 829 0.007 ==> 10.0.1.5:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 35614 -> 3306 [14/Aug/2020:15:38:49 +0000] 3200 829 0.007 ==> 10.0.1.5:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 44070 -> 3307 [14/Aug/2020:15:41:54 +0000] 3200 829 0.023 ==> 10.0.1.8:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 44084 -> 3307 [14/Aug/2020:15:41:55 +0000] 3200 829 0.013 ==> 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 44110 -> 3307 [14/Aug/2020:15:41:59 +0000] 3200 829 0.008 ==> 10.0.1.8:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 44118 -> 3307 [14/Aug/2020:15:41:59 +0000] 3200 829 0.007 ==> 10.0.1.10:3306
```  

### 테스트
이번엔 중간에 `slave` 중 하나의 서비스가 내려가게 된 상황을 2가지 상황으로 분류해서 진행해 본다. 
1. 컨테이너가 중지된 상황
1. 서비스가 중지된 상황

> 유지되는 커넥션(DBCP)와 관련된 테스트가 아닌, 단순이 커넥션이 `HAProxy` 를 통해 처리되는 상황을 보기 위함이다.

컨테이너가 중지된 상황의 재현은 `docker servcie scale <서비스이름>=0` 으로 수행한다. 
`slave1` 의 스케일을 0으로 설정하고 다시 `server_id` 를 가져오는 명령을 수행하면 아래와 같다. 

```bash
$ docker service scale slave1_slave-db=0
slave1_slave-db scaled to 0
overall progress: 0 out of 0 tasks
verify: Service converged

.. master container ..
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
```  

`slave1` 에 해당하는 `server_id` 는 출력되지 않는 것을 확인 할 수 있다. 
`nginx_lb` 로그를 확인하면 아래와 같이 `slave1` 으로 한번 시도가 있었지만 타임아웃이 발생해 `slave2` 로 요청이 넘어 간것을 확인 할 수 있다. 

```bash
$ docker service logs --tail 4 nginx_lb
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 2020/08/14 15:58:25 [error] 28#28: *15 upstream timed out (110: Connection timed out) while connecting to upstream, client: 10.0.1.7, server: 0.0.0.0:3307, upstream: "10.0.1.8:3306", bytes from/to client:0/0, bytes from/to upstream:0/0
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 50896 -> 3307 [14/Aug/2020:15:58:25 +0000] 3200 829 1.009 ==> 10.0.1.8:3306, 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 50914 -> 3307 [14/Aug/2020:15:58:27 +0000] 3200 829 0.006 ==> 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 50924 -> 3307 [14/Aug/2020:15:58:28 +0000] 3200 829 0.006 ==> 10.0.1.10:3306
```  

`slave1` 서비스의 스케일을 다시 1로 설정해주고 `nginx_lb` 컨테이너에서 `nginx -s reload` 명령을 수행하고 다시 `server_id` 명령을 수행하면 아래와 같다. 

```bash
$ docker service scale slave1_slave-db=1
slave1_slave-db scaled to 1
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
$ docker exec `docker ps -q -f name=nginx_lb` nginx -s reload
2020/08/14 16:03:20 [notice] 29#29: signal process started

.. master container ..
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
.. end ..

$ docker service logs --tail 4 nginx_lb
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 53052 -> 3307 [14/Aug/2020:16:03:33 +0000] 3200 829 0.020 ==> 10.0.1.8:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 53062 -> 3307 [14/Aug/2020:16:03:34 +0000] 3200 829 0.009 ==> 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 53070 -> 3307 [14/Aug/2020:16:03:35 +0000] 3200 829 0.007 ==> 10.0.1.8:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 53082 -> 3307 [14/Aug/2020:16:03:37 +0000] 3200 829 0.007 ==> 10.0.1.10:3306
```  

`nginx -s reload` 를 하기 전까지는 계속해서 `slave2` 에 해당하는 `server_id` 만 출력 된다. 
그리고 해당 명령을 수행한 후에는 기존과 동일하게 번갈아가면서 출력 되는 것을 확인 할 수 있다.  

서비스가 중지된 상황은 `docker stack rm <서비스이름>` 으로 수행한다. 
테스트 과정은 위와 동일하기 때문에 별도의 설명 없이 결과만 기술한다. 

```bash
$ docker stack rm slave1
Removing service slave1_slave-db

.. master container ..
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
.. end..

$ docker service logs --tail 4 nginx_lb
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 2020/08/14 16:07:49 [error] 34#34: *36 upstream timed out (110: Connection timed out) while connecting to upstream, client: 10.0.1.7, server: 0.0.0.0:3307, upstream: "10.0.1.8:3306", bytes from/to client:0/0, bytes from/to upstream:0/0
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 54820 -> 3307 [14/Aug/2020:16:07:49 +0000] 3200 829 1.008 ==> 10.0.1.8:3306, 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 54840 -> 3307 [14/Aug/2020:16:07:51 +0000] 3200 829 0.006 ==> 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 54848 -> 3307 [14/Aug/2020:16:07:51 +0000] 3200 829 0.007 ==> 10.0.1.10:3306
$ docker stack deploy -c docker-compose-slave-1.yaml slave1
Creating service slave1_slave-db
$ docker exec `docker ps -q -f name=nginx_lb` nginx -s reload
2020/08/14 16:09:49 [notice] 35#35: signal process started

.. master container ..
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4553d02dc9ba:/# mysql -hnginx_lb -P3307 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
.. end ..

$ docker service logs --tail 4 nginx_lb
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 55742 -> 3307 [14/Aug/2020:16:09:59 +0000] 3200 829 0.026 ==> 10.0.1.21:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 55756 -> 3307 [14/Aug/2020:16:10:00 +0000] 3200 829 0.008 ==> 10.0.1.10:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 55760 -> 3307 [14/Aug/2020:16:10:01 +0000] 3200 829 0.006 ==> 10.0.1.21:3306
nginx_lb.1.ry8eezibwm9y@docker-desktop    | 10.0.1.7 55770 -> 3307 [14/Aug/2020:16:10:01 +0000] 3200 829 0.006 ==> 10.0.1.10:3306
```  

`slave1` 에 해당하는 `IP` 가 변경된거 이외에는 컨테이너가 중지된 상황과 대부분 비슷한 결과를 보인다. 
해당 서비스를 지우고 다시 생성했기 때문에 서비스에 해당하는 `IP` 가 변경된다는 점을 주의 해야 한다.  


---
## Reference
[MySQL High Availability with NGINX Plus and Galera Cluster](https://www.nginx.com/blog/mysql-high-availability-with-nginx-plus-and-galera-cluster/)  