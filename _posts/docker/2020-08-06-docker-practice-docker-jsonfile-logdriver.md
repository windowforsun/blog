--- 
layout: single
classes: wide
title: "[Docker 실습] Docker json-file 로그 드라이버의 사용과 관리"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 가 호스트에 파일로 로그를 남기는 json-file 로그를 관리해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Docker Compose
  - Log
  - Logging
  - log-driver
  - json-file
toc: true
use_math: true
---  

## Docker Logging Driver
도커는 자체적으로 컨테이너에서 발생하는 `STDOUT`, `STDERR` 의 내용을 로그로 잘 남겨준다. 
그래서 컨테이너에서는 남기고 싶은 로그를 표준출력으로 출력하면 알아서 도커에서 호스트에 정해진 경로에 파일로 남긴다.  

도커 로그의 기본 `log-driver` 설정 값은 `json-file` 이고, `log-driver` 에 설정 가능한 드라이버의 종류는 아래와 같고 관련 자세한 설명은 [여기](https://docs.docker.com/config/containers/logging/configure/#supported-logging-drivers)
에서 확인 할 수 있다. 
- none	
- local	
- json-file
- syslog
- journald
- gelf
- fluentd
- awslogs
- splunk
- etwlogs
- gcplogs
- logentries

설정할 수 있는 다양한 로그 드라이버가 있지만 , 
본 포스트에서는 파일로 로그를 남기는 `json-file` 드라이버에 대해 다룬다. 

설치된 도커의 현재 기본 `log-driver` 값은 아래 명령어로 확인해 볼 수 있다. 

{% raw %}
```bash
$ docker info --format '{{.LoggingDriver}}'
"json-file"
```  
{% endraw %}

`json-file` 일 때 설정 가능한 옵션 값은 아래와 같고, 
관련 자세한 설명은 [여기](https://docs.docker.com/config/containers/logging/json-file/#options)
에서 확인 할 수 있다. 
- max-size	
- max-file
- labels
- env
- env-regex
- compress	


## 테스트 컨테이너
테스트 용으로 사용할 컨테이너의 `Dockerfile` 내용은 아래와 같다. 

```dockerfile
FROM ubuntu:latest

COPY generate-log.sh /generate-log.sh
RUN chmod +x /generate-log.sh

CMD ["/generate-log.sh"]
```  

로그를 계속해서 생성하는 쉘 스크립트인 `generate-log.sh` 파일 내용은 아래와 같다. 

```bash
#!/bin/sh

if [ -z "${SIZE}" ]; then
  SIZE=1
fi

if [ -z "${TIMER}" ]; then
  TIMER=1
fi

while true; do
  STR_RANDOM=`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w $SIZE | sed 1q`
  STR_RANDOM="~~~~start${STR_RANDOM}end~~~~"
  echo $STR_RANDOM
  echo $STR_RANDOM | wc -c
  sleep $TIMER
done
```  

`TIMER` 값 만큼 슬립을 수행하며 `SIZE` 만큼 랜덤 문자열을 생성하고, 
표준출력해서 로그를 남긴다.   

이미지의 태그는 `generate-log` 로 아래 명령어로 빌드를 수행한다. 

```bash
docker build --no-cache --tag generate-log .
Sending build context to Docker daemon   12.8kB
Step 1/4 : FROM ubuntu:latest
 ---> adafef2e596e
Step 2/4 : COPY generate-log.sh /generate-log.sh
 ---> 35469f5104ac
Step 3/4 : RUN chmod +x /generate-log.sh
 ---> Running in c9fd248ca1a8
Removing intermediate container c9fd248ca1a8
 ---> e61ca086bd07
Step 4/4 : CMD ["/generate-log.sh"]
 ---> Running in 578e6c15c63a
Removing intermediate container 578e6c15c63a
 ---> a3d3c892b183
Successfully built a3d3c892b183
Successfully tagged generate-log:latest
```  

## Docker Daemon 로그 관리 
호스트에 설치된 도커에 로그 관련 설정을 해서, 
해당 `Docker Daemon` 에서 실행되는 모든 컨테이너의 로그를 한번에 관리할 수 있다.  

설정 파일 경로는 아래와 같다. 
- `Linux` : `/etc/docker/daemon.json`
- `Windows` : `C:\Users\<사용자이름>\.docker\daemon.json`

`daemon.json` 파일에 로그 설정 예시는 아래와 같다. 

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "2m",
    "max-file": "2" 
  }
}
```  

위 설정은 해당 `Docker Daemon` 에서 실행되는 모든 컨테이너에 기본으로 적용되는 로그 설정이고, 
`Docker Daemon` 을 재시작 하면 적용된다. 

테스트를 위해서 앞서 생성한 이미지로 컨테이너를 `SIZE=1000000`, `TERM=2` 으로 환경 변수를 설정해서 실행 한다. 

```bash
$ docker run \
-d \
--rm \
--name log-test \
-e SIZE=1000000 \
-e TIMER=2 \
generate-log:latest
180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc
```  

호스트에서 실행되는 컨테이너들이 사용하는 로그의 경로는 아래의 명령어로 확인해 볼 수 있다. 

{% raw %}
```bash
docker inspect 180 --format '{{.LogPath}}'
/var/lib/docker/containers/180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc/180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc-json.log
```  
{% endraw %}

>WSL2 를 사용 중이라면 상위 경로중 `/var/lib` 대신 `/mnt/wsl/docker-desktop-data/data` 을 사용해야 한다. 

해당 경로로 접근해서 파일 리스트를 확인해 보면 아래와 같다. 

```bash
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc
$ ll
total 3.0M
-rw-r----- 1 root root 982K Aug  6 20:22 180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc-json.log
-rw-r----- 1 root root 2.0M Aug  6 20:22 180f0b5b3a42e8ecb504291296623ee12cc9751d775ea64a87969ba45a5ce6dc-json.log.1

.. 생략 ..
```  

`<컨테이너 아이디>-json.log` 형식으로 2개의 파일이 있는 것을 확인 할 수 있다. 
그리고 각 로그 파일은 최대 크기인 `2MB` 를 넘지 않게 계속해서 관리 된다. 


## Docker Container 로그 관리
개별 컨테이너 마다 실행 시 로그 옵션을 통해 로그 관리를 설정할 수 있다. 
아래 명령어로 `max-size=1k`, `max-file=3` 로그 옵션과 `SIZE=1000`, `TIMER=2` 로 컨테이너를 실행 한다. 

```bash
$ docker run \
-d \
--rm \
--name log \
--log-opt max-size=1k \
--log-opt max-file=3 \
-e SIZE=1000 \
-e TIMER=2 \
generate-log:latest
b11880b9d189f0e0e6baff5b383fcda4bae77146ceca79c0fd3ed3eebc767200
```  

컨테이너의 로그 경로로 이동해서 확인하면 아래와 같이 

```bash
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/b11880b9d189f0e0e6baff5b383fcda4bae77146ceca79c0fd3ed3eebc767200
$ ll
total 44K
-rw-r----- 1 root root   73 Aug  6 20:55 b11880b9d189f0e0e6baff5b383fcda4bae77146ceca79c0fd3ed3eebc767200-json.log
-rw-r----- 1 root root 1.2K Aug  6 20:55 b11880b9d189f0e0e6baff5b383fcda4bae77146ceca79c0fd3ed3eebc767200-json.log.1
-rw-r----- 1 root root 1.2K Aug  6 20:54 b11880b9d189f0e0e6baff5b383fcda4bae77146ceca79c0fd3ed3eebc767200-json.log.2

.. 생략 ..
```  

앞서 설정한 `daemon.json` 의 로그 설정을 따라가지 않고, 
컨테이너를 실행할때 설정한 로그 설정인 파일 최대 크기는 `1KB`, 파일 개수는 3개로 관리 되는 것을 확인 할 수 있다. 

## Docker Compose Service 로그 관리
`Docker Compose` 를 사용하면 서비스에 해당하는 각 컨테이너의 설정을 템플릿화해서 유지보수와 문서화를 할 수 있다. 
그래서 `Docker Compose` 에서 로그 설정을 하면 서비스 단위로 로그를 관리할 수 있다.  

```yaml
services:
    service-1:
      logging:
        drivers: <로그 드라이버>
        options:
          max-size: "<로그 파일 크기>"
          max-file: "<로그파일개수>"
          
```  

### 심플 서비스
심플 서비스는 `generate-log` 이미지를 `Docker Compose` 로 간단하게 구성한 서비스이다. 
`docker-compose.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  generate-log:
    image: generate-log:latest
    deploy:
      replicas: 3
    environment:
      SIZE: 1000000
      TIMER: 1
    logging:
      options:
        max-size: "600k"
        max-file: "5"
```  

- `generate-log` 서비스에 1초마다 `1MB` 크기 로그를 출력한다. 
- 서비스의 로그 설정으로 `600KB` 크기의 로그파일을 5개관리하도록 설정했다. 

`docker stack deploy -c docker-compose.yaml log-simple` 명령으로 서비스를 `Swarm` 클러스터에 적용한다. 
그리고 서비스에서 실행 중인 컨테이너를 조회하고, 컨테이너의 로그 경로로 이동해 로그 리스트를 확인하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose.yaml log-simple
Creating network log-simple_default
Creating service log-simple_generate-log
$ docker ps | grep log-simple
59203eab1cbc        generate-log:latest           "/generate-log.sh"       11 seconds ago      Up 9 seconds                    log-simple_generate-log.1.yiojdfvhi0vzhqb861dzk4i6e
ac1963b7a917        generate-log:latest           "/generate-log.sh"       11 seconds ago      Up 9 seconds                    log-simple_generate-log.2.zse3rx0wfijs3muwvbvbpr5ca
9807eb24e61c        generate-log:latest           "/generate-log.sh"       11 seconds ago      Up 9 seconds                    log-simple_generate-log.3.0jzmewgzcx6ah2v97tegfyxcw
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f
$ ll
total 2.7M
-rw-r----- 1 root root 339K Aug  6 21:27 59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f-json.log
-rw-r----- 1 root root 595K Aug  6 21:27 59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f-json.log.1
-rw-r----- 1 root root 596K Aug  6 21:27 59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f-json.log.2
-rw-r----- 1 root root 596K Aug  6 21:27 59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f-json.log.3
-rw-r----- 1 root root 595K Aug  6 21:27 59203eab1cbcb7b87215b55a5532687838c31aad452a4505fbdb25f3934c6a9f-json.log.4

.. 생략 ..
```  

설정된 크기와 개수로 컨테이너 로그파일이 관리되는 것을 확인 할 수 있다.  

`docker-compose.yaml` 에서 로그 설정을 아래와 같이 수정하고, 
다시 클러스터에 적용해서 확인해보면 템플릿의 수정사항이 정상적으로 적용되는 것도 확인 가능하다. 

```bash
$ vi docker-compose.yaml

services:
  generate-log:

    .. 생략 ..

    logging:
      options:
        max-size: "1m"
        max-file: "2"

$ docker stack deploy -c docker-compose.yaml log-simple
Updating service log-simple_generate-log (id: 9lw47qfqdcf73rseva9sqyoma)
image generate-log:latest could not be accessed on a registry to record
its digest. Each node will access generate-log:latest independently,
possibly leading to different nodes running different
versions of the image.
$ docker ps | grep log-simple
69dc5460767d        generate-log:latest           "/generate-log.sh"       14 seconds ago      Up 2 seconds                  log-simple_generate-log.2.zdkh09blbuxf2y90z7f0qkqva
0db1c6f0775f        generate-log:latest           "/generate-log.sh"       28 seconds ago      Up 16 seconds                  log-simple_generate-log.1.ozfwupy9cpw7bvy986h057app
fa01f887a9a0        generate-log:latest           "/generate-log.sh"       43 seconds ago      Up 31 seconds                  log-simple_generate-log.3.oziz1gjz4tzoftfff6xa0728p
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/69dc5460767d1be4f446afd8db091c80f8fdd1bb3c415cc0d50ae17a075f5b7c
$ ll
total 1020K
-rw-r----- 1 root root  737 Aug  6 21:35 69dc5460767d1be4f446afd8db091c80f8fdd1bb3c415cc0d50ae17a075f5b7c-json.log
-rw-r----- 1 root root 981K Aug  6 21:35 69dc5460767d1be4f446afd8db091c80f8fdd1bb3c415cc0d50ae17a075f5b7c-json.log.1

.. 생략 ..
```  

### EFK 서비스
간단하게 `EFK` 스택을 구성해서 해당 구성에서 몇가지 테스트를 진행해 본다. 
서비스의 구성은 [여기]({{site.baseurl}}{% link _posts/docker/2020-07-21-docker-practice-efk-biglog.md %})
에서 진행한 템플릿과 대부분 유사하고 다른 점은 로그를 생성하는 서비스를 이번에는 앞서 빌드한 `generate-log` 이미지는 사용 한다는 점이다.  

사용하는 디렉토리의 구조는 아래와 같다. 

```bash
efk
├── docker-compose.yaml
└── fluentd
    ├── Dockerfile
    └── conf
        └── fluent.conf
```  

구성한 `docker-compose.yaml` 의 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  generate-log:
    image: generate-log:latest
    environment:
      SIZE: 100000
      TIMER: 1
    deploy:
      replicas: 3
    networks:
      - log-test
    depends_on:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: :24225
        tag: my.log

  fluentd:
    image: test-fluentd
    networks:
      - log-test
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24225:24225"
      - "24225:24225/udp"
    logging:
      options:
        max-size: "1m"
        max-file: "2"


  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.8.0
    networks:
      - log-test
    ports:
      - 9200:9200
    environment:
      - node.name=elasticsearch
      - cluster.initial_master_nodes=elasticsearch
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.8.0
    networks:
      - log-test
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200

networks:
  log-test:
    driver: overlay
```  

- `.services.generate-log` : 로그를 생성하고 `.logging` 설정으로 `fluentd` 로 해당 로그를 `my.log` 태그로 전송한다. 
로그는 `100KB` 로그를 1초마다 생성한다. 
- `.services.fluentd` : `generate-log` 서비스로 부터 로그를 받아 `elasticsearch` 서비스로 전송하는 역할을 수행한다. 
`fluentd` 의 로그 파일은 최대 `1MB` 크기의 파일을 2개로 관리하도록 설정했다. 
- `.services.elasticsearch` : `fluentd` 서비스로 부터 로그를 받아 로그를 저장하고 인덱싱 하는 역할을 수행한다. 
- `.services.kibana` : `elasticsearch` 에서 관리하고 있는 로그를 조회 할 수 있는 서비스이다. 

`fluentd` 서비스는 `test-fluentd` 라는 이미지를 사용한다. 
해당 이미지에 대한 스크립트인 `Dockerfile` 과 `fluentd` 관련 설정 파일인 `fluent.conf` 파일의 내용은 아래와 같다. 

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.7
USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
RUN ["gem", "install", "fluent-plugin-rewrite-tag-filter"]
RUN ["gem", "install", "fluent-plugin-concat"]
USER fluent
```  

```
<source>
    @type forward
    port 24225
    bind 0.0.0.0
</source>

# 큰 로그로 인해 Docker 에서 나눠 찍힌 로그를 하나의 로그로 합친다.
<filter **>
    @type concat
    key log
    use_first_timestamp true
    use_partial_metadata true
    separator ""
</filter>

<match my.**>
    @type copy

    # elaticsearch 로 해당 로그를 전송한다.
    <store>
      @type elasticsearch
      # elasticsearch 호스트 설정
      host elasticsearch
      port 9200
      logstash_format true
      logstash_prefix fluentd
      logstash_dateformat %Y%m%d
      include_tag_key true
      type_name access_log
      tag_key @log_name
      flush_interval 1s
    </store>

    # 표준 출력으로 출력한다.
    <store>
        @type stdout
    </store>
</match>
```  

아래 명령어로 `fluentd` 서비스에서 사용할 이미지를 빌드한다. 

```bash
$ pwd
/efk/fluentd
$ docker build --no-cache --tag test-fluentd .
Sending build context to Docker daemon  4.608kB
Step 1/6 : FROM fluent/fluentd:v1.7
---> f7b0c84773fe
Step 2/6 : USER root
---> Running in e49816b1bb1b
Removing intermediate container e49816b1bb1b
---> 9d3b6ea982fb
Step 3/6 : RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
---> Running in cd864c810ddf

.. 생략 ..

Successfully built 0f3a4941ea78
Successfully tagged test-fluentd:latest
```  

`EFK` 스택을 구성한 `Docker Compose` 를 `docker stack deploy -c docker-compose.yaml log-efk` 명령으로 클러스터에 적용한다. 
그리고 `fluentd` 서비스에서 실행되는 컨테이너에 로그 경로를 조회하면 아래와 같다. 

```bash
$ docker stack deploy -c docker-compose.yaml log-efk
Creating network log-efk_log-test
Creating service log-efk_fluentd
Creating service log-efk_elasticsearch
Creating service log-efk_kibana
Creating service log-efk_generate-log
$ docker ps | grep log-efk_fluentd
3cb959e1b852        test-fluentd:latest                                   "tini -- /bin/entryp…"   30 seconds ago      Up 24 seconds       5140/tcp, 24224/tcp   log-efk_fluentd.1.ppxxr17p9a29xksveh8wq2hti
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/3cb959e1b852aba6f65b7c656d3806bce55efbb9cef04d15a94238ee7e4048a7
$ ll
total 1.8M
-rw-r----- 1 root root 793K Aug  6 22:00 3cb959e1b852aba6f65b7c656d3806bce55efbb9cef04d15a94238ee7e4048a7-json.log
-rw-r----- 1 root root 988K Aug  6 22:00 3cb959e1b852aba6f65b7c656d3806bce55efbb9cef04d15a94238ee7e4048a7-json.log.1

.. 생략 ..
```  

`http://localhost:5601` 에 접속해 키바나에서 인덱스 설정을 하고, 확인해보면 애플리케이션에서 생성한 큰 로그가 키바나까지 잘 전달되는 것을 확인 할 수 있다.  

지금 `json` 형식의 로그를 파일 단위로 크기와 개수 제한을 두고 관리하고 있다. 
항상 한 로그가 한 파일에 완전히 남지 못하고, 
두 파일로 나눠지는 경우에도 `fluentd` 서비스에서 나눠져서 전달된 로그를 합쳐서 `elasticsearch` 에 전달 할 수 있는 지에 대한 테스트를 진행해 본다.  

`docker-compose.yaml` 파일에서 `fluentd` 서비스의 로그 설정을 아래와 같이 애플리케이션에서는 `100KB` 로그를 생성하지만, 
로그 파일의 최대 크기를 `60KB` 로 수정하고 파일개수도 수정한다. 
그리고 다시 `docker stack deploy` 명령으로 스택을 클러스터에 적용해 한다. 

```bash
$ vi docker-compose.yaml

services:

  .. 생략 ..

  fluentd:

  .. 생략 ..

    logging:
      options:
        max-size: "60k"
        max-file: "5"

  .. 생략 ..

$ docker stack deploy -c docker-compose.yaml log-efk
Updating service log-efk_generate-log (id: im3cdoes7nmr8c9202oes6yv0)
image generate-log:latest could not be accessed on a registry to record
its digest. Each node will access generate-log:latest independently,
possibly leading to different nodes running different
versions of the image.

Updating service log-efk_fluentd (id: q0gzssq53iylcwn3uqwxsyls2)
image test-fluentd:latest could not be accessed on a registry to record
its digest. Each node will access test-fluentd:latest independently,
possibly leading to different nodes running different
versions of the image.

Updating service log-efk_elasticsearch (id: k5l9q5un0b49u9jr6537c81m2)
Updating service log-efk_kibana (id: nlb9pv2m7qxq0hkfsxvchbam5)
```  

`fluentd` 서비스에서 실행 중인 컨테이너를 확인해서 컨테이너의 로그 경로를 조회하면 아래와 같이, 
수정된 로그 설정 만큼 로그 파일이 관리되는 것을 확인 할 수 있다. 

```bash
$ docker ps | grep log-efk_fluentd
45eb33daef80        test-fluentd:latest                                   "tini -- /bin/entryp…"   3 minutes ago       Up 3 minutes        5140/tcp, 24224/tcp   log-efk_fluentd.1.u9s8fm7lfr5bmamj9zidutqmq
$ cd /mnt/wsl/docker-desktop-data/data/docker/containers/45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b
$ ll
total 344K
-rw-r----- 1 root root  35K Aug  6 22:11 45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b-json.log
-rw-r----- 1 root root  67K Aug  6 22:11 45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b-json.log.1
-rw-r----- 1 root root  65K Aug  6 22:11 45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b-json.log.2
-rw-r----- 1 root root  67K Aug  6 22:11 45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b-json.log.3
-rw-r----- 1 root root  67K Aug  6 22:11 45eb33daef8025f8bb0fa2b266389a0cea7006b322e46985e37d155434745b8b-json.log.4

.. 생략 ..
```  

다시 `http://localhost:5601` 에 접속해서 키바나 로그를 확인하면 이전과 동일하게, 
애플리케이션에서 생성한 `100KB` 로그가 모두 하나의 로그로 잘 남는 것을 확인 할 수 있다.  

---
## Reference
[JSON File logging driver](https://docs.docker.com/config/containers/logging/json-file/)  
[Docker Compose logging](https://docs.docker.com/compose/compose-file/#logging)  
[View logs for a container or service](https://docs.docker.com/config/containers/logging/)  
[Configure logging drivers](https://docs.docker.com/config/containers/logging/configure/)  