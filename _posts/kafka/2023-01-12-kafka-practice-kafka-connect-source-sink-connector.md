--- 
layout: single
classes: wide
title: "[Kafka]"
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

## Kafka Connect Source, Sink Connector
`Kafka Connect` 는 크게 소스로 부터 데이터를 읽어와 카프카에 넣는 `Source Connector` 와 
카프카에서 데이터를 읽어 타겟으로 전달하는 `Sink Connector` 로 구성된다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-1.drawio.png)  

이번 포스트에서는 직접 `Source Connector` 와 `Sink Connector` 를 구현해서 `Kafka Connect` 를 통해 실행하는 방법에 대해 알아본다. 
그리고 커스텀하게 구현한 `Source Connector` 외 `Sink Connector` 를 `docker` 를 기반으로 `Kafka Cluster` 와 연동해 실행하는 방법까지 알아본다. 

### Source Connector
소스 커넥터는 소스 애플리케이션 혹은 소스 파일로 부터 데이터를 읽어와 토픽으로 넣는 역할을 한다. 
이미 구현돼 있는 오픈소스를 사용해도 되지만 라이센스 혹은 추가 기능등이 필요 한 경우는 카프카 커넥트 라이브러리에서 제공하는 
`SourceConnector` 와 `SourceTask` 를 사용해서 직접 구현 가능하다. 
직접 구현한 소스 커넥터는 빌드해서 `jar` 파일로 만든 후 카프카 커넥트 실행 시 플러그인으로 추가해서 사용해야 한다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-2.drawio.png)  

`SourceConnector` 클래스는 태스크 실행 전 커넥터 설정파일을 초기화하고 태스크를 결정하는 정의하는 역할을 한다. 
`SourceConnector` 구현은 상속받을 사용자 클래스를 생성해서 구현한 필요한 메서드를 구현해 주면 된다.  

```java
// 해당 클래스의 이름이 커텍트에서 호출 할때 사용된다. 
// 그러므로 커넥터 동작을 좀더 명확히 표현할 수 있는 이름으로 만들어주는 것이 좋다. 
// MySqlSourceConnector, MongoDbSourceConnector, ..
public class ExamSourceConnector extends SourceConnector {
    @Override
    public void start(Map<String, String> props) {
        // 사용자가 커넥터 실행을 위해 JSON, config 파일 등으로 입력한 설정값을 초기화 한다. 
        // 잘못된 값이 들어온 경우 ConnectException() 을 호출해서 커넥터를 종료시킬 수 있다. 
    }

    @Override
    public Class<? extends Task> taskClass() {
        // 커넥터가 사용할 태스트 클래스를 반환 한다. 
        return null;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // 실행하는 태스크의 수가 2개 이상인 경우 태스크마다 각 다른 옵션 설정이 필요한 경우 사용할 수 있다. 
        return null;
    }

    @Override
    public void stop() {
        // 커넥터 종료에 필요한 동작을 작성한다. 
    }

    @Override
    public ConfigDef config() {
        // 커넥터가 사용할 설정값에 정보를 반환한다. 
        // ConfigDef 클래스를 사용해서 커넥터의 설정 정보를 작성 할 수 있다. 
        return null;
    }

    @Override
    public String version() {
        // 해당 커넥터의 버전을 반환한다. 
        // 커넥트에서 플러그인을 조회할 떄 노출 되는 버전정보 이다.
        return null;
    }
}
```  

`SourceTask` 는 실제로 데이터를 다루는 역할을 하는 클래스로, 
소스(애프리케이션, 파일, ..)에서 데이터를 읽어와 토픽으로 데이터를 보내는 역할을 한다. 
`SourceTask` 는 데이터를 넣는 토픽의 오프셋을 사용하지 않고 자체적인 오프셋을 사용해서 상태관리를 한다. 
해당 오프셋은 소스 데이터를 어느 부분까지 읽었는지에 대한 상태 값으로 중복 발송 등을 방지 할 수 있다.  

```java
public class ExamSourceTask extends SourceTask {
    @Override
    public void start(Map<String, String> props) {
        // 태스크 시점 시점에 필요한 동작을 작성한다. 
        // 주로 태스크에서 사용할 리소스를 초기화 하는 용도로 쓰인다. 
        // JDBC 커넥션, File 객체 초기화 등 ..
    }

    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        // 소스(애플리케이션, 파일)로 데이터를 읽어오는 동작을 작성한다. 
        // 읽어온 데이터는 SourceRecord 객체에 담아 타겟이 되는 토픽으로 전송한다. 
        // 토픽으로 전송할 데이터는 List<SourceRecord> 와 같이 리스트 형태로 리턴하면 한번에 전송 가능하다. 
        return null;
    }

    @Override
    public void stop() {
        // 태스크 종료에 필요한 동작을 작성한다. 
        // 일반적으로 start() 에서 초기화한 리소스를 반환하는 용도로 사용된다. 
    }

    @Override
    public String version() {
        // 태스트의 버전를 반환한다. 
        // 일반적으로 SourceConnector 와 동일한 버전을 작성한다. 
        return null;
    }
}
```  

#### Source Connector 구현해보기 
`SourceConnector` 에 대한 개념과 구현에 필요한 내용에 대해 알아보았으니, 
이제 실제로 구현해보고 이를 `Kafka Cluster` 와 연동해서 동작 결과를 살펴본다. 
구현할 `SourceConnector` 는 `MyFileSourceConnector` 로 특정 파일에 작성된 내용을 읽어 특정 토픽으로 전송하는 `SourceConnector` 이다.  

- `build.gradle`

```groovy
plugins {
    id 'java'
}

group 'com.windowforsun.kafkaconnect.sourceconnector'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.apache.kafka:connect-api:3.2.0'
    implementation 'org.slf4j:slf4j-simple:1.7.30'
}

test {
    useJUnitPlatform()
}
```  

- `MyFileSourceConnectorConfig`

```java
public class MyFileSourceConnectorConfig extends AbstractConfig {
    public static final String DIR_FILE_NAME = "file";
    private static final String DIR_FILE_NAME_DEFAULT_VALUE = "/tmp/kafka.txt";
    private static final String DIR_FILE_NAME_DOC = "읽을 파일 경로와 이름";
    public static final String TOPIC_NAME = "topic";
    private static final String TOPIC_DEFAULT_NAME = "test";
    private static final String TOPIC_DOC = "보낼 토픽명";

    // 커넥터 실행시 옵션으로 받을 필드 설정
    public static ConfigDef CONFIG = new ConfigDef()
            .define(DIR_FILE_NAME,
                    ConfigDef.Type.STRING,
                    DIR_FILE_NAME_DEFAULT_VALUE,
                    ConfigDef.Importance.HIGH,
                    DIR_FILE_NAME_DOC)
            .define(TOPIC_NAME,
                    ConfigDef.Type.STRING,
                    TOPIC_DEFAULT_NAME,
                    ConfigDef.Importance.HIGH,
                    TOPIC_DOC);

    public MyFileSourceConnectorConfig(Map<?, ?> originals) {
        super(CONFIG, originals);
    }
}
```  

- `MyFileSourceConnector`

```java
public class MyFileSourceConnector extends SourceConnector {
    private final Logger logger = LoggerFactory.getLogger(MyFileSourceConnector.class);

    private Map<String, String> configProperties;

    @Override
    public void start(Map<String, String> props) {
        this.configProperties = props;
        try {
            new MyFileSourceConnectorConfig(props);
        } catch (ConfigException e) {
            throw new ConnectException(e.getMessage(), e);
        }
    }

    @Override
    public Class<? extends Task> taskClass() {
        return MyFileSourceTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        List<Map<String, String>> taskConfigs = new ArrayList<>();
        Map<String, String> taskProps = new HashMap<>(this.configProperties);

        for(int i = 0; i < maxTasks; i++) {
            taskConfigs.add(taskProps);
        }

        return taskConfigs;
    }

    @Override
    public void stop() {
    }

    @Override
    public ConfigDef config() {
        return MyFileSourceConnectorConfig.CONFIG;
    }

    @Override
    public String version() {
        return "1.0";
    }
}
```  

- `MyFileSourceTask`

```java
public class MyFileSourceTask extends SourceTask {
    private Logger logger = LoggerFactory.getLogger(MyFileSourceTask.class);

    public final String FILENAME_FIELD = "filename";
    public final String POSITION_FIELD = "position";

    private Map<String, String> fileNamePartition;
    private Map<String, Object> offset;
    private String topic;
    private String file;
    private Long position = -1L;

    @Override
    public void start(Map<String, String> props) {
        try {
            // 태스크 실행에 필요한 변수 초기화
            MyFileSourceConnectorConfig config = new MyFileSourceConnectorConfig(props);
            this.topic = config.getString(MyFileSourceConnectorConfig.TOPIC_NAME);
            this.file = config.getString(MyFileSourceConnectorConfig.DIR_FILE_NAME);
            this.fileNamePartition = Collections.singletonMap(FILENAME_FIELD, this.file);
            this.offset = this.context.offsetStorageReader().offset(this.fileNamePartition);

            // 오프셋 최신화
            if(this.offset != null) {
                Object lastReadFileOffset = this.offset.get(POSITION_FIELD);
                if(lastReadFileOffset != null) {
                    this.position = (Long) lastReadFileOffset;
                }
            } else {
                this.position = 0L;
            }

        } catch (Exception e) {
            throw new ConnectException(e.getMessage(), e);
        }
    }

    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        List<SourceRecord> results = new ArrayList<>();
        try {
            Thread.sleep(1000);

            List<String> lines = this.getLines(position);

            if(lines.size() > 0) {
                lines.forEach(line -> {
                    Map<String, Long> sourceOffset = Collections.singletonMap(POSITION_FIELD, ++position);
                    SourceRecord sourceRecord = new SourceRecord(this.fileNamePartition, sourceOffset, this.topic, Schema.STRING_SCHEMA, line);
                    results.add(sourceRecord);
                });
            }

            return results;
        } catch(Exception e) {
            logger.error(e.getMessage(), e);
//            throw new ConnectException(e.getMessage(), e);
        }

        return results;
    }

    private List<String> getLines(long readLine) throws Exception {
        BufferedReader reader = Files.newBufferedReader(Paths.get(this.file));
        return reader.lines().skip(readLine).collect(Collectors.toList());
    }

    @Override
    public void stop() {

    }

    @Override
    public String version() {
        return "1.0";
    }
}
```  

- `Dockerfie`
  - `docker build -t my-source-connector .` 명령으로 빌드한다. 

```dockerfile
FROM gradle:7-jdk11 as builder
WORKDIR /build
COPY build.gradle /build/
RUN gradle build -x test --parallel --continue > /dev/null 2>&1 || true
COPY . /build
RUN gradle build -x test --parallel


FROM confluentinc/cp-kafka-connect-base:6.1.1
ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components,/tmp"
COPY --from=builder /build/build/libs/*.jar /tmp/kafka-connect-source-connector.jar
```  

- `docker-compose.yaml`
  - `docker-compose up --build` 명령으로 전체 구성을 실행한다. 

```yaml
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
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

  mySourceConnector:
    container_name: mySourceConnector
    image: localhost/my-source-connector
    ports:
      - "8083:8083"
    environment:
      CONNECT_GROUP_ID: 'my-source-connector'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_PLUGIN_PATH: '/usr/share/java,/usr/share/confluent-hub-components,/tmp'
      CONNECT_CONFIG_STORAGE_TOPIC: 'my-connect-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'my-connect-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'my-connect-status'
      CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.json.JsonConverter'
      CONNECT_REST_ADVERTISED_HOST_NAME: mySourceConnector
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
```  

`docker-compose` 구성이 모두 정상적으로 실행 된 후 `REST API` 를 통해 현재 커넥트에 설정된 플러그인 목록을 조회하면 아래이 
우리가 직접 구현한 `MyFileSourceConnector` 가 위치한 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:8083/connector-plugins | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   737  100   737    0     0   2358      0 --:--:-- --:--:-- --:--:--  2400
[
  {
    "class": "com.windowforsun.kafkaconnect.sourceconnector.exam.MyFileSourceConnector",
    "type": "source",
    "version": "1.0"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "6.1.1-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "6.1.1-ccs"
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

해당 플러그인을 사용해서 커넥터를 실행하기 위해 아래 `REST API` 요청을 수행해 준다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
"name" : "my-source-connector",
"config" : {
"connector.class" : "com.windowforsun.kafkaconnect.sourceconnector.exam.MyFileSourceConnector",
"file" : "/tmp/my-source-file.txt",
"topic" : "my-source-topic"
}
}' \
http://localhost:8083/connectors
```  
 
실행한 커넥터가 정상적으로 실행 중인지 확인하면 아래와 같다.  

```bash
$ curl -X GET http://localhost:8083/connectors/my-source-connector/status | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   189  100   189    0     0   1308      0 --:--:-- --:--:-- --:--:--  1350
{
  "name": "my-source-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "mySourceConnector:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "mySourceConnector:8083"
    }
  ],
  "type": "source"
}
```  

커넥터가 실행 중인 컨테이너에 접속해서 정의한 소스 파일에 값을 넣어 준다.  

```bash
$ docker exec -it mySourceConnector /bin/bash
$ echo -e "foo\nbar\n" >> /tmp/my-source-file.txt
```  

최종적으로 타겟이 되는 `my-source-topic` 을 컨슈머를 통해 들어간 데이터를 확인하면 아래와 같이 `my-source-file.txt` 의 파일 내용이 토픽에 들어가 있는 것을 확인 할 수 있다. 

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-source-topic --from-beginning

{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
{"schema":{"type":"string","optional":false},"payload":""}
```  


### Sink Connector


![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-3.drawio.png)  



---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  

[robcowart/cp-kafka-connect-custom](https://github.com/robcowart/cp-kafka-connect-custom)  
[conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose)  
