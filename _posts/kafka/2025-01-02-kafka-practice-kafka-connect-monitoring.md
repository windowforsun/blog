--- 
layout: single
classes: wide
title: "[Kafka] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
toc: true
use_math: true
---  

## Kafka Connect Monitoring
`Kafka Connect` 를 모니터링 하는 것은 `Kafka` 기반 데이터 통합 파이프라인의 
안전성, 성능, 신뢰성을 보장하는데 필요한 작업이다. 
모니터링을 통해 `Kafka Connector` 들의 성능과 상태를 추적하고 관리할 수 있다. 
이를 위해 `Kafka Connect` 에서 제공하는 `JMX Metrics` 를 수집해 이를 식각화 하는 환경을 구축하는 방법에 대해 알아보고자 한다.  

`Kafka Connect` 의 모니터링을 구축하기 위해 `JMX` 로 메트릭을 `Export` 하고 이를 `Prometheus` 를 통해 수집한 후 `Grafana` 를 통해 
모니터링 대시보드를 구축한다. 
이를 도식화해 그리면 아래와 같다.  

구축의 상세 내용이 있는 예제는 [여기]()
에서 확인 할 수 있다.  

### Kafka Connect JMX Metrics
`Kafka Connect` 에서는 `JMX` 를 통해 다양한 메트릭을 제공한다. 
관련해 제공되는 메트릭의 종류는 [여기](https://docs.confluent.io/platform/current/connect/monitoring.html#use-jmx-to-monitor-kconnect)
에서 확인 할 수 있다.  

`Kafka Connect` 의 `JMX` 메트릭을 수집하기 위해서는 `JMX Exporter` 설치가 필요하다. 
`Docker` 환경에서 사용하기 위해 `Dockerfile` 에 아래와 같이 설치 스크립트를 추가한다.  

```dockerfile
FROM confluentinc/cp-kafka-connect-base:7.0.10

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components,/usr/share/filestream-connectors" \
    CUSTOM_SMT_PATH="/usr/share/java/custom-smt" \
    CUSTOM_CONNECTOR_MYSQL_PATH="/usr/share/java/custom-connector-mysql"
ENV TZ=Asia/Seoul
ENV KAFKA_OPTS="-Duser.timezone=Asia/Seoul"

ARG CONNECT_TRANSFORM_VERSION=1.4.4

# Download Using confluent-hub
RUN confluent-hub install --no-prompt confluentinc/connect-transforms:$CONNECT_TRANSFORM_VERSION

user root

RUN chmod -R 755 /usr/share/java

# Install tzdata package
RUN yum update -y && yum install -y tzdata

# Set the timezone to Asia/Seoul
RUN ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone

ENV TZ=Asia/Seoul
ENV KAFKA_OPTS="-Duser.timezone=Asia/Seoul"

# Set JVM timezone through JAVA_OPTS
ENV JAVA_OPTS="-Duser.timezone=Asia/Seoul"

# Download JMX Exporter
ADD https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.16.1/jmx_prometheus_javaagent-0.16.1.jar /opt/jmx_prometheus_javaagent.jar

# Add JMX Exporter configuration file
COPY config.yml /opt/config.yml

# Expose JMX port
EXPOSE 5556

CMD ["/bin/bash", "-c", "/usr/local/bin/init-connect.sh & /etc/confluent/docker/run"]
```  

그리고 `config.yaml` 파일을 아래와 같이 작성해 `JMX Exporter` 에서 메트릭을 공개할 룰을 정의한다. 
아래는 전체 메트릭을 공개하는 내용이다. 

```yaml
# config.yml
rules:
  - pattern: ".*"
```  

위 설정 내용에서 알 수 있듯이 `JMX` 를 수집할 때 사용할 포트는 `5556` 이다.  

### Prometheus
`Kafka Connect` 에서 `JMX Exporter` 를 사용해 메트릭을 수집할 수 있도록 공개했다면, 
메트릭을 수집하는 과정이 필요하다. 
이때 사용할 것이 바로 `Prometheus` 로 `Prometheus` 의 설정 파일인 `prometheus.yml` 파일에 
아래와 같이 `Kafka Connect` 에서 메트릭을 수집하도록 추가한다.  

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']


  - job_name: 'grafana'
    scrape_interval: 5s
    static_configs:
      - targets: ['grafana:3000']

  - job_name: 'kafka-connect'
    static_configs:
      - targets: [ 'exam-connect:5556' ]
```  



### 데모 환경 구성
이제 `docker-compose.yaml` 파일을 통해 테스트에 필요한 전체 구성을 작성하고 실행한다. 
`docker-compose.yaml` 파일 내용은 아래와 같이 `kafka`, `zookeeper`, `kafka-connect`, `prometheus`, `grafana` 로 구성돼있다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      - exam-net

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
      - exam-net

  exam-connect:
    container_name: exam-connect
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8083:8083"
      - "5556:5556"
    command: sh -c "
      echo 'waiting 10s' &&
      sleep 10 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'exam-connect'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'file-source-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'file-source-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'file-source-status'
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_REST_ADVERTISED_HOST_NAME: exam-connect
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
      KAFKA_JMX_OPTS: "-javaagent:/opt/jmx_prometheus_javaagent.jar=5556:/opt/config.yml -Dcom.sun.management.jmxremote.authenticate=false"
      TZ: 'Asia/Seoul'
      JAVA_OPTS: "-Duser.timezone=Asia/Seoul"
    volumes:
      - ./data:/data
      - ./config.yml:/opt/config.yml
    networks:
      - exam-net

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - exam-net

  # id : admin, pw : admin
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - exam-net

networks:
  exam-net:
```  

`Kafka Connect` 쪽 `environment` 를 보면 `KAFKA_JMX_OPTS` 환경변수를 통해 
다운로드 받은 `JMX Exporter` 실행 파일과 사용할 포트 그리고 설정 파일인 `config.yml` 지정했다.  

```yaml
KAFKA_JMX_OPTS: "-javaagent:/opt/jmx_prometheus_javaagent.jar=5556:/opt/config.yml -Dcom.sun.management.jmxremote.authenticate=false"
```

모든 구성은 `docker-compose up --build` 명령으로 실행할 수 있다.  

```bash
$ docker-compose up --build                                                   0.0s
[+] Running 6/0
 ✔ Container docker-grafana-1                                                                                                                               Created0.0s 
 ✔ Container myZookeeper                                                                                                                                    Created0.0s 
 ✔ Container exam-connect                                                                                                                                   Created0.0s 
 ✔ Container docker-prometheus-1                                                                                                                            Created0.0s 
 ! zookeeper The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested 0.0s 
 ✔ Container myKafka                                                                                                                                        Created0.0s 
Attaching to docker-grafana-1, docker-prometheus-1, exam-connect, myKafka, myZookeeper

...

```  

그리고 데모 테스트를 위해 `FileStreamSourceConnector` 를 사용하는 `Kafka Connector` 를 아래 요청으로 등록한다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
  "name": "exam-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/file-source-input.txt",
    "topic" : "file-source-topic"
  }
}' \
http://localhost:8083/connectors | jq

{
  "name": "exam-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/file-source-input.txt",
    "topic": "file-source-topic",
    "name": "exam-file-source"
  },
  "tasks": [],
  "type": "source"
}
```  

그리고 `Kafka` 에서 `file-source-topic` 컨슘하면 파일의 내용이 토픽에 추가가 된 것으로 
`Kafka Connect` 및 `Kafka Connector` 실행이 정상적으로 된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    111
NO_HEADERS      null    222
```  
