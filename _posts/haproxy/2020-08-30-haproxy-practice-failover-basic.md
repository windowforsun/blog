--- 
layout: single
classes: wide
title: "[HAProxy 실습] Failover 설정과 테스트"
header:
  overlay_image: /img/haproxy-bg-2.png
excerpt: 'HAProxy 에서 Failover 설정에 대해 알아보고 간단한 테스트로 실제 동작을 확인해보자'
author: "window_for_sun"
header-style: text
categories :
  - HAProxy
tags:
  - HAProxy
  - Practice
  - Failover
toc: true
use_math: true
---  

## Failover
`HAProxy` 를 로드밸런서로 사용할 때 다양한 조건과 방식을 사용해서 `failover` 처리를 할 수 있다. 
여러 케이스에 상황을 재연하고 적절한 `failover` 처리에 대해 알아본다. 
실습은 `Docker` 를 기반으로 구성한다. 
실습에서는 `HAProxy` 가 헬스체크를 수행하며 발생할 수 있는 딜레이, 트래픽 단절에 대한 부분은 다루지 않는다. 
그리고 `HAProxy` 에서 `failover` 처리에 설정을 하게되면 어떤 결과를 보이는지를 테스트하는 포스트이다. 

## 로드밸런싱 대상
로드밸런싱 대상으로는 `Nginx` 를 사용한다. 
`Docker` 를 사용해서 `Nginx` 컨테이너를 실행시킨다. 
`Nginx` 설정파일은 아래와 같은 `conf.template` 파일을 사용한다. 

```
# nginx.conf.template

server {
    listen ${NGINX_PORT};

    location / {
        default_type text/plain;
        return 200 ${STR};
    }
}
```  

`listen` 필드는 `NGINX_PORT` 환경 변수를 사용해서 리슨 포트를 설정할 수 있도록 했다. 
그리고 `return` 필드를 사용해서 `Nginx` 에서 바로 특정 문자열인 `STR` 환경변수에 설정된 값을 응답하도록 구성했다. 


## Backup server
로드밸런서에 등록된 모든 서버가 사용이 불가능하게 되면 `backup server` 를 통해 `failover` 처리를 할 수 있다. 

```
# backup-server.cfg

defaults
    timeout connect 200ms
    timeout client 1000ms
    timeout server 1000ms

frontend ft-app
    bind 0.0.0.0:81
    default_backend bk-app-main

backend bk-app-main
    server nginx1 nginx-1:81 check
    server nginx2 nginx-2:81 check
    server nginx3 nginx-3:81 check backup
    server nginx4 nginx-4:81 check backup


listen stats
    bind 0.0.0.0:8711
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth admin:admin
```  

`defaults` 설정 파일에서 공통으로 적용되는 기본 설정 내용이고, 
`listen stats` 은 `HAProxy` 에서 관리하는 서버들의 상태를 확인 할 수 있는 상태 페이지에 대한 설정이다.  

`ft-app` 이라는 프론트엔드는 81 포트를 리슨하며 들어오는 트래픽을 백엔드인 `bk-app-main` 으로 전달한다. 
그리고 `bk-app-main` 은 `nginx1, 2, 3` 은 로드밸런싱 대상이고, `nginx3, 4` 는 백업 서버로 설정된 상태이다. 
여기서 `nginx1, 2, 3` 이 모두 가용 불가능한 상태가 되면 `nginx3` 으로 트래픽이 전달되고, 
`nginx3` 까지 가용불가능 상태가되면 마지막으로 `nginx4` 로 트래픽이 전달된다.  

전체 서버를 구성하는 `docker-compose-backup-server.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  haproxy:
    image: haproxy:latest
    volumes:
      - ./backup-server.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - 8181:81
      - 8711:8711
    networks:
      - ha-net

  nginx-1:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-1"
    networks:
      - ha-net
  nginx-2:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-2"
    networks:
      - ha-net

  nginx-3:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-3"
    networks:
      - ha-net

  nginx-4:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-4"
    networks:
      - ha-net

networks:
  ha-net:
```  

`haproxy` 서비스는 `8181` 포트로 요청하면 `nginx` 의 응답을 받을 수 있고, 
`8711` 포트로 요청하면 현재 서비스의 상태 페이지에 접속할 수 있다. 
`nginx1 ~ 4` 서비스는 81 포트를 리슨 포트로 설정했고, 응답하면 문자열은 서비스 이름에 맞춰 `nginx1 ~ 4` 이다.  

`docker-compose -f docker-compose-backup-server.yaml up -d` 명령을 사용해서 각 서비스를 실행시키고, 
`curl localhost:8181` 로 몇번의 요청을 수행하면 아래와 같이 `nginx1 ~ 2` 이 로드밸런싱 대상인 것을 확인할 수 있다. 

```bash
$ docker-compose -f docker-compose-backup-server.yaml up -d
Creating network "failover_ha-net" with the default driver
Creating failover_haproxy_1 ... done
Creating failover_nginx-2_1 ... done
Creating failover_nginx-3_1 ... done
Creating failover_nginx-4_1 ... done
Creating failover_nginx-1_1 ... done
$ curl localhost:8181
nginx-1
$ curl localhost:8181
nginx-2
$ curl localhost:8181
nginx-1
$ curl localhost:8181
nginx-2
```  

`docker-compose -f docker-compose-backup-server.yaml stop <서비스이름>` 으로 먼저 `nginx-1` 을 중지시키면 `nginx-2` 에만 트래픽이 전달된다. 
그리고 `nginx-2` 까지 중지시키면 `nginx-3` 에 트래픽이 전달되다가, `nginx-3` 까지 중지시키면 `nginx-4` 로 트래픽이 전달되는 것을 확인 할 수 있다. 

```bash
$ docker-compose -f docker-compose-backup-server.yaml stop nginx-1
Stopping failover_nginx-1_1 ... done
$ curl localhost:8181
nginx-2
$ curl localhost:8181
nginx-2
$ docker-compose -f docker-compose-backup-server.yaml stop nginx-2
Stopping failover_nginx-2_1 ... done
$ curl localhost:8181
nginx-3
$ curl localhost:8181
nginx-3
$ docker-compose -f docker-compose-backup-server.yaml stop nginx-3
Stopping failover_nginx-3_1 ... done
$ curl localhost:8181
nginx-4
$ curl localhost:8181
nginx-4
```  

## Multiple backup servers
먼저 살펴본 예제에서는 `backup server` 가 한개씩 적용됐다. 
이번에는 로드밸런싱 대상인 서버가 모두 가용할 수 없을 때, `backup server` 를 로드밸런싱 하는 설정에 대해 알아본다. 
다수개의 `backup server` 가 등록된 상태에서 이를 모두 한번에 사용하고 싶을 때는 `option allbackups` 설정을 추가해주면 된다. 

```
# multiple-backup-servers.cfg

defaults
    timeout connect 200ms
    timeout client 1000ms
    timeout server 1000ms

frontend ft-app
    bind 0.0.0.0:81
    default_backend bk-app-main

backend bk-app-main
    option allbackups
    server nginx1 nginx-1:81 check
    server nginx2 nginx-2:81 check
    server nginx3 nginx-3:81 check backup
    server nginx4 nginx-4:81 check backup


listen stats
    bind 0.0.0.0:8711
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth admin:admin
```  

대부분 설정은 앞선 예제와 비슷하고, 차이점이라면 `bk-app-main` 에 `option allbakcups` 설정이 추가된 부분이다.  

테스트를 위한 서비스를 구성한 `docker-compose-multiple-backup-servers.yaml` 파일의 내용은 아래와 같다. 

```yaml
# docker-compose-multiple-backup-server.yaml

version: '3.7'

services:
  haproxy:
    image: haproxy:latest
    volumes:
      - ./multiple-backup-servers.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - 8182:81
      - 8712:8711
    networks:
      - ha-net

.. 이하 동일 ..

```  

`haproxy` 서비스에서 마운트한 설정파일과 호스트와 포워딩한 포트만 8182, 8712 로 차이가 있고 다른 내용은 모두 동일하다.  

`docker-compose -f docker-compose-multiple-backup-servers.yaml` 명령으로 실행시키면, 바로 `nginx-1, 2` 를 중지시킨다. 
그리고 `curl localhost:8182` 로 요청을 보내면 `nginx-3, 4` 로 트래픽이 전달되는 것을 확인 할 수 있다. 

```bash
$ docker-compose -f docker-compose-multiple-backup-servers.yaml up -d
Creating network "failover_ha-net" with the default driver
Creating failover_nginx-3_1 ... done
Creating failover_haproxy_1 ... done
Creating failover_nginx-2_1 ... done
Creating failover_nginx-4_1 ... done
Creating failover_nginx-1_1 ... done
$ docker-compose -f docker-compose-multiple-backup-servers.yaml stop nginx-1
Stopping failover_nginx-1_1 ... done
$ docker-compose -f docker-compose-multiple-backup-servers.yaml stop nginx-2
Stopping failover_nginx-2_1 ... done
$ curl localhost:8182
nginx-3
$ curl localhost:8182
nginx-4
$ curl localhost:8182
nginx-3
$ curl localhost:8182
nginx-4
```  

## Farm failover
로드밸런싱 대상인 서버의 상태를 체크하며 설정된 가용개수 이하일때 트래픽을 `backup server` 로 전환시킬 수 있다. 

```
# farm-failover.cfg

defaults
    timeout connect 200ms
    timeout client 1000ms
    timeout server 1000ms
    
frontend ft-app
    bind 0.0.0.0:81

    # detect capacity issues in production farm
     acl MAIN_not_enough_capacity nbsrv(bk-app-main) le 2
    # failover traffic to backup farm
     use_backend bk-app-backup if MAIN_not_enough_capacity

    default_backend bk-app-main

backend bk-app-main
    server nginx1 nginx-1:81 check
    server nginx2 nginx-2:81 check
    server nginx3 nginx-3:81 check
    server nginx4 nginx-4:81 check

backend bk-app-backup
    server nginx5 nginx-5:81 check
    server nginx6 nginx-6:81 check
    server nginx7 nginx-7:81 check
    server nginx8 nginx-8:81 check


listen stats
    bind 0.0.0.0:8711
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth admin:admin
```  

`ft-app` 프론트엔드에 `acl` 설정으로 `bk-app-main` 의 가용 개수가 2개 이하일 때의 조건을 `MAIN_not_enough_capacity` 로 설정했다. 
그리고 `use_backend` 설정으로 `MAIN_not_enough_capacity` 조건이 만족하면 `bk-app-backup` 백엔드로 트래픽을 전달한다.  

서비스를 구성한 템플릿 파일인 `docker-compose-farm-failover.yaml` 파일 내용은 아래와 같다. 

```yaml
# docker-compose-farm-failover.yaml

version: '3.7'

services:
  haproxy:
    image: haproxy:latest
    volumes:
      - ./farm-failover.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - 8183:81
      - 8713:8711
    networks:
      - ha-net

  nginx-1:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-1"
    networks:
      - ha-net
  nginx-2:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-2"
    networks:
      - ha-net

  nginx-3:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-3"
    networks:
      - ha-net

  nginx-4:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-4"
    networks:
      - ha-net

  nginx-5:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-5"
    networks:
      - ha-net


  nginx-6:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-6"
    networks:
      - ha-net


  nginx-7:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-7"
    networks:
      - ha-net


  nginx-8:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/templates
    environment:
      NGINX_PORT: 81
      STR: "nginx-8"
    networks:
      - ha-net

networks:
  ha-net:
```  

`haproxy` 서비스에 마운트하는 설정파일 이름과 포트가 변경되었고, 
`Nginx` 서비스가 8개로 늘어났다.  

`docker-compose -f docker-compose-farm-failover.yaml up -d` 명령으로 실행하고, 
`curl localhost:8183` 으로 요청하면 `nginx-1 ~ 4`(`bk-app-main`) 까지 로드밸런싱이 되는 것을 확인 할 수 있다. 

```bash
$ docker-compose -f docker-compose-farm-failover.yaml up -d
Creating network "failover_ha-net" with the default driver
Creating failover_nginx-4_1 ... done
Creating failover_nginx-3_1 ... done
Creating failover_nginx-1_1 ... done
Creating failover_nginx-5_1 ... done
Creating failover_nginx-7_1 ... done
Creating failover_nginx-2_1 ... done
Creating failover_nginx-8_1 ... done
Creating failover_haproxy_1 ... done
Creating failover_nginx-6_1 ... done
$ curl localhost:8183
nginx-1
$ curl localhost:8183
nginx-2
$ curl localhost:8183
nginx-3
$ curl localhost:8183
nginx-4
$ curl localhost:8183
nginx-1
$ curl localhost:8183
nginx-2
$ curl localhost:8183
nginx-3
$ curl localhost:8183
nginx-4
```  

`nginx-1, 2` 를 중지시키고 다시 `curl localhost:8183` 으로 요청을 보내면 `nginx-5 ~ 8`(`bk-app-backend`) 로 트래픽이 전달되는 것을 확인 할 수 있다. 

```bash
$ docker-compose -f docker-compose-farm-failover.yaml stop nginx-1
Stopping failover_nginx-1_1 ... done
$ docker-compose -f docker-compose-farm-failover.yaml stop nginx-2
Stopping failover_nginx-2_1 ... done
$ curl localhost:8183
nginx-5
$ curl localhost:8183
nginx-6
$ curl localhost:8183
nginx-7
$ curl localhost:8183
nginx-8
$ curl localhost:8183
nginx-5
$ curl localhost:8183
nginx-6
$ curl localhost:8183
nginx-7
$ curl localhost:8183
nginx-8
```  

## Farm failover with backup servers
앞서 설명한 방법을 모두 적용하면 아래와 같다. 
로드밸런싱 대상이 가용 불가능한 상태가 되면 백업용 백엔드로 트래픽이 전달되고, 
백업용 백엔드에서도 로드밸런싱 대상이 가용 불가능한 상태가되면 설정된 백업 서버로 트래픽이 전달된다. 

```
# farm-failover-backup-servers.cfg

defaults
    timeout connect 200ms
    timeout client 1000ms
    timeout server 1000ms

frontend ft-app
    bind 0.0.0.0:81

    # detect capacity issues in production farm
     acl MAIN_not_enough_capacity nbsrv(bk-app-main) le 2
    # failover traffic to backup farm
     use_backend bk-app-backup if MAIN_not_enough_capacity

    default_backend bk-app-main

backend bk-app-main
    server nginx1 nginx-1:81 check
    server nginx2 nginx-2:81 check
    server nginx3 nginx-3:81 check
    server nginx4 nginx-4:81 check

backend bk-app-backup
    option allbackups
    server nginx5 nginx-5:81 check
    server nginx6 nginx-6:81 check
    server nginx7 nginx-7:81 check backup
    server nginx8 nginx-8:81 check backup


listen stats
    bind 0.0.0.0:8711
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth admin:admin
```  

`bk-app-backup` 백엔드에 `option allbackups` 를 설정하고, `nginx-7 ,8` 을 `backup server` 로 설정했다. 
`bk-app-main` 백엔드에서 2개 이상의 서버가 가용 불가능 상태가 되면 트래픽은 `bk-app-backup` 백엔드의 `nginx-5, 6` 으로 전달되고, 
이마저도 모두 가용 불가능하다면 `nginx-7, 8` 로 트래픽이 전달된다.  

서비스를 구성한 템플릿 파일인 `docker-compose-farm-failover-backup-servers.yaml` 파일 내용은 아래와 같다. 

```yaml
# docker-compose-farm-failover-backup-servers.yaml

version: '3.7'

services:
  haproxy:
    image: haproxy:latest
    volumes:
      - ./farm-failover-backup-servers.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - 8184:81
      - 8714:8711
    networks:
      - ha-net

.. 이하 동일 ..
```  

`haproxy` 서비스에 마운트하는 설정파일이 `farm-failover-backup-server.cfg` 인것과 포워딩 포트가 8184, 8714 인것을 제외하곤 이전 템플릿과 모두 동일하다.  

`docker-compose -f docker-compose-farm-failover-backup-servers.yaml up -d` 명령으로 서버 구성을 실행 시키고, 
`curl localhost:8184` 로 요청을 보내면 `bk-app-main` 으로 트래픽이 전달되는 것을 확인 할 수 있다.  

```bash
$ docker-compose -f docker-compose-farm-failover-backup-servers.yaml up -d
Creating network "failover_ha-net" with the default driver
Creating failover_nginx-1_1 ... done
Creating failover_nginx-2_1 ... done
Creating failover_nginx-7_1 ... done
Creating failover_nginx-8_1 ... done
Creating failover_nginx-5_1 ... done
Creating failover_nginx-6_1 ... done
Creating failover_haproxy_1 ... done
Creating failover_nginx-4_1 ... done
Creating failover_nginx-3_1 ... done
$ curl localhost:8184
nginx-1
$ curl localhost:8184
nginx-2
$ curl localhost:8184
nginx-3
$ curl localhost:8184
nginx-4
$ curl localhost:8184
nginx-1
$ curl localhost:8184
nginx-2
$ curl localhost:8184
nginx-3
$ curl localhost:8184
nginx-4
```  

`nginx-1, 2` 를 중시키고 요청을 보내면 `bk-app-backup` 백엔드의 `nginx-5, 6` 으로 트래픽이 전달된다. 

```bash
$ docker-compose -f docker-compose-farm-failover-backup-servers.yaml stop nginx-1
Stopping failover_nginx-1_1 ... done
$ docker-compose -f docker-compose-farm-failover-backup-servers.yaml stop nginx-2
Stopping failover_nginx-2_1 ... done
$ curl localhost:8184
nginx-5
$ curl localhost:8184
nginx-6
$ curl localhost:8184
nginx-5
$ curl localhost:8184
nginx-6
```  

`nginx-5, 6` 까지 중지시키면 `bk-app-backup` 백엔드에 `backup server` 로 설정한 `nginx-7, 8` 로 트래픽이 전달되는 것을 확인할 수 있다. 

```bash
$ docker-compose -f docker-compose-farm-failover-backup-servers.yaml stop nginx-5
Stopping failover_nginx-5_1 ... done
$ docker-compose -f docker-compose-farm-failover-backup-servers.yaml stop nginx-6
Stopping failover_nginx-6_1 ... done
$ curl localhost:8184
nginx-7
$ curl localhost:8184
nginx-8
$ curl localhost:8184
nginx-7
$ curl localhost:8184
nginx-8
```   



---
## Reference
[Failover and Worst Case Management with HAProxy](https://www.haproxy.com/blog/failover-and-worst-case-management-with-haproxy/)  