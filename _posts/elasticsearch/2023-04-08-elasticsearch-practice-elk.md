--- 
layout: single
classes: wide
title: "[Elasticsearch 실습] ELK with Filebeat 로그 수집"
header:
  overlay_image: /img/elasticsearch-bg.png
excerpt: 'ELK 스택과 Filebeat 을 사용해서 로그를 수집하는 방법과 구성에 대해 알아보자'
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
toc: true
use_math: true
---  

## ELK Stack
`ELK Stack` 이란 `Elasticsearch`, `Logstash`, `Kibana` 의 조합을 통칭하는 기술 스택을 의미한다. 
`Elasticsearch` 는 `JSON` 기반 분산 오픈소스 검색 및 분석 엔진이다. 
`Logstash` 는 여러 소스에서 동시에 데이터를 수집하여 변환 수행 후, 
`Elasticsearch` 같은 `stash` 로 전송하는 서버사이드 데이터 파이프라인이다. 
그리고 `Kibana` 는 `Elasticsearch` 에서 차트와 그래프를 이용해 데이터를 시각화 할 수 있게 해주는 툴이다. 
마지막으로 `ELK` 에서 보편적으로 추가되는 것이 바로 `Filebeat` 이다. 
여기서 `Filebeat` 은 서버에서 에이전트로 설치 되어 다양한 유형의 데이터를 `Elasticsearch` 혹은 `Logstash` 로 전송하는 오픈 소스 데이터 발송자를 의미힌다.  

`Logstash` 와 `Filebeat` 모두 데이터를 `Elasticsearch` 로 전송한다는 목적에서는 동일한 역할을 하지만 아래와 같은 차이가 있다.  

- `Logstash` : 비교적 많은 자원을 사용해서 다를 수 있는 `input`, `output` 유형(종류)가 많고, `filter` 를 사용해서 로그(데이터)를 분석하기 쉽게끔 구조화 된 형식으로 변환 가능하다. 
- `Filebeat` : 가벼운 대신 가능한 `input`, `output` 종류가 한정적이다. 설정에 지정된 로그 데이터를 바라보는 하나 이상의 `input` 을 가진다. 지정도니 로그 파일에서 이벤트(데이터 변경)가 바생 할때 마다 데이터 수확을 시작한다. 

구성을 하나더 추가한다면 `Kafka` 가 중간에 들어갈 수 있지만, 
이번 포스트에서는 `Kafka` 는 제외한 `ELK Stack` 에 대해서 간단한 구성 방법을 알아본다. 
여기서 `Kafka` 역할에 대해서만 간단하게 알아보면, 중간 버퍼 역할을 한다고 할 수 있다. 
순간적으로 급증하는 데이터양에 대한 버퍼도 될 수 있고, 특정 시스템이 다운 됐을 때 로그의 버퍼 역할도 할 수 있다.  


### ELK Example
구성할 `ELK` 의 간단한 예시는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-1.drawio.png)

`Filebeat` 이 호스트에 설치된 간단한 애플리케이션(`Nginx`)이 생산하는 `Access`, `Error` 로그를 `Logstash` 에 전송하고, 
`Logstash` 는 이를 파싱해서 다시 `Elasticsearch` 에 전송하면 최종적으로 `Kibana` 를 통해 로그를 확인 하는 예제이다.  

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
│       └── nginx
└── nginx-lb
    └── nginx.conf

```

#### docker-compose
구성하고자 하는 `ELK Stack` 의 전체 구성을 담고 있는 `docker-compose.yaml` 파일 내용은 아래와 같다.  

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
      - "ELASTIC_IP=es-single"
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

  nginx-lb:
    image: nginx:1.21.1
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

`Logstash` 를 사용할 때 호스트 이름이나 컨테이너 이름에 `_` 나 `.` 가 들어가면 에러가 발생할 수 있으므로 주의해야 한다.  

#### Logstash
`Logstash` 설정은 `/usr/share/logstash/pipeline/` 하위에 `logstash` 로 시작하는 파일을 두면 된다. 
그리고 `pipeline/patterns` 디렉토리에 `logstash` 설정에서 사용할 `grok` 정규식 사전 파일을 두면 파상할 로그의 
패턴을 미리 정의해서 사용 할 수 있다.  

로그의 패턴을 미리 정의하는 `pipeline/patterns/nginx` 파일 내용은 아래와 같다.  

```
# nginx
# nginx
NGINXACCESS %{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \[%{HTTPDATE:[nginx][access][time]}\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\"
NGINXERROR %{DATA:[nginx][error][time]} \[%{DATA:[nginx][error][level]}\] %{NUMBER:[nginx][error][pid]}#%{NUMBER:[nginx][error][tid]}: (\*%{NUMBER:[nginx][error][connection_id]} )?%{GREEDYDATA:[nginx][error][message]}
```  

`Logstash` 의 설정을 담고 있는 `logstash.conf` 파일 내용은 아래와 같다.  

```
# filebeat -> logstash 통신을 5044 로 수행한다. 
input {
    beats {
        port => 5044
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
  elasticsearch {
    action => "index"
    # 환경변수 ELASTIC_IP 를 사용해서 로그를 전송한다. 
    hosts => ["${ELASTIC_IP:=localhost}:9200"]
    manage_template => false
    # 일자 단위로 인덱스를 생성한다. 
    index => "app-filebeat-%{+YYYY.MM.dd}"
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

```yaml
filebeat.inputs:
  - type: filestream
    enabled: true
    paths:
      - /var/log/nginx/access.log
    tags: ["nginx-access"]

  - type: filestream
    enabled: true
    paths:
      - /var/log/nginx/error.log
    tags: ["nginx-error"]

output.logstash:
  hosts: ["${LOGSTASH_IP:=localhost}:5044"]

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
[+] Building 261.0s (25/25) FINISHED                                                                                    
 => [internal] booting buildkit                                                                                    4.3s
 => => pulling image moby/buildkit:buildx-stable-1                                                                 3.3s
 => => creating container buildx_buildkit_default                                                                  1.0s
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 1.47kB                                                                             0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load metadata for docker.io/library/nginx:1.21.3                                                    2.4s
 => [ 1/18] FROM docker.io/library/nginx:1.21.3@sha256:644a70516a26004c97d0d85c7fe1d0c3a67ea8ab7ddf4aff193d9f3016  0.0s
 => => resolve docker.io/library/nginx:1.21.3@sha256:644a70516a26004c97d0d85c7fe1d0c3a67ea8ab7ddf4aff193d9f301670  0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 64B                                                                                   0.0s
 => CACHED [ 2/18] WORKDIR /root                                                                                   0.0s
 => CACHED [ 3/18] RUN mkdir -p /usr/share/nginx/html/js                                                           0.0s
 => CACHED [ 4/18] RUN cp /etc/apt/sources.list sources.list_backup                                                0.0s
 => CACHED [ 5/18] RUN sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list                   0.0s
 => CACHED [ 6/18] RUN sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list                  0.0s
 => CACHED [ 7/18] RUN apt-get update && apt-get upgrade -y                                                        0.0s
 => [ 8/18] RUN apt-get -y update                                                                                 14.9s
 => [ 9/18] RUN apt-get install wget gnupg -y                                                                    216.5s
 => [10/18] RUN rm -f /var/log/nginx/access.log                                                                    0.1s
 => [11/18] RUN rm -f /var/log/nginx/error.log                                                                     0.1s
 => [12/18] RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -                      0.6s
 => [13/18] RUN apt-get install apt-transport-https -y                                                             6.4s
 => [14/18] RUN echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | tee -a /etc/apt/source  0.1s
 => [15/18] RUN apt-get update && apt-get install filebeat -y                                                     10.1s
 => [16/18] COPY ./filebeat.yaml /etc/filebeat/filebeat.yml                                                        0.0s
 => [17/18] RUN update-rc.d filebeat defaults 90 10                                                                0.1s
 => [18/18] COPY ./script.sh /root/                                                                                0.0s
 => exporting to docker image format                                                                               5.2s
 => => exporting layers                                                                                            1.8s
 => => exporting manifest sha256:45cff930475c94e70f2c8e10d394bd894309679f992719928232d766bd73d3ac                  0.0s
 => => exporting config sha256:93b17b50a5301d919d27cc61c8e0fc0b702a1cda20697bcdc3627d396665ff9b                    0.0s
 => => sending tarball                                                                                             3.5s
 => importing to docker                                                                                            0.0s
[+] Running 7/7
 ⠿ Container es-single             Created                                                                         0.0s
 ⠿ Container exam1-nginx-lb-1      Created                                                                         0.0s
 ⠿ Container kibana                Created                                                                         0.0s
 ⠿ Container logstash              Created                                                                         0.0s
 ⠿ Container exam1-app-filebeat-3  Created                                                                         0.0s
 ⠿ Container exam1-app-filebeat-1  Created                                                                         0.0s
 ⠿ Container exam1-app-filebeat-2  Created                                                                         0.0s
Attaching to es-single, exam1-app-filebeat-1, exam1-app-filebeat-2, exam1-app-filebeat-3, exam1-nginx-lb-1, kibana, logstash

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

그리고 브라우저에서 `Kibana` 주소인 `localhost:5601` 로 접속하고 아래와 같이 기본 설정을 마치면 `app-fiebeat` 컨테이너의 
`Nginx` 에서 생성된 로그가 `Filebest`, `Logstash`, `Elasticsearch` 를 거쳐 `Kibana` 에서 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-2.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-3.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-4.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-5.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-6.png)

![그림 1]({{site.baseurl}}/img/elasticsearch/elasticsearch-elk-7.png)

---  
## Reference
[docker container 환경에서 ELK(Elasticsearch, Logstash, Kibana)로 로깅해보기](https://a3magic3pocket.github.io/posts/elk-container-example/#%EC%8B%A4%ED%97%98-%EC%86%8C%EA%B0%90)  
