--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 에서 파이프라인 구성을 할때 반복 작업을 최소화 할 수 있는 Kafka Connect 에 대해 알아 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
    - Pipeline
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


`Kafka Connect` 는 `REST API` 를 통해 실행 중인 커넥트, 플러그인 종류, 태스트 상태 등을 조회 할 수 있다. 
기본으로 `8083` 포트를 사용하고 엔드 포인트의 종류와 설명은 아래와 같다. 

Method| Path                                             |Desc
---|--------------------------------------------------|---
GET| /                                                |실행 중인 커넥트 정보 확인
GET| /connectors                                      |실행 중인 커넥터 이름 확인
POST| /connectors                                      |새로운 커넥터 생성
GET| /connectors/<커넥터 이름>                             |실행 중인 커넥터 정보 확인
GET| /connectors/<커넥터 이름>/config                      |실행 중인 커넥터 설정값 확인
PUT| /connectors/<커넥터 이름>/config                      |실행 중인 커넥터 설정값 변경
GET| /connectors/<커넥터 이름>/status                      |실행 중인 커넥터 상태 확인
POST| /connectors/<커넥터 이름>/restart                     |실행 중인 커넥터 재시작
PUT| /connectors/<커넥터 이름>/pause                       |커넥터 일시 중지
PUT| /connectors/<커넥터 이름>/resume                      |일시 중지 커넥터 실행
DELETE| /connectors/<커넥터 이름>/                            |실행 중인 커넥터 종료
GET| /connectors/<커넥터 이름>/tasks                       |실행 중인 커넥터의 태스크 정보 확인
GET| /connectors/<커넥터 이름>/tasks/<태스크 아이디>/status      |실행 중인 커넥터의 태스크 상태 확인
POST| /connectors/<커넥터 이름>/tasks/<태스크 아이디>/restart     |실행 중인 커넥터의 태스크 재시작
GET| /connectors/<커넥터 이름>/topics                      |커넥터별 연동된 토픽 정보 확인
GET| /connector-plugins/                              |커넥트에 존재하는 커넥터별 플러그인 확인
PUT| /connector-plugins/<커넥터 플러그인 이름>/config/validate |커넥터 생성 시 설정값 유효 여부 확인

### 단일 모드 커넥트
`standalone mode kafka connect` 는 1개 프로세스만 실행되는 커텍트이다. 
즉 단일 프로세스로만 구동 시키기 때문에 고가용성에 대한 대응이 되지 않아, 
단일 장애점(SPOF)이 될 수 있다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-3.drawio.png)  

단일 모드 커넥트 실행은 설정 파일인 `connect-standalone.properties` 파일 수정을 통해 가능하다. 

```bash
$ docker exec -it myKafka cat /opt/kafka/config/connect-standalone.properties

# 커넥트와 연동할 카프카 클러스터의 호스트:포트를 작성한다.
# 2개 이상 브로커인 경우 , 로 구분해서 작성한다. 
bootstrap.servers=localhost:9092

# 데이터를 카프카에 저장할 때, 카프카에서 데이터를 읽어올 때 변환 방식을 정의 한다. 
# 카프카 커넥트는 기본으로 JsonConverter, StringConverter, ByteArrayConverter 를 제공한다. 
# 스키마 형태를 사용하고 싶지 않다면 schemas.enable=false 로 설정 한다. 
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true

# 커넥트의 오프셋 정보를 저장할 파일 위치를 작성한다.
# 단일 모드의 경우 로컬 파일에 관리하기 때문에 로컬 경로를 작성해 준다. 
# 소스 커넥터, 싱크 커넥터가 데이터를 어디 까지 처리했는지에 대한 정보를 저장한다. 
# 이 정보는 커넥터가 재시작 된 후 마지막 위치를 읽어 작업을 이어사 처리하게 된다. 
offset.storage.file.filename=/tmp/connect.offsets

# 태스크가 처리 완료한 오프셋을 커밋하는 주기를 설정 한다. 
offset.flush.interval.ms=10000

# 플러그인 형태로 추가할 커넥터의 디렉토리 경로를 입력한다. 
# 직접 개발, 오픈소스의 커넥터 jar 파일이 위치하는 디렉토리 주소를 값으로 입력한다. 
# 2개 이상인 경우 , 로 구분해서 작성 할 수 있다. 
# 커넥터, Converter, Transform 등 추가가 가능하다. 
#plugin.path=
#or
plugin.path=<plugin-path>
```  

예제는 기본으로 제공하는 파일 소스 커넥터를 사용해서 진행한다. 
파일 소스 커넥터는 특정 위치에 있는 파일을 읽어 토픽으로 데이터를 저장하는 데 사용할 수 있는 커넥터이다. 
파일 소스 커넥터의 설정 파일의 위치와 내용은 아래와 같다.  

```bash
$ docker exec -it myKafka cat /opt/kafka/config/connect-file-source.properties

# 커넥터 이름
name=local-file-source

# 사용할 커넥터 클래스 이름
# 카프카에서 제공하는 기본 클래스 중 하나인 FileStreamSource 를 사용한다. 
connector.class=FileStreamSource

# 커넥터로 실행할 태스크 수
# 태스트 수를 늘리면 병렬처리가 가능하다. 
# 다수의 파일을 읽어서 토픽에 저장하는 니즈가 있다면 태스크 수를 늘려 병렬 처리 할 수 있다. 
tasks.max=1

# 읽을 파일 위치
file=/tmp/test.txt

# 읽은 파일 데이터를 저장할 토픽 이름
topic=connect-test
```  

단일 모드의 실행은 `connect-standalone.sh` 파일을 실행 할때 
커넥트 설정 파일인 `connect-standalone.properties` 와 
사용하고자 하는 소스 커넥터의 설정 파일인 `connect-file-source.properties` 를 넣어 실행하면 된다. 

```bash
$ docker exec -it myKafka /opt/kafka/bin/connect-standalone.sh \
/opt/kafka/config/connect-standalone.properties \
/opt/kafka/config/connect-file-source.properties
```  

그리고 `/tmp/test.txt` 에 아래와 같이 데이터를 추가해 주면 `connect-test` 토픽에 파일에 추가한 데이터가 들어간 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka echo -e "foo\nbar\n" >> /tmp/test.txt

$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
```  


### 분산 모드 커넥트
`distributed mode kafka connect` 는 2대 이상의 서버에 클러스터 형태로 운영하기 때문에, 
단일 모드 커넥트와 비교해서 더 안전성 있게 운영할 수 있다. 
특정 1개 커넥트에 이슈가 발생하더라도 나머지 커넥트가 작업을 이어 지속적으로 수행할 수 있기 때문이다. 
또한 작업 처리량에 대한 대응도 무중단 스케일 아웃으로 자유롭게 대응 할 수 있다는 장점이 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-4.drawio.png)  

분산 모드 커넥트 설정 파일은 `connect-distributed.properties` 로 경로와 내용은 아래와 같다.  

```bash
$ docker exec -it myKafka cat /opt/kafka/config/connect-distributed.properties

# 커넥트와 연동할 카프카 클러스터 호스트:포트를 작성한다. 
# 2개 이상의 브로커인 경우 , 로 구분한다. 
bootstrap.servers=localhost:9092

# 다수의 커넥트 프로세스들을 묶는 그룹 이름을 설정한다. 
# 동일한 group.id 로 지정된 커넥트는 분산 되어 실행 된다. 
group.id=connect-cluster

# 데이터를 카프카에 저장할 때, 카프카에서 데이터를 읽어올 때 변환 방식을 정의 한다. 
# 카프카 커넥트는 기본으로 JsonConverter, StringConverter, ByteArrayConverter 를 제공한다. 
# 스키마 형태를 사용하고 싶지 않다면 schemas.enable=false 로 설정 한다. 
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true

# 분산 모두 커넥트는 카프카 토픽에 오프셋 정보를 저장해서 관리한다. 
# 오프셋 정보는 소스, 싱크 커넥터가 데이터 처리 시점에 대한 정보이다. 
# 이를 통해 특정 커넥터가 중지 후 재시작 되더라더 토픽의 오픽셋 값을 통해 작업을 이어서 진행 할 수 있다. 
# 복제 개수는 안정 성을 위해 3보다 큰 값으로 설정하는 것이 좋다. 
offset.storage.topic=connect-offsets
offset.storage.replication.factor=1
#offset.storage.partitions=25
config.storage.topic=connect-configs
config.storage.replication.factor=1
status.storage.topic=connect-status
status.storage.replication.factor=1
#status.storage.partitions=5

# 태스크가 처리 완료한 오프셋 커밋 주기를 설정한다. 
offset.flush.interval.ms=10000

# 플러그인 형태로 추가할 커넥터의 디렉토리 경로를 입력한다. 
# 직접 개발, 오픈소스의 커넥터 jar 파일이 위치하는 디렉토리 주소를 값으로 입력한다. 
# 2개 이상인 경우 , 로 구분해서 작성 할 수 있다. 
# 커넥터, Converter, Transform 등 추가가 가능하다. 
#plugin.path=
#or
plugin.path=<plugin-path>
```  

분산 모드 커넥트 실행은 `connect-distributed.sh` 와 커넥트 설정 파일인 `connect-distribueted.properties` 
만 있으면 실행 가능하고, 커넥터 실행, 추가, 삭제, 변경 등은 `REST API` 를 사용해서 가능하다.  

```bash
$ docker exec -it myKafka /opt/kafka/bin/connect-distributed.sh \
/opt/kafka/config/connect-distributed.properties
```  

예제에서는 테스트를 위해 커넥트 프로세스를 1개만 설정 했지만, 
실제 환경에서는 2대 이상의 서버에 분산 모드로 실행하는 것을 권장 한다. 

.. 그림 ..

분산 모드가 실행 된 이후 커넥트에 등록된 플러그인을 조회하면 아래와 같다.  

```bash
$ docker exec -it myKafka curl -X GET http://localhost:8083/connector-plugins | jq
[
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]
```  

`Kafka 2.5.0` 부터는 위와 같이 5개의 커넥터가 기본으로 제공된다.  

이번에도 `FileStreamSourceConnector` 를 사용해서 파일의 내용을 특정 카프카 토픽에 추가하는 예제를 진행해 본다. 
커넥터 실행을 위해 아래와 같이 `REST API` 요청을 해주면 된다.  

```bash
$ docker exec -it myKafka curl -X POST -H "Content-Type: application/json" \
> --data '{
quote>   "name" : "local-file-source",
quote>   "config" : {
quote>     "connector.class" : "org.apache.kafka.connect.file.FileStreamSourceConnector",
quote>     "file" : "/tmp/test.txt",
quote>     "tasks.max" : "1",
quote>     "topic" : "connect-distributed-test"
quote>   }
quote> }' \
> http://localhost:8083/connectors
```  

커넥터 실행 요청이 정상적으로 수행 중인지 확인을 위해 `REST API` 로 확인하면 아래와 같다.  

```bash
$ docker exec -it myKafka curl -X GET http://localhost:8083/connectors/local-file-source/status | jq
{
  "name": "local-file-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.89.0.2:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.89.0.2:8083"
    }
  ],
  "type": "source"
}
```  

그리고 최종적으로 `connect-distributed-test` 토픽에 `/tmp/test.txt` 파일 데이터가 잘 들어 갔는지 확인하면 아래와 같다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-distributed-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
```  

정상동작을 확인 했으니 실행 중인 커넥터를 종료하고 삭제하는 방법은 아래와 같다.  

```bash
$ docker exec -it myKafka curl -X DELETE http://localhost:8083/connectors/local-file-source

$ docker exec -it myKafka curl -X GET http://localhost:8083/connectors
[]
```  

삭제이후 현재 실행 중인 커넥터가 없는 것을 확인 할 수 있다.  


---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
