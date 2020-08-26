--- 
layout: single
classes: wide
title: "[Docker 실습] Database Loadbalancing(HAProxy)"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'HAProxy 를 사용해서 Database 를 Loadbalancing 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - HAProxy
  - MySQL
  - LoadBalancing
  - LoadBalancer
  - Replication
toc: true
use_math: true
---  

## HAProxy
[이전 글]({{site.baseurl}}{% link _posts/docker/2020-08-12-docker-practice-db-replication-loadbalance-nginx.md %})
에 이어 이번에는 `HAProxy` 를 사용해서 `Database Loadbalancing` 을 구성해 본다. 
이전 글에서 소개한 `lb-net` 네트워크와 `Master-Slave` 구조를 사용해서 이번 예제도 진행한다. 

![그림 1]({{site.baseurl}}/img/docker/practice_db_replication_loadbalance_haproxy_1.png)

### 구성 소개
디렉토리 구조는 아래와 같다. 

```
lb-haproxy/
├── docker-compose.yaml
└── haproxy
    └── haproxy.cfg
```  

`HAProxy` 는 앞서 살펴본 `Nginx` 와 달리 `HealthCheck` 기능을 제공한다. 
사용할 수 있는 다양한 방법의 헬스체크가 있지만 그중 `mysql-check` 와 `tcp-check` 를 사용해본다. 
- `mysql-check` : `L7` 레어에서 두개의 패킷을 사용해서 헬스체크를 수행한다. 
하나는 클라이언트 인증을 위한 패킷이고, 
다른 하나는 `MySQL` 세션을 정상적으로 종료하기 위한 `QUIT` 패킷이다. 
`HAProxy` 는 위 패킷을 파싱해서 헬스체크에 대한 판별을 수행한다. 
실제로 `MySQL` 이 패킷을 받을 수 있는 상황에서 헬스체크가 성공하기 때문에, 서버는 정상동작하지만 `MySQL` 이 정상동작하지 않는 경우를 판별해볼 수 있다. 
하지만 해당 헬스체크 기능을 사용하기 위해서는 별도로 `MySQL` 유저를 생성해주어야 한다. 

- `tcp-check` : `L4` 레이어에서 호스트와 포트가 통신가능한 상태인지 확인한다. 
하지만 이는 `MySQL` 이 정상적으로 동작하고 있는지에 대한 판단은 되지 않는다. 
`tcp-check` 를 사용하고 추가적인 스크립트와 구성으로 `MySQL` 의 정상동작 판별까지 수행할 수 있는 방법이 있긴하다. 

`HAproxy` 을 구성하는 설정 파일인 `haproxy.cfg` 의 내용은 아래와 같다.  

```
global
    log 127.0.0.1 local0 notice
    user root
    group root

## resolve docker dynamic dns
resolvers docker
    nameserver dns1 127.0.0.11:53
    resolve_retries 2
    timeout resolve 1s
    timeout retry   1s
    hold other      5s
    hold refused    5s
    hold nx         5s
    hold timeout    5s
    hold valid      5s
    hold obsolete   5s

defaults
    log global
    retries 2
    option tcplog
    option dontlognull
    timeout connect 3000
    timeout server 5000
    timeout client 5000

log stdout format raw daemon debug

## use tcp-check
listen master-tcp-node
    bind 0.0.0.0:3316
    mode tcp
    option tcp-check
    balance roundrobin
    server master master_master-db:3306 check resolvers docker init-addr libc,none inter 2s fall 2 rise 2

listen slave-tcp-node
    bind 0.0.0.0:3317
    mode tcp
    option tcp-check
    balance roundrobin
    timeout connect 1000
    default-server port 3306 resolvers docker init-addr libc,none inter 2s fall 2 rise 2
    server slave1 slave1_slave-db:3306 check
    server slave2 slave2_slave-db:3306 check


# use mysql-check
listen master-mysql-node
    bind 0.0.0.0:3326
    mode tcp
    option mysql-check user haproxy_user
    balance roundrobin
    server master master_master-db:3306 check resolvers docker init-addr libc,none inter 2s fall 2 rise 2

listen slave-mysql-node
    bind 0.0.0.0:3327
    mode tcp
    option mysql-check user haproxy_user
    balance roundrobin
    timeout connect 1000
    default-server resolvers docker init-addr libc,none inter 2s fall 2 rise 2
    server slave1 slave1_slave-db:3306 check
    server slave2 slave2_slave-db:3306 check

listen stats
    bind 0.0.0.0:8811
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth admin:admin
```  

`tcp-check` 는 `L4` 에서 동작하고, `myslq-check` 은 `L7` 에서 동작하는 헬스체크 기능이다. 
리슨 노드는 4개로 아래와 같이 구성돼있다.  

노드 이름|헬스체크|리슨 포트|포워딩 주소
---|---|---|---
master-tcp-node|tcp-check|3316|master_master-db:3306
slave-tcp-node|tcp-check|3317|slave1_slave-db:3306<br>slave2_slave-db:3306
master-mysql-node|mysql-check|3326|master_master-db:3306
slave-mysql-node|mysql-check|3327|slave1_slave-db:3306<br>slave2_slave-db:3306

`tcp-check` 를 사용하는 노드와 `mysql-check` 를 사용하는 두 가지 종류로 구성했고, 포워딩 주소는 역할에 따라 동일하다.  
`option` 프로퍼티를 확인하면 `tcp-check`, `mysql-check` 이 설정된 것을 확인할 수 있다. 
`tcp-check` 에는 별도로 설정할 추가 값이 없지만, `mysql-check` 에는 클라이언트 인증시에 사용할 계정 정보를 설정해 줘야한다. 
현재 유저이름은 `haproxy_user` 를 설정하고 비밀번호는 설정하지 않은 상태이다. 
[이전 글]({{site.baseurl}}{% link _posts/docker/2020-08-12-docker-practice-db-replication-loadbalance-nginx.md %})
에서 `Master` 데이터베이스의 `init.sql` 에 아래 비밀번호는 설정하지 않은 유저를 생성하는 쿼리를 추가해 준다. 
테스트로 목적으로 구성하기 때문에 계정에 대한 비밀번호를 설정하지 않았고, 많약 실환경을 목적으로 한다면 필수로 구성해야 한다.  

```sql
create user 'haproxy_user'@'%';
```  

`Master` 데이터베이스에만 유저를 추가하는 이유는 현재 `Master-Slave` 구성이 전체 데이터베이스를 덤프해서 적용하는 방식이기 때문이다.  

`server` 프로퍼티에 `호스트:포트` 를 작성하고 뒤에 `check` 를 명시해서 헬스체크를 활성화 했다. 
모든 노드에 포함되는 서버의 헬스체크 설정은 아래와 같다.
- `inter` : 헬스체크의 주기를 설정한다. 2초
- `fall` : 헬스체크가 몇번 실패하면 서버의 상태를 `down` 으로 할지 설정한다. 2번
- `rise` : 헬스체크가 몇번 성공하면 서버의 상태를 `up` 으로 할지 설정한다. 2번

설정 파일의 상단을 보면 `resolvers docker` 라는 설정이있다. 
이 설정은 `Docker Network` 에 구성된 서비스의 `DNS` 관련 이슈를 해결하기 위한 설정이다. 
도커에서 서비스가 클러스터에 등록되면 해당 서비스의 아이피 주소가 변경될 수 있다. 
하지만 `HAProxy` 는 아이피가 변경되면 서비스는 정상적으로 올라왔지만 서비스의 아이피를 찾지못한다. 
이를 해결할 수 있는 방법은 호스트의 `hosts` 파일을 컨테이너에 마운트하는 방법이 있지만 `Docker Swarm` 과 같은 클러스터 구성에서는 적합하지 않을 수 있다. 
그래서 호스트에서 실행되는 도커 `DNS` 주소를 명시해 줌으로써 해결하는 방법을 사용했다.  

모든 노드에 서버를 확인하면 `resolver docker` 를 등록해 서버의 주소를 찾을 때 명시한 `DNS` 주소를 사용하도록 한다.  

`listen stats` 은 `HAProxy` 의 상태를 노드와 서버별로 확인할 수 있는 페이지이다. 
`stats auth <계정명>:<비밀번호>` 로 페이지에 접속할 수 있는 계정을 설정한다. 
이후 서비스 실행 후 웹브라우저를 사용해서 `localhost:8811` 주소로 접속하면 아래와 같은 페이지를 확인할 수 있다.  

![그림 1]({{site.baseurl}}/img/docker/practice-replication-loadbalance-haproxy-status.png)



아래는 `HAProxy` 서비스의 템플릿 파일인 `docker-compose.yaml` 내용이다. 

```yaml
version: '3.7'

services:
  lb:
    image: haproxy:latest
    ports:
    - 30016:3316
    - 30017:3317
    - 30026:3326
    - 30027:3327
    - 8811:8811
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    environment:
      BALANCE: leastconn
    networks:
      - lb-net

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

- `.services.lb` : `HAProxy` 의 서비스 이름은 `lb` 이고 관련 구성요소를 설정한다. 
- `.services.lb.ports[]` : 설정파일과 매칭 시키면 호스트의 `30016` 포트는 `tcp-check` 를 사용하는 `master` 와 연결되고, `30017` 포트는 `tcp-check` 를 사용하는 `slave` 와 연결된다. 
그리고 `30026` 포트는 `mysql-check` 를 사용하는 `master` 와 연결되고, `30027` 포트는 `mysql-check` 를 사용하는 `slave` 와 연결된다. 
- `.services.lb.networks[]` : 앞서 생성한 `lb-net` 을 네트워크로 설정해 준다. 
- `.services.lb.volumes[]` : 앞서 설명한 `haproxy.cnf` 를 컨테이너로 마운트 시킨다. 
- `.services.netshoot` : 네트워크 테스트용으로 사용할 서비스이다. 없어도 무관하다.
- `.networks.lb-net` : 외부에서 생성한 네트워크를 템플릿에서 사용할 수 있도록 설정해 준다.  

### 실행
구성한 템플릿을 `Docker Swarm` 에 적용한다. 

```bash
$ docker stack deploy -c docker-compose.yaml haproxy
Creating service haproxy_lb
Creating service haproxy_netshoot
$ docker service ls -f name=haproxy
ID                  NAME                MODE                REPLICAS            IMAGE                      PORTS
ok04joakjvap        haproxy_lb          replicated          1/1                 haproxy:latest             *:8811->8811/tcp, *:30016-30017->3316-3317/tcp, *:30026-30027->3326-3327/tcp
f2gs7ryle0vm        haproxy_netshoot    replicated          1/1                 nicolaka/netshoot:latest
```  

테스트를 위해 `master` 컨테이너에 `docker exec -it $(docker ps -q -f name=<서비스이름혹은컨테이너이름>) /bin/bash` 

```bash
docker exec -it `docker ps -q -f name=master` /bin/bash
root@21ab9ea103a8:/#
```  

`HAProxy` 에 설정된 `3316`, `3326` 로 명령을 수행하면 `master` 의 `server_id` 가 출력되는 것을 확인 할 수 있다.

```bash
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3316 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3316 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3326 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3326 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```  

그리고 이어서 `3317`, `3327` 로 명령을 수행하면 `slave` 의 `server_id` 인 3, 4가 출력되는 것을 확인 할 수 있다.

```bash
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@21ab9ea103a8:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
```  

지금까지 요청을 `HAProxy` 로그로 확인하면 아래와 같다.

```bash
docker service logs --tail 20 haproxy_lb
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:37336 [25/Aug/2020:18:24:02.387] master-tcp-node master-tcp-node/master 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:37348 [25/Aug/2020:18:24:03.254] master-tcp-node master-tcp-node/master 1/0/4 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:49680 [25/Aug/2020:18:24:07.140] master-mysql-node master-mysql-node/master 1/0/4 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:49708 [25/Aug/2020:18:24:08.968] master-mysql-node master-mysql-node/master 1/0/4 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:48542 [25/Aug/2020:18:25:38.871] slave-tcp-node slave-tcp-node/slave1 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:48562 [25/Aug/2020:18:25:39.995] slave-tcp-node slave-tcp-node/slave2 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:48578 [25/Aug/2020:18:25:41.483] slave-tcp-node slave-tcp-node/slave1 1/0/4 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:48594 [25/Aug/2020:18:25:42.461] slave-tcp-node slave-tcp-node/slave2 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:53592 [25/Aug/2020:18:25:46.571] slave-mysql-node slave-mysql-node/slave1 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:53608 [25/Aug/2020:18:25:47.667] slave-mysql-node slave-mysql-node/slave2 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:53622 [25/Aug/2020:18:25:48.557] slave-mysql-node slave-mysql-node/slave1 1/0/5 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.9xv3jpwuhk3k@docker-desktop    | 10.0.1.8:53634 [25/Aug/2020:18:25:49.331] slave-mysql-node slave-mysql-node/slave2 1/0/4 3200 -- 1/1/0/0/0 0/0
```  

### 테스트
`Nginx` 와 동일하게 아래 2가지 상황에 대해서 `slave` 가 다운된 상황을 테스트 해본다. 
1. 컨테이너가 중지된 상황
1. 서비스가 중지된 상황

컨테이너 중지된 상황과 서비스가 중지된 상황의 차이점은 컨테이너만 재시작 되는 경우 아피는 변경되지 않지만, 
서비스가 재시작 되면 `HAProxy` 에서 로드밸런싱으로 등록한 노드 아이피가 변경된다.  

> 유지되는 커넥션(DBCP)와 관련된 테스트가 아닌, 단순이 커넥션이 `HAProxy` 를 통해 처리되는 상황을 보기 위함이다. 

그리고 `HAProxy` 는 헬스체크 기능이 있기 때문에 `status` 페이지(`localhost:8811`) 에 접속해서 함께 확인해본다. 
먼저 컨테이너가 중지된 상황은 서비스의 스케일을 0으로 만드는 방법으로 진행한다. 
아래 명령어로 `slave1` 서비스의 스케일을 0으로 수정하고 상태페이지 및 `master` 컨테이너에서 명령을 수행하면 아래와 같다. 

```bash
$ docker service scale slave1_slave-db=0

slave1_slave-db scaled to 0
overall progress: 0 out of 0 tasks
verify: Service converged
```  

![그림 1]({{site.baseurl}}/img/docker/practice-replication-loadbalance-haproxy-containerdown-1.png)  

![그림 1]({{site.baseurl}}/img/docker/practice-replication-loadbalance-haproxy-containerdown-2.png)  

우선 헬스체크가 실패하면 `down` 상태로 됐다가, `maint` 상태로 전환된다. 

```bash
docker exec -it `docker ps -q -f name=master` /bin/bash
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
```  

`master` 컨테이너에 접속해서 명령을 수행하면, `slave2` 서비스에 해당하는 `server_id` 인 4반 출력하는 것을 확인 할 수 있다.  

컨테이너가 종료된 시점부터 명령 수행까지 로그를 확인하면 아래와 같다. 
`L4` 헬스체크에서 타입아웃이 발생한 이후에 `DOWN` 상태로 되고, 그 이후에 `MAINT` 상태로 전환된 것을 확인 할 수 있다. 
그리고 이후에는 `slave2` 서비스로만 트래픽이 전달된다. 

```
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/062605 (6) : Server slave-tcp-node/slave1 is DOWN, reason: Layer4 timeout, info: " at initial connection step of tcp-check", check duration: 2000ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 is DOWN, reason: Layer4 timeout, info: " at initial connection step of tcp-check", check duration: 2000ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/062606 (6) : Server slave-mysql-node/slave1 is DOWN, reason: Layer4 timeout, check duration: 2001ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 is DOWN, reason: Layer4 timeout, check duration: 2001ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/062612 (6) : Server slave-tcp-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/062612 (6) : Server slave-mysql-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:54756 [26/Aug/2020:12:37:07.405] slave-tcp-node slave-tcp-node/slave2 1/0/12 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:54766 [26/Aug/2020:12:37:08.686] slave-tcp-node slave-tcp-node/slave2 1/0/6 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:37686 [26/Aug/2020:12:37:15.297] slave-mysql-node slave-mysql-node/slave2 1/0/8 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:37694 [26/Aug/2020:12:37:15.930] slave-mysql-node slave-mysql-node/slave2 1/0/8 3200 -- 1/1/0/0/0 0/0
```  

다시 `slave1` 서비스의 스케일을 1로 설정한다. 
`HAProxy` 는 현재 헬스체크를 수행 중이기 때문에, 
`Nginx` 처럼 서비스가 다시 올라왔을 때 로드밸런서를 재시작하는 등의 수행 없이 헬스체크가 성공하면 자동으로 등록된다. 
등록된 이후 `master` 컨테이너에서 명령을 수행하면 `slave1` 서비스에도 트래픽이 전달되는 것을 확인할 수 있다. 

```bash
$ docker service scale slave1_slave-db=1
slave1_slave-db scaled to 1
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
```  


![그림 1]({{site.baseurl}}/img/docker/practice-replication-loadbalance-haproxy-containerup-1.png)  

![그림 1]({{site.baseurl}}/img/docker/practice-replication-loadbalance-haproxy-containerup-2.png)  

`master` 컨테이너에서 명령을 수행하면 `slave1`, `slave2` 서비스에 해당하는 `server_id` 가 모두 조회되는 것을 확인 할 수 있다. 

```bash
$ docker exec -it `docker ps -q -f name=master` /bin/bash
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
```  

마지막으로 로그를 확인하면 `MAINT` 상태에서 `DOWN` 상태로 전환되고, 
헬스체크가 모두 성공하면 다시 `UP` 상태로 변경되면서 로드밸런싱 대상이 되는 것을 확인 할 수 있다. 

```bash
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/065318 (6) : Server slave-mysql-node/slave1 is UP, reason: Layer7 check passed, code: 0, check duration: 8ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 is UP, reason: Layer7 check passed, code: 0, check duration: 8ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/065320 (6) : Server slave-tcp-node/slave1 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:37900 [26/Aug/2020:12:54:09.530] slave-tcp-node slave-tcp-node/slave1 1/0/9 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:37914 [26/Aug/2020:12:54:10.427] slave-tcp-node slave-tcp-node/slave2 1/0/9 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:49052 [26/Aug/2020:12:54:15.398] slave-mysql-node slave-mysql-node/slave1 1/0/7 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:49074 [26/Aug/2020:12:54:16.489] slave-mysql-node slave-mysql-node/slave2 1/0/5 3200 -- 1/1/0/0/0 0/0
```  

서비스가 재시작되는 상황을 `docker stack rm <서비스이름>` 명령을 사용해서 재연해본다. 
테스트는 컨테이너가 재시작 되는 과정과 동일하다. 

```bash
$ docker stack rm slave1
Removing service slave1_slave-db

$ docker exec -it `docker ps -q -f name=master` /bin/bash
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# exit
exit
$ docker service logs --tail 12 haproxy_lb
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 1049ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070538 (6) : Server slave-mysql-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 1049ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070540 (6) : Server slave-tcp-node/slave1 is DOWN, reason: Layer4 timeout, info: " at initial connection step of tcp-check", check duration: 2000ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 is DOWN, reason: Layer4 timeout, info: " at initial connection step of tcp-check", check duration: 2000ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070547 (6) : Server slave-tcp-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070547 (6) : Server slave-mysql-node/slave1 was DOWN and now enters maintenance (DNS NX status).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:47018 [26/Aug/2020:13:06:01.986] slave-tcp-node slave-tcp-node/slave2 1/0/6 3200 -- 3/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:47032 [26/Aug/2020:13:06:02.957] slave-tcp-node slave-tcp-node/slave2 1/0/6 3200 -- 3/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:58154 [26/Aug/2020:13:06:07.329] slave-mysql-node slave-mysql-node/slave2 1/0/7 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:58168 [26/Aug/2020:13:06:08.203] slave-mysql-node slave-mysql-node/slave2 1/0/7 3200 -- 1/1/0/0/0 0/0
$ docker stack deploy -c docker-compose-
slave-1.yaml slave1
Creating service slave1_slave-db
$ docker exec -it `docker ps -q -f name=master` /bin/bash
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3317 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+
root@4ac4613494d7:/# mysql -hhaproxy_lb -P3327 -uroot -proot -e "show variables like 'server_id'"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 4     |
+---------------+-------+
root@4ac4613494d7:/# exit
exit
$ docker service logs --tail 24 haproxy_lb
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : slave-tcp-node/slave1 changed its IP from 10.0.7.64 to 10.0.7.120 by docker/dns1.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : Server slave-tcp-node/slave1 ('slave1_slave-db') is UP/READY (resolves again).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | slave-tcp-node/slave1 changed its IP from 10.0.7.64 to 10.0.7.120 by docker/dns1.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 ('slave1_slave-db') is UP/READY (resolves again).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : Server slave-tcp-node/slave1 administratively READY thanks to valid DNS answer.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 administratively READY thanks to valid DNS answer.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | slave-mysql-node/slave1 changed its IP from 10.0.7.64 to 10.0.7.120 by DNS cache.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : slave-mysql-node/slave1 changed its IP from 10.0.7.64 to 10.0.7.120 by DNS cache.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : Server slave-mysql-node/slave1 ('slave1_slave-db') is UP/READY (resolves again).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 ('slave1_slave-db') is UP/READY (resolves again).
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 administratively READY thanks to valid DNS answer.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : Server slave-mysql-node/slave1 administratively READY thanks to valid DNS answer.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070838 (6) : Server slave-tcp-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused at initial connection step of tcp-check", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused at initial connection step of tcp-check", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070840 (6) : Server slave-mysql-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070902 (6) : Server slave-mysql-node/slave1 is UP, reason: Layer7 check passed, code: 0, check duration: 14ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-mysql-node/slave1 is UP, reason: Layer7 check passed, code: 0, check duration: 14ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | [WARNING] 238/070902 (6) : Server slave-tcp-node/slave1 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | Server slave-tcp-node/slave1 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:49184 [26/Aug/2020:13:09:12.249] slave-tcp-node slave-tcp-node/slave1 1/0/8 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:49200 [26/Aug/2020:13:09:12.943] slave-tcp-node slave-tcp-node/slave2 1/0/7 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:60338 [26/Aug/2020:13:09:18.007] slave-mysql-node slave-mysql-node/slave1 1/0/7 3200 -- 1/1/0/0/0 0/0
haproxy_lb.1.j0v5awcu7s9q@docker-desktop    | 10.0.7.14:60356 [26/Aug/2020:13:09:18.818] slave-mysql-node slave-mysql-node/slave2 1/0/6 3200 -- 1/1/0/0/0 0/0
```  

서비스가 다시 올라오고 난후 `haproxy_lb` 로그 부분만 좀 더 확인해본다. 
로그를 보면 `HAProxy` 에 등록된 노드의 아이피가 변경된 것을 감지하고 설정된 `resolver` 의 `docker/dns1` 에서 새로운 아이피를 받아와 등록하는 절차를 먼저 수행한다. 
실제 로그를 확인하면 원래 해당 노드에 등록된 아이피는 `10.0.7.64` 였는데 서비스가 재시작 되고 난후 `10.0.7.120` 으로 변경 되었다. 
`DNS resolver` 관련 동작이 완료되면 `slave-tcp-node` 는 `L4` 레이어에서 헬스체크를 수행하고, 
`slave-mysql-node` 는 `L4` 레이어 헬스체크가 성공하면 `L7` 레이어에서 헬스체크를 수행한 후 모두 성공하면 상태를 `UP` 으로 변경하고 로드밸런싱 대상으로 등록된다.  


만약 실행 중인 `HAProxy` 의 설정파일을 변경 후 컨테이너 재시작 없이 적용하고 싶다면 아래 명령으로 가능하다. 

```bash
$ docker kill -s HUP <컨테이너아이디 혹은 이름>

$ docker kill -s HUP `docker ps -q -f name=<컨테이너를 식별가능한 이름 혹은 문자열>`
```  


---
## Reference
[Performing Health Checks](https://www.haproxy.com/de/documentation/aloha/12-0/traffic-management/lb-layer7/health-checks/)  
[Docker II: Replicated MySQL with HAProxy Load Balancing](http://blog.ditullio.fr/2016/07/07/docker-replicated-mysql-haproxy-load-balancing/)  
[Docker II: HAProxy – Mysql cluster on Docker](https://vnextcoder.wordpress.com/2016/09/22/haproxy-mysql-cluster-on-docker/)  
[MySQL Load Balancing with HAProxy](https://severalnines.com/resources/database-management-tutorials/mysql-load-balancing-haproxy-tutorial)  
[Dynamic DNS Resolution with HAProxy and Docker](https://stackoverflow.com/questions/41152408/dynamic-dns-resolution-with-haproxy-and-docker)  
[HAProxy and container IP changes in Docker](https://www.haproxy.com/blog/haproxy-and-container-ip-changes-in-docker/)  
