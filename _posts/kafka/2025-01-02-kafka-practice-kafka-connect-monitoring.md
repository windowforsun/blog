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
