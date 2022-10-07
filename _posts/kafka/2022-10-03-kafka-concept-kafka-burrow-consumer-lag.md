--- 
layout: single
classes: wide
title: "[Kafka] Burrow, Consumer Lag 모니터링"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 에서 발생하는 Consumer Lag 를 모니터링 할 수 있는 Burrow 와 구성 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
  - Kafka
  - Concept
  - Kafka
  - Kubernetes
  - Burrow
  - Consumer Lag
toc: true
use_math: true
---  

## Consumer Lag
컨슈머 랙(`Consumer Lag`)이란 프로듀서가 데이터를 넣는 속도가 컨슈머가 가져가는 속도보다 빠른 경우, 
프로듀서가 넣은 데이터 오프셋과 컨슈머가 가져간 데이터의 오프셋 차이 또는 토픽의 가장 최신 오프셋과 컨슈머 오프셋의 차이를 의미한다.  

프로듀서가 데이터를 토픽내의 파티션에 추가하면 각 데이터에는 오프셋이라는 숫자가 증가하며 붙게 된다. 
그리고 컨슈머는 파티션의 데이터를 하나씩 읽어올 때, 데이터를 어디까지 읽었는지 표시하기 위해 자신이 마지막에 읽은 데이터의 오프셋을 저장해둔다. 
즉 `Consumer Lag` 이란 위서 설명한 것처럼 프로듀서는 빠르게 많은 데이터를 파티션에 넣어 오프셋값이 크게 증가 했지만, 
컨슈머는 느리게 읽고 있어서 마지막으로 읽은 오프셋의 값이 작고 읽어야 하는 데이터가 너무 많이 남아 있다는 의미이다. 
이는 다르게 말하면 데이터 지연이 발생한다 던가 컨슈머가 정상적으로 동작을 해주지 못하고 있는 상황이라고 추측 할 수 있다.  

프로듀서와 컨슈머의 오프셋은 파티션을 기준으로 한 값이기 때문에 
토픽 하나에 여러 파티션이 존재하는 상황이라면 `Lag` 은 여러개가 존재하 수 있다. 
예를 들어 파티션이 2개이고 컨슈머 그룹이 1개
토픽에 여러 파티션이 존재하는 경우 `Lag` 은 여러개 존재 할 수 있다. 
예를들어 파티션이 2개이고 하나의 컨슈머 그룹이 있는 상태라면 2개의 `Lag` 이 발생할 수 있다. 
그리고 이중에서 가장 높은 `Lag` 숫자를 갖는 것을 `records-lag-max` 이라고 한다.  


### Consumer Lag 증상
`Consumer Lag` 은 `Kafka` 를 운영할때 가장 중요한 지표중 하나라고 할 수 있다. 
실질적인 `Lag` 의 크기는 작거나 클 수 있는데 이를 통해 해당하는 토픽의 프로듀서, 컨슈머의 상태를 파악하고 분석하는데 중요한 정보가 될 수 있다.  

`Consumer Lag` 이 증가하고 있다면 아래와 같은 상황을 고려해 볼 수 있다.  
- 컨슈머의 성능에 비해 프로듀서가 순간적으로 많은 메시지를 보내는 경우
- 컨슈머의 `polling` 속도가 느려지는 경우, `polling` 후 메시지 처리 로직
- 네트워크 지연 이슈
- `Kafka broker` 내부 이슈

## Burrow 

[Burrow](https://github.com/linkedin/Burrow) 는 `Linkedin` 에서 만든 `Consumer Lag` 모니터링 오픈소스 툴이다. 
기존 `Kafka client` 를 사용해서 `Consumer Lag` 을 모니터링할 경우 `consumer` 와 `metrics()` 메소드를 사용해서 `lag metric`(`records-lag-max`)를 확인 할 수 있다. 
이는 가장 높은 `Lag` 을 보이는 파티션을 보여주는 지표이기 때문에 다른 파티션들은 어떠한 상태인지 확인이 어렵다. 
그리고 이러한 방법은 `Conumser` 가 정상동작 하고 있다는 가정을 두고 있기 때문에 `Consumer` 에 이상이 발생하는 경우 모니터링까지 이뤄지지 않을 수 있다. 
이러한 이유로 인해 정확한 지연 모니터링을 위해서는 외부 모니터링 시스템이 필요한데 대표적인 것이 바로 `Burrow` 이다.  

위 내용을 `Linkedin` 의 자료를 바탕으로 설명하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/practice-burrow-1.png)

위 그림의 4개 `Consumer` 는 하나의 토픽 메시지를 소비하는 상황이고, 상태를 설명하면 아래와 같다. 

- `Consumer A` : `Lag` 이 점차 감소하고 있다. 
- `Consumer B` : 순간 `Lag` 이 급증했지만 빠르게 회복됐다. 
- `Consumer C` : 일정하면서 약간 지연이 발생하고 있다.
- `Consumer D` : 우상향 지표를 보이고 있지만, 부분적으로 해소되고 있다. 

전체적으로 해당 토픽을 상태를 본다면 대량의 트래픽이 한번에 들어오는 상황은 있지만, 
현재 잘 처리가 되고 있는 상태라고 할 수 있다. 
하지만 만약 `MaxLag` 를 바탕으로 모니터링을 한다면 `Consumer B`, `Consumer D` 에 의해 문제가 있는 상태로 감지 될 수 있다.  

### Burrow Architecture

![그림 1]({{site.baseurl}}/img/kafka/practice-burrow-2.png)

`Burrow` 는 위와 같은 `Architecture` 를 통해 `Consumer status` 를 모니터링하기 위한 가장 최적의 방법을 제공한다.  

- `Clusters` : 모든 파티션에 대해 주기적으로 토픽 목록, `offset` 을 가져오는 `Kafka Client` 실행
- `Consumers` : `Kafka`, `Zoopeer` 로부터 `Consumer Group` 의 정보를 가져온다. 
- `Storage` : `Burrow` 의 모든 정보 저장
- `Notifier` : 일정기준을 넘었을 때 알림(`email`, `http`)
- `HTTP Server` : `Cluster` 와 `Consumer` 정보를 `HTTP API` 를 통해 제공
- `Evaluator` : `Consumer Group` 의 상태 평가, 평가기준은 `Consumer lag evaluation rules` 를 따른다. 


### Burrow 구성
`Burrow` 는 `Consumer Lag` 을 모니터링 할 수 있는 외부 모니터링 시스템이다. 
그러므로 기존 구동중인 `Kafka Cluster` 에 추가 설치는 필요하지 않고, 별도로 구성한 후 `Kafka`, `Zookeeper` 만 연걸해주면 된다.  

간단한 환경 구성을 위해 `Docker`, `docker-compose` 를 사용했고 예제는 [Burrow](https://github.com/linkedin/Burrow)
에 있는 `docker-compose.yaml` 을 사용했다.  

구성한 방식은 아주 간단하다. 
먼저 `Burrow` 저장소를 클론받고 애서 `docker-compose` 명령어만 실행해 주면 `Kafka`, `Zookeeper`, `Burrow` 가 다 구성된다.  


```bash
$ git clone https://github.com/linkedin/Burrow.git

$ cd Burrow

$ docker-compose up -d --build

$ curl localhost:8000/burrow/admin
GOOD

$ curl localhost:8000/v3/kafka/local/topic | jq
{
  "error": false,
  "message": "topic list returned",
  "topics": [
    "test-topic3",
    "test-topic2",
    "__consumer_offsets",
    "test-topic"
  ],
  "request": {
    "url": "/v3/kafka/local/topic",
    "host": "419d8a7a5881"
  }
}
```  

만약 `docker-compose` 명령어 실행 단계에서 볼륨 마운트 에러가 발생한다면 경로에 대한 권한 확인 진행이 필요하고, 
볼륨마운트가 필요하지 않는 경우는 주석처리 하더라도 간단한 사용성 테스트하는 데는 문제 없다.  

구성할떄 사용한 `docker-compose.yaml` 파일 내용을 보면 아래와 같다.  

```yaml
version: "2"
services:
  burrow:
    build: .
    volumes:
      - ${PWD}/docker-config:/etc/burrow/
      - ${PWD}/tmp:/var/tmp/burrow
    ports:
		# burrow 포트
      - 8000:8000
    depends_on:
      - zookeeper
      - kafka
    restart: always

  zookeeper:
    image: wurstmeister/zookeeper
    ports:
		# zookeeper 포트
      - 2181:2181

  kafka:
    image: wurstmeister/kafka
    ports:
		# kafka 포트
      - 9092:9092
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181/local
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_CREATE_TOPICS: "test-topic:2:1,test-topic2:1:1,test-topic3:1:1"
```  

그리고 사용한 `Burrow` 설정 파일인 `docker-config/burrow.toml` 내용은 아래와 같다.  

```
[zookeeper]
servers=[ "zookeeper:2181" ]
timeout=6
root-path="/burrow"

[client-profile.profile]
kafka-version="0.11.0"
client-id="docker-client"

[cluster.local]
client-profile="profile"
class-name="kafka"
servers=[ "kafka:9092" ]
topic-refresh=60
offset-refresh=30
groups-reaper-refresh=30

[consumer.local]
class-name="kafka"
cluster="local"
servers=[ "kafka:9092" ]
group-denylist="^(console-consumer-|python-kafka-consumer-).*$"
group-allowlist=""

[consumer.local_zk]
class-name="kafka_zk"
cluster="local"
servers=[ "zookeeper:2181" ]
zookeeper-path="/local"
zookeeper-timeout=30
group-denylist="^(console-consumer-|python-kafka-consumer-).*$"
group-allowlist=""

[httpserver.default]
address=":8000"
```  

`docker-compose.yaml` 에 정의된 `kafka`, `zookeeper` 의 서비스이름과 포트를 사용해서 `burrow.toml` 설정 파일을 잘 구성만 해주면 
`Burrow` 구성은 완료 된다.  



---
## Reference
[Burrow - Kafka Consumer Lag Checking](https://github.com/linkedin/Burrow)  
[Burrow: Kafka Consumer Monitoring Reinvented](https://engineering.linkedin.com/apache-kafka/burrow-kafka-consumer-monitoring-reinvented)  
[Consumer Lag Evaluation Rules](https://github.com/linkedin/Burrow/wiki/Consumer-Lag-Evaluation-Rules)  
[Setting Up Burrow & Burrow Dashboard For Kafka Lag Monitoring](https://www.lionbloggertech.com/setting-up-burrow-dashboard-for-kafka-lag-monitoring/)  
