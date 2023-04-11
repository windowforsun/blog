--- 
layout: single
classes: wide
title: "[Elasticsearch 실습] ELK with Filebeat + Kafka 로그 수집"
header:
  overlay_image: /img/elasticsearch-bg.png
excerpt: 'ELK 스택과 Filebeat 와 Kafka 를 사용해서 로그를 수집하는 방법과 구성에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Elasticsearch
tags:
    - Practice
    - Elasticsearch
    - Filebeat
    - ELK
    - Logstash
    - Kibana
    - Nginx
    - Kafka
toc: true
use_math: true
---  

## ELK Stack with Kafka
[ELK with Filebeat]({{site.baseurl}}{% link _posts/elasticsearch/2023-04-08-elasticsearch-practice-elk.md %})
에서는 `ELK` 스택에 `Filebeat` 을 사용해서 로그를 수집하는 방법에 대해 알아보았다. 
여기서 `Kafka` 까지 중간에 추가해주면 좀더 대용량의 로깅을 처리할 수 있는 구성으로 만들 수 있다. 
대용량 메시지 처리 플랫폼인 `Kafka` 가 `Filebeat` 과 `Logstash` 중간에 있기 때문에 
`Filebeat` 은 자신이 가능한 만큼 `Kafka` 에 로그를 밀어 넣고, 
`Logstash` 도 자신이 가능한 만큼 `Kafka` 에서 로그를 가져와 `Elastidsearch` 에 넣을 수 있기 때문이다. 
즉 `Kafka` 가 중간에서 갑작스럽게 많은 로그가 발생하더라도 적절하게 중재를 해주는 역할을 해준다는 의미이다.  


### Example
구성할 `ELK + Kafka` 의 간단한 예시는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-kafka-1.drawio.png)

전체 예저 구성은 [ELK with Filebeat]({{site.baseurl}}{% link _posts/elasticsearch/2023-04-08-elasticsearch-practice-elk.md %})
에서 사용한 것과 대부분 동일하고, `Kafka` 구성 추가 및 관련 설정들만 변경 되었다. 
동일한 부분은 빠르게 넘어가고 변경사항이 있는 부분에 대해서만 조금 더 자세히 설명할 예정이다.  

예제는 `Docker` 와 `docker-compose` 를 기반으로 구성한다.  

예제에 필요한 전체 파일이 있느 디렉토리 트리구조는 아래와 같다.  

```
.
├── README.md
├── app-filebeat
│   ├── Dockerfile
│   ├── filebeat.yaml
│   └── script.sh
├── docker-compose.yaml
├── logstash
│   ├── my-logstash.conf
│   └── patterns
│       ├── gin
│       └── nginx
└── nginx-lb
    └── nginx.conf
```

#### docker-compose
구성하고자 하는 `ELK + Kafka` 의 전체 구성을 담고 있는 `docker-compose.yaml` 파일 내용은 아래와 같다.  

```yaml
# docker-compose.yaml

version: '3.3'

services:
  es-single:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: es-single
    environment:
      - xpack.security.enabled=false
      - node.name=es-single
      - cluster.name=es-single
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - es-net

  kibana:
    image: docker.elastic.co/kibana/kibana:8.6.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - "ELASTICSEARCH_HOSTS=http://es-single:9200"
    networks:
      - es-net

  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    container_name: logstash
    environment:
      ES_IP: es-single
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    volumes:
      - ./logstash/:/usr/share/logstash/pipeline/
    ports:
      - "5000:5000"
      - "9600:9600"
    depends_on:
      - es-single
      - kibana
    networks:
      - es-net

  # Kafka 구성 추가
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      - es-net

  # Kafka 구성 추가
  kafka:
    container_name: myKafka
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper
    networks:
      - es-net

  nginx-lb:
    image: nginx:1.12
    ports:
      - "8111:80"
    volumes:
      - ./nginx-lb/nginx.conf:/etc/nginx/nginx.conf
    networks:
      - es-net

  app-filebeat:
    build:
      context: app-filebeat/
    environment:
      - "LOGSTASH_IP=logstash"
    networks:
      - es-net
    depends_on:
      - logstash
    deploy:
      mode: replicated
      replicas: 3

networks:
  es-net:
```  


#### Logstash
`Logstash` 설정은 `/usr/share/logstash/pipeline/` 하위에 `logstash` 로 시작하는 파일을 두면 된다. 
그리고 `pipeline/patterns` 디렉토리에 `logstash` 설정에서 사용할 `grok` 정규식 사전 파일을 두면 파상할 로그의 
패턴을 미리 정의해서 사용 할 수 있다.  

로그의 패턴을 미리 정의하는 `pipeline/patterns/nginx` 파일 내용은 아래와 같다.  

```
# nginx
NGINXACCESS %{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \[%{HTTPDATE:[nginx][access][time]}\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\"
NGINXERROR %{DATA:[nginx][error][time]} \[%{DATA:[nginx][error][level]}\] %{NUMBER:[nginx][error][pid]}#%{NUMBER:[nginx][error][tid]}: (\*%{NUMBER:[nginx][error][connection_id]} )?%{GREEDYDATA:[nginx][error][message]}
```  

`Logstash` 의 설정을 담고 있는 `logstash.conf` 파일 내용은 아래와 같다. 
설정에도 나와있지만, `Logstash` 는 `Kafka Topic` 으로 부터 로그를 전달 받아 파이프라인을 진행한다. 
이전 `ELK Filebeat` 과는 달리 `Access`, `Error` 를 별도의 인덱스로 구성해 보았다. 

```
input {
    # Filebeat 에서 사용한 Kafka 토픽을 통해 로그 수신
    kafka {
        client_id => "logstash-app-filebeat-nginx"
        group_id => "logstash-app-filebeat-nginx"
        # ,(콤마)로 구분한다.
        bootstrap_servers => "kafka:9092"
        #topics => ["app-filebeat-nginx"]
        topics => ["app-filebeat-nginx-access", "app-filebeat-nginx-error"]
        codec => json { }
    }
}

# grok 사전 파일을 사용해서 패턴을 통해 데이터를 파싱하고 정제한다. 
filter {
  if "nginx-access" in [tags] {
    grok {
      patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
      match => {"message" => ["%{NGINXACCESS}"]}
      remove_field => "message"
    }
    mutate {
      add_field => { "read_timestamp" => "%{@timestamp}" }
    }
    date {
      match => [ "[nginx][access][time]", "dd/MMM/YYYY:H:m:s Z" ]
      remove_field => "[nginx][access][time]"
    }
    useragent {
      source => "[nginx][access][agent]"
      target => "[nginx][access][user_agent]"
      remove_field => "[nginx][access][agent]"
    }
  }
  else if "nginx-error" in [tags] {
    grok {
      patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
      match => { "message" => ["%{NGINXERROR}"] }
      remove_field => "message"
    }
    mutate {
      rename => { "@timestamp" => "read_timestamp" }
    }
    date {
      match => [ "[nginx][error][time]", "YYYY/MM/dd H:m:s" ]
      remove_field => "[nginx][error][time]"
    }
  }
}

output {
  # stdout 으로 logstash 처리과정 모니터링이 가능하다. 
  stdout {
    codec => rubydebug
  }

  # Nginx access 로그 
  if "nginx-access" in [tags] {
      elasticsearch {
        action => "index"
        hosts => ["${ES_IP:=localhost}:9200"]
        manage_template => false
        #index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        index => "app-filebeat-nginx-access-%{+YYYY.MM.dd}"
      }
  }
  # Nginx error 로그 
  else if  "nginx-error" in [tags] {
      elasticsearch {
        action => "index"
        hosts => ["${ES_IP:=localhost}:9200"]
        manage_template => false
        #index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        index => "app-filebeat-nginx-error-%{+YYYY.MM.dd}"
      }
  }
}
```  


#### Filebeat
`Filebeat` 이 설치되는 컨테이너는 `app-filebeat` 이다. 
`Nginx` 기본 이미지에 추가적으로 `Filebeat` 을 설치해서 `Nginx` 의 `Access` 와 `Error` 로그를 전송한다. 
참고로 `Filebeat` 이미지를 사용해서 별도 컨테이터로 구성 할 수도 있다.  

사용할 `Filebeat` 의 설정 파일인 `filebeat.yaml` 내용은 아래와 같다. 
구현하고자 하는 동작은 호스트의 로그파일을 읽어 `Logstash` 로 전송하는 것이기 때문에 
`type` 은 `filestream` 으로 지정한다. 
각 로그 파일을 `Logstash` 에서도 구분할 수 있도록 태그를 별도로 달아 준다.  

`Filebeat` 의 로그 전송은 `Kafka` 의 특정 토픽에 수행한다. 
그리고 토픽은 로그 파일마다 구분해서 `fields.doc` 에 명시해 사용한다.  

```yaml
filebeat.inputs:
  - type: filestream
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      doc: "app-filebeat-nginx-access"
    tags: ["nginx-access"]

  - type: filestream
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      doc: "app-filebeat-nginx-error"
    tags: ["nginx-error"]

output.kafka:
  hosts: ["kafka:9092"]
  topic: '%{[fields.doc]}'
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: lz4
  max_message_bytes: 1000000

logging.level: debug
logging.to_stderr: true
logging.to_syslog: true

```  

실행할 컨테너의 이미지를 빌드하는 `Dockerfile` 내용은 아래와 같다.  

```dockerfile
FROM nginx:1.21.3

WORKDIR /root

RUN cp /etc/apt/sources.list sources.list_backup
RUN sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
RUN sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
RUN apt-get update && apt-get upgrade -y
RUN apt-get -y update
RUN apt-get install wget gnupg -y

# nginx 도커 이미지에서는 docker logs에 로깅할 수 있도록
# access.log는 stdout으로, error.log는 stderr로 symbolic link를 만들어두었다.
# 우리는 파일 기반으로 처리할 것이므로 symbolic link를 해제한다.
RUN rm -f /var/log/nginx/access.log
RUN rm -f /var/log/nginx/error.log

# filebeat 다운로드 및 실행
# 참고: https://www.elastic.co/guide/en/beats/filebeat/current/setup-repositories.html
RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
RUN apt-get install apt-transport-https -y
RUN echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list
RUN apt-get update && apt-get install filebeat -y
COPY ./filebeat.yaml /etc/filebeat/filebeat.yml
RUN update-rc.d filebeat defaults 90 10

# entrypoint 스크립트 복사 실행
COPY ./script.sh /root/

ENTRYPOINT bash script.sh
```  

마지막으로 컨테이너가 실행될 때 `Nginx` 와 `Filebeat` 을 모두 실행 해줄 `Entrypoint` 인 `script.sh` 내용은 아래와 같다.  

```bash
#!/bin/bash

# filebeat를 background로 실행하고, stderr는 stdout으로 변경하여 filebeat.log에 기록
nohup filebeat -e -c /etc/filebeat/filebeat.yml > filebeat.log 2>&1 &

# docker container를 detached로 실행할 경우, nginx를 daemon으로 실행하면 안된다.
nginx -g 'daemon off;'
```  


#### Nginx Loadbalancer
`app-filebeat` 컨테이너들의 `LoadBalancer` 역할을 수행하는 컨테이너로 `Nginx` 로 구성된다. 
필요한 `nginx.conf` 내용은 아래와 같다.  

```
user  nginx;
worker_processes  8;

events {
    worker_connections  1024;
}

http {
    log_format main '$remote_addr [$time_local] "$request" $status $body_bytes_sent $request_time $connection';

    keepalive_timeout  0;

    upstream app {
        server app-filebeat:80;
    }

    server {
      listen 80;

      access_log /dev/stdout main;
      error_log /dev/stderr warn;

      location / {
          proxy_pass http://app;

      }

      location = /server-status {
          stub_status on;
          access_log off;
      }
    }
}
```  

### 테스트
`docker-compose up --build` 명령으로 전체 구성을 실행시킨다. 
실행시키면 `app-filebeat` 이미지 빌드와 함꼐 실행 된다.  

```bash
$ docker-compose up --build
[+] Building 2.7s (24/24) FINISHED                                                                                                                                                                                                      
 => [internal] load build definition from Dockerfile                                                                                                                                                                               0.0s
 => => transferring dockerfile: 1.47kB                                                                                                                                                                                             0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                  0.0s
 => => transferring context: 2B                                                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/nginx:1.21.3                                                                                                                                                                    0.3s
 => [ 1/18] FROM docker.io/library/nginx:1.21.3@sha256:644a70516a26004c97d0d85c7fe1d0c3a67ea8ab7ddf4aff193d9f301670cf36                                                                                                            0.0s
 => => resolve docker.io/library/nginx:1.21.3@sha256:644a70516a26004c97d0d85c7fe1d0c3a67ea8ab7ddf4aff193d9f301670cf36                                                                                                              0.0s
 => [internal] load build context                                                                                                                                                                                                  0.0s
 => => transferring context: 64B                                                                                                                                                                                                   0.0s
 => CACHED [ 2/18] WORKDIR /root                                                                                                                                                                                                   0.0s
 => CACHED [ 3/18] RUN mkdir -p /usr/share/nginx/html/js                                                                                                                                                                           0.0s
 => CACHED [ 4/18] RUN cp /etc/apt/sources.list sources.list_backup                                                                                                                                                                0.0s
 => CACHED [ 5/18] RUN sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list                                                                                                                                   0.0s
 => CACHED [ 6/18] RUN sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list                                                                                                                                  0.0s
 => CACHED [ 7/18] RUN apt-get update && apt-get upgrade -y                                                                                                                                                                        0.0s
 => CACHED [ 8/18] RUN apt-get -y update                                                                                                                                                                                           0.0s
 => CACHED [ 9/18] RUN apt-get install wget gnupg -y                                                                                                                                                                               0.0s
 => CACHED [10/18] RUN rm -f /var/log/nginx/access.log                                                                                                                                                                             0.0s
 => CACHED [11/18] RUN rm -f /var/log/nginx/error.log                                                                                                                                                                              0.0s
 => CACHED [12/18] RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -                                                                                                                               0.0s
 => CACHED [13/18] RUN apt-get install apt-transport-https -y                                                                                                                                                                      0.0s
 => CACHED [14/18] RUN echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list                                                                                  0.0s
 => CACHED [15/18] RUN apt-get update && apt-get install filebeat -y                                                                                                                                                               0.0s
 => CACHED [16/18] COPY ./filebeat.yaml /etc/filebeat/filebeat.yml                                                                                                                                                                 0.0s
 => CACHED [17/18] RUN update-rc.d filebeat defaults 90 10                                                                                                                                                                         0.0s
 => CACHED [18/18] COPY ./script.sh /root/                                                                                                                                                                                         0.0s
 => exporting to docker image format                                                                                                                                                                                               2.6s
 => => exporting layers                                                                                                                                                                                                            0.0s
 => => exporting manifest sha256:2e440491df0209120f820c0d332cdeede16910219d0fc3b7b97a15504c5cca08                                                                                                                                  0.0s
 => => exporting config sha256:ed1de48db11fb1cd29396c7d08c392fe2870714e6f3cfe7a93ac9dabce1b2395                                                                                                                                    0.0s
 => => sending tarball                                                                                                                                                                                                             2.6s
 => importing to docker                                                                                                                                                                                                            0.0s
[+] Running 6/6
 ⠿ Container es-single             Created                                                                                                                                                                                         0.0s
 ⠿ Container kibana                Created                                                                                                                                                                                         0.0s
 ⠿ Container logstash              Created                                                                                                                                                                                         0.0s
 ⠿ Container kafka-app-filebeat-3  Created                                                                                                                                                                                         0.0s
 ⠿ Container kafka-app-filebeat-1  Created                                                                                                                                                                                         0.0s
 ⠿ Container kafka-app-filebeat-2  Created                                                                                                                                                                                         0.0s
Attaching to es-single, kafka-app-filebeat-1, kafka-app-filebeat-2, kafka-app-filebeat-3, kafka-nginx-lb-1, kibana, logstash, myKafka, myZookeeper

```  

그리고 `nginx-lb` 포트인 `8111` 에 아래와 같은 몇개 요청을 수행한다. 

```bash

.. 6번 반복 ..
$ curl localhost:8111
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

.. 6번 반복 ..
$ curl localhost:8111/nopath
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.21.3</center>
</body>
</html>
```  

요청을 수행한 후 `docker-compose` 혹은 `logstash` 의 도커 컨테이너 로그를 보면 `Nginx` 로그에 대한 출력을 확인 할 수 있다.  

추가적으로 `Kafka` 토픽과 토픽의 내용을 조회하면 아래와 같이 `Filebeat` 과 `Logstash` 에서 사용하는 토픽이 생성된것과 해당 토픽에 로그가 저장돼 있는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-topics.sh --bootstrap-server localhost:9092 --list
__consumer_offsets
app-filebeat-nginx-access
app-filebeat-nginx-error

$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app-filebeat-nginx-access --from-beginning
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"log":{"offset":0,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:41 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"ecs":{"version":"1.12.0"},"host":{"name":"053dd9dc5419"},"agent":{"name":"053dd9dc5419","type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c","id":"3db14b2e-4831-43c4-926d-97136d781be1"}}
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"fields":{"doc":"app-filebeat-nginx-access"},"ecs":{"version":"1.12.0"},"host":{"name":"714d9ae40b4a"},"agent":{"version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a","type":"filebeat"},"log":{"offset":0,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:43 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"}}
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"agent":{"id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59","name":"9f10de4f0a06","type":"filebeat","version":"7.17.9","hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e"},"log":{"offset":0,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:45 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"fields":{"doc":"app-filebeat-nginx-access"},"input":{"type":"filestream"},"ecs":{"version":"1.12.0"},"host":{"name":"9f10de4f0a06"}}
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"agent":{"id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419","type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c"},"log":{"file":{"path":"/var/log/nginx/access.log"},"offset":90},"message":"10.89.2.6 - - [11/Apr/2023:22:11:46 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"ecs":{"version":"1.12.0"},"host":{"name":"053dd9dc5419"}}
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"ecs":{"version":"1.12.0"},"host":{"name":"714d9ae40b4a"},"agent":{"version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a","type":"filebeat"},"message":"10.89.2.6 - - [11/Apr/2023:22:11:48 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","log":{"offset":90,"file":{"path":"/var/log/nginx/access.log"}},"tags":["nginx-access"],"fields":{"doc":"app-filebeat-nginx-access"},"input":{"type":"filestream"}}
{"@timestamp":"2023-04-11T22:11:50.656Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"agent":{"ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59","name":"9f10de4f0a06","type":"filebeat","version":"7.17.9","hostname":"9f10de4f0a06"},"log":{"offset":90,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:50 +0000] \"GET / HTTP/1.0\" 200 615 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"ecs":{"version":"1.12.0"},"host":{"name":"9f10de4f0a06"}}
{"@timestamp":"2023-04-11T22:11:56.663Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"host":{"name":"053dd9dc5419"},"agent":{"type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c","id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419"},"ecs":{"version":"1.12.0"},"log":{"offset":180,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:55 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\""}
{"@timestamp":"2023-04-11T22:12:02.666Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"log":{"offset":276,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:12:00 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"agent":{"type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c","id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419"},"ecs":{"version":"1.12.0"},"host":{"name":"053dd9dc5419"}}
{"@timestamp":"2023-04-11T22:12:04.664Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"message":"10.89.2.6 - - [11/Apr/2023:22:11:57 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"host":{"name":"714d9ae40b4a"},"agent":{"version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a","type":"filebeat"},"ecs":{"version":"1.12.0"},"log":{"offset":180,"file":{"path":"/var/log/nginx/access.log"}}}
{"@timestamp":"2023-04-11T22:12:04.664Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-access"},"ecs":{"version":"1.12.0"},"host":{"name":"714d9ae40b4a"},"agent":{"hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a","type":"filebeat","version":"7.17.9"},"log":{"offset":276,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:12:02 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"]}
{"@timestamp":"2023-04-11T22:12:04.664Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"input":{"type":"filestream"},"host":{"name":"9f10de4f0a06"},"agent":{"version":"7.17.9","hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59","name":"9f10de4f0a06","type":"filebeat"},"ecs":{"version":"1.12.0"},"log":{"offset":180,"file":{"path":"/var/log/nginx/access.log"}},"message":"10.89.2.6 - - [11/Apr/2023:22:11:59 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"fields":{"doc":"app-filebeat-nginx-access"}}
{"@timestamp":"2023-04-11T22:12:04.664Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"agent":{"version":"7.17.9","hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59","name":"9f10de4f0a06","type":"filebeat"},"ecs":{"version":"1.12.0"},"host":{"name":"9f10de4f0a06"},"log":{"file":{"path":"/var/log/nginx/access.log"},"offset":276},"message":"10.89.2.6 - - [11/Apr/2023:22:12:04 +0000] \"GET /nopath HTTP/1.0\" 404 153 \"-\" \"curl/7.79.1\" \"-\"","tags":["nginx-access"],"fields":{"doc":"app-filebeat-nginx-access"},"input":{"type":"filestream"}}
.. 생략 ..

$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app-filebeat-nginx-error --from-beginning
{"@timestamp":"2023-04-11T22:10:46.645Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"log":{"offset":381,"file":{"path":"/var/log/nginx/error.log"}},"message":"2023/04/11 22:10:46 [notice] 4#4: start worker process 5","tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"ecs":{"version":"1.12.0"},"host":{"name":"714d9ae40b4a"},"agent":{"name":"714d9ae40b4a","type":"filebeat","version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed"}}
{"@timestamp":"2023-04-11T22:10:46.646Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"message":"2023/04/11 22:10:46 [notice] 4#4: start worker process 6","tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"ecs":{"version":"1.12.0"},"host":{"name":"714d9ae40b4a"},"agent":{"type":"filebeat","version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a"},"log":{"offset":438,"file":{"path":"/var/log/nginx/error.log"}}}
{"@timestamp":"2023-04-11T22:10:46.643Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"ecs":{"version":"1.12.0"},"host":{"name":"9f10de4f0a06"},"agent":{"name":"9f10de4f0a06","type":"filebeat","version":"7.17.9","hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59"},"message":"2023/04/11 22:10:46 [notice] 4#4: start worker process 5","log":{"offset":381,"file":{"path":"/var/log/nginx/error.log"}}}
{"@timestamp":"2023-04-11T22:10:46.644Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"message":"2023/04/11 22:10:46 [notice] 4#4: start worker process 5","tags":["nginx-error"],"fields":{"doc":"app-filebeat-nginx-error"},"input":{"type":"filestream"},"ecs":{"version":"1.12.0"},"host":{"name":"053dd9dc5419"},"agent":{"hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c","id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419","type":"filebeat","version":"7.17.9"},"log":{"offset":381,"file":{"path":"/var/log/nginx/error.log"}}}

.. 생략 ..

{"@timestamp":"2023-04-11T22:12:00.657Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"host":{"name":"9f10de4f0a06"},"agent":{"hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59","name":"9f10de4f0a06","type":"filebeat","version":"7.17.9"},"ecs":{"version":"1.12.0"},"log":{"file":{"path":"/var/log/nginx/error.log"},"offset":724},"message":"2023/04/11 22:11:59 [error] 5#5: *3 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\""}
{"@timestamp":"2023-04-11T22:12:00.659Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"fields":{"doc":"app-filebeat-nginx-error"},"host":{"name":"714d9ae40b4a"},"agent":{"hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a","type":"filebeat","version":"7.17.9"},"ecs":{"version":"1.12.0"},"log":{"offset":724,"file":{"path":"/var/log/nginx/error.log"}},"message":"2023/04/11 22:11:57 [error] 5#5: *3 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\"","tags":["nginx-error"],"input":{"type":"filestream"}}
{"@timestamp":"2023-04-11T22:12:00.658Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"host":{"name":"053dd9dc5419"},"agent":{"ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c","id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419","type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419"},"ecs":{"version":"1.12.0"},"log":{"file":{"path":"/var/log/nginx/error.log"},"offset":724},"message":"2023/04/11 22:11:55 [error] 5#5: *3 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\""}
{"@timestamp":"2023-04-11T22:12:02.660Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"host":{"name":"714d9ae40b4a"},"agent":{"type":"filebeat","version":"7.17.9","hostname":"714d9ae40b4a","ephemeral_id":"aa56aeec-b0ee-4a6b-939a-836101b7bbff","id":"b1a69528-67da-4872-95cc-bc9001ca93ed","name":"714d9ae40b4a"},"log":{"offset":920,"file":{"path":"/var/log/nginx/error.log"}},"message":"2023/04/11 22:12:02 [error] 5#5: *4 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\"","tags":["nginx-error"],"fields":{"doc":"app-filebeat-nginx-error"},"input":{"type":"filestream"},"ecs":{"version":"1.12.0"}}
{"@timestamp":"2023-04-11T22:12:02.659Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"message":"2023/04/11 22:12:00 [error] 5#5: *4 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\"","tags":["nginx-error"],"input":{"type":"filestream"},"fields":{"doc":"app-filebeat-nginx-error"},"ecs":{"version":"1.12.0"},"host":{"name":"053dd9dc5419"},"agent":{"id":"3db14b2e-4831-43c4-926d-97136d781be1","name":"053dd9dc5419","type":"filebeat","version":"7.17.9","hostname":"053dd9dc5419","ephemeral_id":"5eada860-4c61-4dfc-b515-cb9698da664c"},"log":{"offset":920,"file":{"path":"/var/log/nginx/error.log"}}}
{"@timestamp":"2023-04-11T22:12:06.660Z","@metadata":{"beat":"filebeat","type":"_doc","version":"7.17.9"},"fields":{"doc":"app-filebeat-nginx-error"},"ecs":{"version":"1.12.0"},"host":{"name":"9f10de4f0a06"},"agent":{"name":"9f10de4f0a06","type":"filebeat","version":"7.17.9","hostname":"9f10de4f0a06","ephemeral_id":"372ac361-e43b-41a2-aed3-7c598967fa4e","id":"e3d6e0a4-0f03-4304-a0a3-226133d5ae59"},"log":{"offset":920,"file":{"path":"/var/log/nginx/error.log"}},"message":"2023/04/11 22:12:04 [error] 5#5: *4 open() \"/usr/share/nginx/html/nopath\" failed (2: No such file or directory), client: 10.89.2.6, server: localhost, request: \"GET /nopath HTTP/1.0\", host: \"app\"","tags":["nginx-error"],"input":{"type":"filestream"}}

```  

그리고 브라우저에서 `Kibana` 주소인 `localhost:5601` 로 접속하고 아래와 같이 기본 설정을 마치면 `app-fiebeat` 컨테이너의 
`Nginx` 에서 생성된 로그가 `Filebest`, `Kafka`, `Logstash`, `Elasticsearch` 를 거쳐 `Kibana` 에서 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-2.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-3.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-4.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-kafka-5.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-kafka-6.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-6.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-kafka-7.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-kafka-8.png)

---  
## Reference
[Deploy Kafka + Filebeat + ELK - Docker Edition - Part 1](https://dev.to/dhingrachief/deploy-kafka-filebeat-elk-docker-edition-part-1-3m77)  
[Deploy Kafka + Filebeat + ELK - Docker Edition - Part 2](https://dev.to/dhingrachief/deploy-kafka-filebeat-elk-docker-edition-part-2-hpj)  
