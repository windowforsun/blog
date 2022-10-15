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
```  

만약 `docker-compose` 명령어 실행 단계에서 볼륨 마운트 에러가 발생한다면 경로에 대한 권한 확인 진행이 필요하고, 
볼륨마운트가 필요하지 않는 경우는 주석처리 하더라도 간단한 사용성 테스트하는 데는 문제 없다.  

구성할떄 사용할 `docker-compose.yaml` 파일 내용을 보면 아래와 같다.  

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

그리고 사용할 `Burrow` 설정 파일인 `docker-config/burrow.toml` 내용은 아래와 같다.  

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

이제 `docker-compose` 명령어로 전체 구성을 실행 시킬 수 있다.  

```bash
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

### Burrow HTTP Endpoint
`Burrow` 는 간단한 요청을 통해 `Burrow`, `Kafka`, `Zookeeper` 클러스터에 대한 정보를 편리한 방법으로 조회해올 수 있다. 
`HTTP` 요청을 사용하고 응답 형식은 `JSON` 이다.  

만약 요청과정에서 에러가 발생한다면 `status code` 에 400 ~ 500 사이의 응답코드를 응답하고,
응답된 `JSON` 에서 더 자세한 에러를 확인 할 수 있다.

```json
{
  "error": true,
  "message": "Detailed error message",
  "request": {
    "uri": "/path/to/request",
    "host": "responding.host.example.com",
  }
}
```  

아래 테이블은 `Burrow` 에서 제공하는 모든 엔드포인트는 아래와 같다.  

Request| Method |URL Format
---|---|---
Healthcheck| GET |/burrow/admin
List Clusters| GET |/v3/kafka
Kafka Cluster Detail| GET |/v3/kafka/(cluster)
List Consumers| GET |/v3/kafka/(cluster)/consumer
List Cluster Topics| GET |/v3/kafka/(cluster)/topic
Get Consumer Detail| GET |/v3/kafka/(cluster)/consumer/(group)
Consumer Group Status| GET |/v3/kafka/(cluster)/consumer/(group)/status /v3/kafka/(cluster)/consumer/(group)/lag
Remove Consumer Group| DELETE |/v3/kafka/(cluster)/consumer/(group)
Get Topic Detail| GET |/v3/kafka/(cluster)/topic/(topic)
Get General Config| GET |/v3/config
List Cluster Modules| GET |/v3/config/cluster
Get Cluster Module Config| GET |/v3/config/cluster/(name)
List Consumer Modules| GET |/v3/config/consumer
Get Consumer Module Config| GET |/v3/config/consumer/(name)
List Notifier Modules| GET |/v3/config/notifier
Get Notifier Module Config| GET |/v3/config/notifier/(name)
List Evaluator Modules| GET |/v3/config/evaluator
Get Evaluator Module Config| GET |/v3/config/evaluator/(name)
List Storage Modules| GET |/v3/config/storage
Get Storage Module Config| GET |/v3/config/storage/(name)
Get Log Level| GET |/v3/admin/loglevel
Set Log Level| POST |/v3/admin/loglevel

아래는 자주사용되는 몇개의 엔드포인트에 대한 상세 설명 및 요청 및 응답 예시이다.  

#### Healthcheck
`Burrow` 의 `Healthcheck` 결과를 응답한다. 

`GET /burrow/admin`

```bash
http://localhost:8000/burrow/admin

HTTP/1.1 200 OK
Content-Length: 4
Content-Type: text/plain; charset=utf-8

GOOD
```  

#### List Cluster
`Burrow` 에 등록된 `Kafka cluster` 리스트를 응답한다. 

`GET /v3/kafka`

```bash
http://localhost:8000/v3/kafka

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 122

{
  "error": false,
  "message": "cluster list returned",
  "clusters": [
    "local"
  ],
  "request": {
    "url": "/v3/kafka",
    "host": "f2906c456102"
  }
}
```  

#### Kafka Cluster Detail
요청에 명시한 단일 클러스터에 대한 상세 정보를 응답한다. 
응답에는 `Burrow` 에 등록된 `Kafka broker` 와 `zookeeper` 와 관련된 정보가 포함된다.  

`GET /v3/kafka/(cluster)`

```bash
http://localhost:8000/v3/kafka/local

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 324

{
  "error": false,
  "message": "cluster module detail returned",
  "module": {
    "class-name": "kafka",
    "servers": [
      "kafka:9092"
    ],
    "client-profile": {
      "name": "profile",
      "client-id": "docker-client",
      "kafka-version": "0.11.0",
      "tls": null,
      "sasl": null
    },
    "topic-refresh": 60,
    "offset-refresh": 30
  },
  "request": {
    "url": "/v3/kafka/local",
    "host": "f2906c456102"
  }
}
```  


#### List Consumers
명시한 클러스터에 생성된 `Consumer Group` 리스트를 응답한다. 
응답되는 `Consumer Group` 은 `Kafka` 와 완전히 동기화된 목록이 아닌 지금까지 `Burrow` 에 등록된 목록이다.  

`GET /v3/kafka/(cluster)/consumer`

```bash
http://localhost:8000/v3/kafka/local/consumer

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 165

{
  "error": false,
  "message": "consumer list returned",
  "consumers": [
    "test-topic-group",
    "burrow-local"
  ],
  "request": {
    "url": "/v3/kafka/local/consumer",
    "host": "f2906c456102"
  }
}
```  

#### List Cluster Topics
명시한 클러스터에 생성된 토픽 목록을 응답한다.  

`GET /v3/kafka/(cluster)/topic`

```bash
http://localhost:8000/v3/kafka/local/topic

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 184

{
  "error": false,
  "message": "topic list returned",
  "topics": [
    "test-topic",
    "test-topic2",
    "test-topic3",
    "__consumer_offsets"
  ],
  "request": {
    "url": "/v3/kafka/local/topic",
    "host": "f2906c456102"
  }
}
```  

#### Get Consumer Detail
요청에 명시한 `Consumer Group` 의 상세 정보를 응답한다.  

`GET /v3/kafka/(cluster)/consumer/(group)`

```bash
http://localhost:8000/v3/kafka/local/consumer/test-topic-group

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 15 Oct 2022 09:30:17 GMT
Content-Length: 1828

{
  "error": false,
  "message": "consumer detail returned",
  "topics": {
    "test-topic": [
      {
        "offsets": [
          {
            "offset": 0,
            "timestamp": 1665826170881,
            "observedAt": 1665826170000,
            "lag": 0
          },
			.. 생략 ..
          {
            "offset": 0,
            "timestamp": 1665826215890,
            "observedAt": 1665826215000,
            "lag": 0
          }
        ],
        "owner": "/10.89.2.3",
        "client_id": "consumer-test-topic-group-1",
        "current-lag": 0
      },
      {
        "offsets": [
          {
            "offset": 0,
            "timestamp": 1665826170881,
            "observedAt": 1665826170000,
            "lag": 0
          },
			.. 생략 ..
          {
            "offset": 0,
            "timestamp": 1665826215890,
            "observedAt": 1665826215000,
            "lag": 0
          }
        ],
        "owner": "/10.89.2.3",
        "client_id": "consumer-test-topic-group-1",
        "current-lag": 0
      }
    ]
  },
  "request": {
    "url": "/v3/kafka/local/consumer/test-topic-group",
    "host": "f2906c456102"
  }
}
```  


#### Consumer Group Status
명시한 `Consumer Group` 의 전체 피타션에 대한 평가 결과를 응답한다. 
평가는 요청과 함께 수행되고 결과는 `Burrow` 의 `consumer lag evailuation rules` 를 따른다.  

해당 요청은 두 종류가 있는데, 
우선 `/status` 는 가장 상태가 좋지 않는 파티션에 대한 정보만 응답한다. 
그리고 `/lag` 는 전체 파티션에 대한 평가 결과를 응답한다.  

`status` 의 종류는 아래와 같다.  

status|설명
---|---
NOTFOUND|Consumer group 이 cluster 에 존재하지 않는 경우
OK|Consumer group, Partition 정상동작 중
WARN|offset 이 증가하면서 lag도 증가하는 경우(생산되는 데이터는 증가했지만 Consumer 가 이를 따라가지 못하는 경우)
ERR|offset 이 중지된 경우이지만 lag이 0이 아닌 경우
STOP|offset commit 이 일정시간 이상 중지된 경우
STALL|offset 이 commit 되고 있지만 lag 이 0이 아닌 경우

`GET /v3/kafka/(cluster)/consumer/(group)/status`

```bash
http://localhost:8000/v3/kafka/local/consumer/test-topic-group/status

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 15 Oct 2022 09:43:17 GMT
Content-Length: 589

{
  "error": false,
  "message": "consumer status returned",
  "status": {
    "cluster": "local",
    "group": "test-topic-group",
    "status": "OK",
    "complete": 1,
    "partitions": [],
    "partition_count": 2,
    "maxlag": {
      "topic": "test-topic",
      "partition": 0,
      "owner": "/10.89.2.3",
      "client_id": "consumer-test-topic-group-1",
      "status": "OK",
      "start": {
        "offset": 0,
        "timestamp": 1665826951059,
        "observedAt": 1665826951000,
        "lag": 0
      },
      "end": {
        "offset": 0,
        "timestamp": 1665826996067,
        "observedAt": 1665826996000,
        "lag": 0
      },
      "current_lag": 0,
      "complete": 1
    },
    "totallag": 0
  },
  "request": {
    "url": "/v3/kafka/local/consumer/test-topic-group/status",
    "host": "f2906c456102"
  }
}
```  

`GET /v3/kafka/(cluster)/consumer/(group)/lag`

```bash
http://localhost:8000/v3/kafka/local/consumer/test-topic-group/lag

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 15 Oct 2022 09:43:33 GMT
Content-Length: 1195

{
  "error": false,
  "message": "consumer status returned",
  "status": {
    "cluster": "local",
    "group": "test-topic-group",
    "status": "OK",
    "complete": 1,
    "partitions": [
      {
        "topic": "test-topic",
        "partition": 0,
        "owner": "/10.89.2.3",
        "client_id": "consumer-test-topic-group-1",
        "status": "OK",
        "start": {
          "offset": 0,
          "timestamp": 1665826966062,
          "observedAt": 1665826966000,
          "lag": 0
        },
        "end": {
          "offset": 0,
          "timestamp": 1665827011068,
          "observedAt": 1665827011000,
          "lag": 0
        },
        "current_lag": 0,
        "complete": 1
      },
      {
        "topic": "test-topic",
        "partition": 1,
        "owner": "/10.89.2.3",
        "client_id": "consumer-test-topic-group-1",
        "status": "OK",
        "start": {
          "offset": 0,
          "timestamp": 1665826966062,
          "observedAt": 1665826966000,
          "lag": 0
        },
        "end": {
          "offset": 0,
          "timestamp": 1665827011068,
          "observedAt": 1665827011000,
          "lag": 0
        },
        "current_lag": 0,
        "complete": 1
      }
    ],
    "partition_count": 2,
    "maxlag": {
      "topic": "test-topic",
      "partition": 0,
      "owner": "/10.89.2.3",
      "client_id": "consumer-test-topic-group-1",
      "status": "OK",
      "start": {
        "offset": 0,
        "timestamp": 1665826966062,
        "observedAt": 1665826966000,
        "lag": 0
      },
      "end": {
        "offset": 0,
        "timestamp": 1665827011068,
        "observedAt": 1665827011000,
        "lag": 0
      },
      "current_lag": 0,
      "complete": 1
    },
    "totallag": 0
  },
  "request": {
    "url": "/v3/kafka/local/consumer/test-topic-group/lag",
    "host": "f2906c456102"
  }
}
```  

#### Remove Consumer Group
명시한 `Consumer Group` 의 `offset` 을 제거한다. 
토픽에 대한 `Consumer` 목록의 변경사항이 있는 경우 사용할 수 있는 엔드포인트이다. 
`Kafka Broker` 에서 `Consumer Group` 이 삭제 되더라도, `Burrow` 에서는 계속해서 `Consumer Group` 의 `offset` 을 유지하고 있다는 점을 기억해야 한다.  

`DELETE /v3/kafka/(cluster)/consumer/(group)`

```bash
http://localhost:8000/v3/kafka/local/consumer/test-topic-group

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 134

{
  "error": false,
  "message": "consumer group removed",
  "request": {
    "url": "/v3/kafka/local/consumer/test-topic-group",
    "host": "f2906c456102"
  }
}
```  

#### Get Topic Detail
명시한 토픽의 상세정보인 최신 `offset` 값을 모든 파티션에 대해서 응답한다. 

`GET /v3/kafka/(cluster)/topic/(topic)`

```bash
http://localhost:8000/v3/kafka/local/topic/test-topic

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 141

{
  "error": false,
  "message": "topic offsets returned",
  "offsets": [
    0,
    0
  ],
  "request": {
    "url": "/v3/kafka/local/topic/test-topic",
    "host": "f2906c456102"
  }
}
```  



---
## Reference
[Burrow - Kafka Consumer Lag Checking](https://github.com/linkedin/Burrow)  
[Burrow: Kafka Consumer Monitoring Reinvented](https://engineering.linkedin.com/apache-kafka/burrow-kafka-consumer-monitoring-reinvented)  
[HTTP Endpoint](https://github.com/linkedin/Burrow/wiki/HTTP-Endpoint)  
[Setting Up Burrow & Burrow Dashboard For Kafka Lag Monitoring](https://www.lionbloggertech.com/setting-up-burrow-dashboard-for-kafka-lag-monitoring/)  
