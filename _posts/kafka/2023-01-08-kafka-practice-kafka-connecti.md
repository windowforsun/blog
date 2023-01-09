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
    - Kafka Connect
toc: true
use_math: true
---  

## Kafka Connect
`Kafka Connect` 는 카프카 오픈소스로 데이터 파이프라인 생성 시 반복 작업을 최소화 하고 효율적인 전송을 도와주는 애플리케이션이다. 
`Kafka Connect` 가 없으면 파이프라인을 생성할 때 마다 프로듀서, 컨슈머 애플리케이션을 매번 개발 및 배포 운영 해줘야 한다. 
이러한 불편함은 `Kafka Connect` 를 사용해서 파이프라인의 작업을 템플릿으로 만들어 `Connector` 형태로 실행하는 방식으로 반복 작업을 중인다.  

`Connector` 는 프로듀서 역할을 하는 `Source Connector(소스 커넥터)` 와 컨슈머 역할을 하는 `Sink Connector(싱크 커넥터)` 로 나뉜다. 
이러한 `Connector` 를 통해 파일, `MySQL`, `S3`, `MongoDB` 등에 데이터를 가져오거나 데이터를 저장 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-1.drawio.png)  

`Kafka 2.6` 부터는 클러스터 간 토픽 미러링을 지원하는 `미러메이커2` 와 파일(싱크/소스) 를 기본 커넥터 플러그인으로 제공한다. 
이외 추가적인 커넥터는 플러그인 형태로 `jar` 파일을 추가할 수 있는데, 직접 구현할 수도 있고, 
오픈소스로 공개된 허브에서 검색해서 사용할 수도 있다. ([허브](https://www.confluent.io/hub/))
사용에서 사용할 때는 라이센스 범위가 다를 수 있으므로 확인 후 사용해야 한다.  

커넥트를 생성하면 커넥트는 내부에 커넥터와 태스트를 생성하는데, 커넥터는 태스크를 관리하는 역할을 한다. 
태스크는 커넥터에 종속되면서 데이터 처리를 담당한다. 
그러므로 실질적으로 데이터 처리가 잘 되고 있는 지 확인 하고 싶다면 태스트 상태 확인이 필요하다. 
그리고 커넥터를 사용해서 파이프라인을 생성할 때 `Converter` 와 `Transform` 기능을 옵션으로 추가해서 더 다양한 처리를 수행할 수 있다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-2.drawio.png)

`Converter` 는 데이터 처리 전 스키마 변경을 도와주는데 아래와 같은 종류가 있고 필요시 직접 구현할 수도 있다. 

- `JsonConverter`
- `StringConverter`
- `ByteArrayConverter`

`Transform` 은 데이처를 처리할 때 메시지 단위로 데이터를 변환 하는 용도로 사용 할 수 있다. 
`JSON` 데이터에 특정 키를 삭제 한다던가, 추가하는 등과 같은 작업을 의미한다. 
기본으로 제공하는 `Transform` 은 아래와 같다. 

- `Cast`
- `Drop`
- `ExtractField`

커넥트를 실행하는 방법으로는 아래와 같은 2가지 방법이 있다. 

- `Standalone mode kafka connect` (단일 모드 커넥트)
- `Distributed mode kafka connect` (분산 모드 커넥트)

`Kafka Connect` 실행을 위해 `Kafka` 를 `docker-compose` 를 사용해서 구성한다. 
파일 내용은 아래와 같다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"

  kafka:
    container_name: myKafka
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```  

`docker` 로 실행한 `Kafka` 컨테이너에서 `Kafka Connect` 관련 파일의 경로는 아래와 같이 `/opt/kafka/config` 에서 확인 할 수 있다. 

```bash
$ docker exec -it myKafka ls /opt/kafka/config
connect-console-sink.properties    consumer.properties
connect-console-source.properties  kraft
connect-distributed.properties     log4j.properties
connect-file-sink.properties       producer.properties
connect-file-source.properties     server.properties
connect-log4j.properties           tools-log4j.properties
connect-mirror-maker.properties    trogdor.conf
connect-standalone.properties      zookeeper.properties
```  

### 단일 모드 커넥트
`standalone mode kafka connect` 는 1개 프로세스만 실행되는 커텍트이다. 
즉 단일 프로세스로만 구동 시키기 때문에 고가용성에 대한 대응이 되지 않아, 
단일 장애점(SPOF)이 될 수 있다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-3.drawio.png)  


### 분산 모드 커넥트

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-4.drawio.png)  


---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
