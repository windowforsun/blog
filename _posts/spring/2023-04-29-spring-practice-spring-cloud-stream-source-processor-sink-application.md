--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Stream Source, Processor, Sink 개발"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Stream 을 사용해서 Source, Processor, Sink 애플리케이션을 개발하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Integration
    - Java DSL
    - MessageChannel
    - PollableChannel
    - SubscribableChannel
    - PublishSubscribeChannel
    - QueueChannel
    - PriorityChannel
    - RendezvousChannel
    - DirectChannel
    - ExecutorChannel
    - FluxMessageChannel
toc: true
use_math: true
---  

## Spring Cloud Stream
`Spring Cloud Stream` 을 사용하면 `Spring` 환경에서 메시지기반 `Stream` 애플리케이션을 개발할 수 있다. 
그리고 추후에는 이를 `Spring Cloud Data Flow` 와 같은 툴을 사용해 `Stream` 을 시각화 해서 관리 또한 가능하다.  

예제로 개발할 3개의 스트림 애플리케이션은 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/spring-cloud-stream-source-processor-sink-1.drawio.png)

- `DataSource` : `Source Application` 으로 `DataModel` 데이터를 생성해 `data-source-topic` 으로 전송한다.
- `DataProcessor` : `Processor Application` 으로 `data-source-topic` 에서 `DataModel` 을 받아 우선순위를 계산해 `DataPriorityModel` 을 생성하고 `data-process-topic` 으로 전송한다. 
- `DataSinkLog` : `Sink Application` 으로 `data-process-topic` 에서 `DataPriorityModel` 을 받아 최종 로그를 출력한다. 

개발을 진행한 애플리케이션의 디렉토리와 파일 구성은 아래와 같다.  

```
data-source
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── dataflow
    │   │               └── datasource
    │   │                   ├── DataModel.java
    │   │                   ├── DataSender.java
    │   │                   └── DataSourceApplication.java
    │   └── resources
    │       └── application.yaml
    └── test
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── dataflow
        │               └── datasource
        │                   ├── DataSourceApplicationTest.java
        │                   └── TmpTest.java
        └── resources

data-processor
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── dataflow
    │   │               └── dataprocessor
    │   │                   ├── DataModel.java
    │   │                   ├── DataPriorityModel.java
    │   │                   ├── DataProcessor.java
    │   │                   └── DataProcessorApplication.java
    │   └── resources
    │       └── application.yaml
    └── test
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── dataflow
        │               └── dataprocessor
        │                   └── DataProcessorApplicationTest.java
        └── resources

data-sink-log
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── dataflow
    │   │               └── datasinklog
    │   │                   ├── DataLog.java
    │   │                   ├── DataPriorityModel.java
    │   │                   └── DataSinkLogApplication.java
    │   └── resources
    │       └── application.yaml
    └── test
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── dataflow
        │               └── datasinklog
        │                   └── DataSinkLogApplicationTest.java
        └── resources

```

### 공통 부분
개발할 3개의 애플리케이션에서 공통으로 사용될 수 있는 파일이 있는데, 
실제로는 각 애플리케이션에 개발이 돼 있지만 해당 챕터에서 한번만 파일을 설명하도록 한다. 

- `docker-compose.yaml`
  - `Spring Cloud Stream` 애플리케이션에서 사용할 메시징 미들웨어 구성을 위한 파일이다. 
  - `docker-compose` 를 사용해서 `Kafka` 를 실행해 `Spring Cloud Stream` 에서 메시지 전달용으로 사용한다. 

```yaml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:6.1.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-server:6.1.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CONFLUENT_BALANCER_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: broker:29092
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'
```  

- `build.grale`
  - `Source`, `Processor`, `Sink` 애플리케이션에서 모두 공통으로 사용되는 `build.gradle` 이다. 

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.6.4'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'

ext {
    springCloudVersion = '2021.0.1'
}
repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-stream'
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    compileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"

    // spring cloud stream test
    testImplementation("org.springframework.cloud:spring-cloud-stream") {
        artifact {
            name = "spring-cloud-stream"
            extension = "jar"
            type ="test-jar"
            classifier = "test-binder"
        }
    }
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

test {
    useJUnitPlatform()
}
```  

- `DataModel`
  - `data-source-topic` 으로 전달되는 메시지 모델로 `Source`, `Processor` 에서 사용된다. 

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DataModel {
    private Long uid;
    private String dataLevel;
}
```  

- `DataPriorityModel`
  - `data-process-topic` 으로 전달되는 메시지 모델로 `Processor`, `Sink` 에서 사용된다. 

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DataPriorityModel {
    private Long uid;
    private String dataLevel;
    private Long priority;
}
```  

### DataSource
`Spring Cloud Stream` 에서 `Souce Application` 은 `Supplier` 를 반환하는 빈 객체를 구현해 주면된다. 

- `DataSourceApplication`

```java
@SpringBootApplication
public class DataSourceApplication {
    public static void main(String... args) {
        SpringApplication.run(DataSourceApplication.class, args);
    }
}
```  

- `DataSender`
  - `DataModel` 객체를 생성해 `uid` 와 `dataLevel` 에 값을 설정한다. 
  - `uid` 는 1씩 증가하는 값이고, `dataLevel` 은 `A` ~ `Z` 까지 랜덤 값이 설정된다. 
  - `Supplier` 에서 반환된 데이터는 `application.yaml` 에 설정된 `data-source-topic` 으로 전달된다. 

```java
@Slf4j
@Configuration
public class DataSender {
    @Bean
    public Supplier<DataModel> sendDataModel() {
        AtomicLong atomicLong = new AtomicLong();
        Random random = new Random();
        return () -> {
            DataModel dataModel =  DataModel.builder()
                    .uid(atomicLong.getAndIncrement())
                    .dataLevel(Character.toString('A' + random.nextInt(('Z' - 'A') + 1)))
                    .build();

            log.info("{}", dataModel);
            return dataModel;
        };
    }
}
```  

- `application.yaml`

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        # source 역할을 하는 bean 이름
        definition: sendDataModel
      bindings:
        # source 로 선언된 빈이름을 기반으로 stream out 바안딩 <빈이름>-<out>-<index>
        sendDataModel-out-0:
          # stream out 의 데이터가 들어갈 토픽
          destination: data-source-topic
```  

- `DataSourceApplicationTest`
  - 구현한 `Source Appliction` 을 테스트하는 클래스

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class DataSourceApplicationTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private CompositeMessageConverter converter;
    @Autowired
    private DataSender dataSender;

    @Test
    public void testSendDataModel() {
        this.dataSender.sendDataModel();

        Message<byte[]> result = this.outputDestination.receive(10000, "data-source-topic");
        DataModel dataModel = (DataModel) this.converter.fromMessage(result, DataModel.class);

        assertThat(dataModel.getUid()).isGreaterThanOrEqualTo(0L);
        assertThat(dataModel.getDataLevel()).isBetween("A", "Z");
    }
}
```  

### DataProcessor
`Spring Cloud Stream` 에서 `Processor Application` 은 `Function` 을 반환하는 빈 객체를 구현하면 된다.  

- `DataProcessorApplication`

```java
@SpringBootApplication
public class DataProcessorApplication {
    public static void main(String... args) {
        SpringApplication.run(DataProcessorApplication.class, args);
    }
}
```  

- `DataProcessor`
  - `data-source-topic` 에서 받은 `DataModel` 을 받아 `priority` 를 계산해 `DataPriorityModel` 을 리턴한다. 
  - `priority` 는 `(Z - dataLevel + 1) * uid` 로 계산된다. 
  - 그리고 `Function` 에서 반환된 데이터는 `application.yaml` 에 설정된 `data-process-topic` 으로 전달된다. 

```java
@Slf4j
@Configuration
public class DataProcessor {
    @Bean
    public Function<DataModel, DataPriorityModel> processPriority() {
        return dataModel -> {
            DataPriorityModel dataPriorityModel = DataPriorityModel.builder()
                    .uid(dataModel.getUid())
                    .dataLevel(dataModel.getDataLevel())
                    // (Z - dataLevel + 1) * uid
                    .priority(('Z' - dataModel.getDataLevel().charAt(0) + 1) * dataModel.getUid())
                    .build();

            log.info("{}", dataPriorityModel);

            return dataPriorityModel;
        };
    }
}
```  

- `application.yaml`

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        # processor 역할을 하는 bean 이름
        definition: processPriority
      bindings:
        # processor 로 선언된 빈이름을 기반으로 stream in 바안딩 <빈이름>-<in>-<index>
        processPriority-in-0:
          # stream in 의 데이터를 가져올 토픽
          destination: data-source-topic
        # processor 로 선언된 빈이름을 기반으로 stream out 바안딩 <빈이름>-<out>-<index>
        processPriority-out-0:
          # stream out 의 데이터가 들어갈 토픽
          destination: data-process-topic
```  

- `DataProcessorApplicatonTest`
    - 구현한 `Processor Appliction` 을 테스트하는 클래스

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class DataProcessorApplicationTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private InputDestination inputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void testProcessPriority() {
        DataModel dataModel = DataModel.builder()
                .uid(10L)
                .dataLevel("E")
                .build();
        Map<String, Object> headers = new HashMap<>();
        headers.put("contentType", "application/json");
        MessageHeaders messageHeaders = new MessageHeaders(headers);
        Message<?> sourceMessage = this.converter.toMessage(dataModel, messageHeaders);

        inputDestination.send(sourceMessage, "data-source-topic");

        Message<byte[]> processMessage = this.outputDestination.receive(10000, "data-process-topic");
        DataPriorityModel dataPriorityModel = (DataPriorityModel) this.converter
                .fromMessage(processMessage, DataPriorityModel.class);

        assertThat(dataPriorityModel.getUid()).isEqualTo(dataModel.getUid());
        assertThat(dataPriorityModel.getDataLevel()).isEqualTo(dataModel.getDataLevel());
        assertThat(dataPriorityModel.getPriority()).isEqualTo(220L);
    }
}
```  

### DataSinkLog
`Spring Cloud Stream` 의 `Sink Application` 은 `Consumer` 를 반환하는 빈을 구현해주면 된다.  

- `DataSinkLogApplication`

```java
@SpringBootApplication
public class DataSinkLogApplication {
    public static void main(String... args) {
        SpringApplication.run(DataSinkLogApplication.class, args);
    }
}
```  

- `DataLog`
  - `data-process-topic` 으로 전달 받은 데이터의 로그를 출력한다. 

```java
@Slf4j
@Configuration
public class DataLog {
    @Bean
    public Consumer<DataPriorityModel> logData() {
        return dataPriorityModel -> log.info("sink log : {}", dataPriorityModel);
    }
}
```  

- `application.yaml`

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        # sink 역할을 하는 bean 이름
        definition: logData
      bindings:
        # sink 로 선언된 빈이름을 기반으로 stream in 바안딩 <빈이름>-<in>-<index>
        logData-in-0:
          # stream in 의 데이터를 가져올 토픽
          destination: data-process-topic
```  

- `DataSinkLogApplicationTest`
    - 구현한 `Sink Appliction` 을 테스트하는 클래스

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class DataSinkLogApplicationTest {
    @Autowired
    private InputDestination inputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void testSinkLog() {
        DataPriorityModel input = DataPriorityModel.builder()
                .uid(10L)
                .dataLevel("E")
                .priority(220L)
                .build();
        Map<String, Object> headers = new HashMap<>();
        headers.put("contentType", "application/json");
        MessageHeaders messageHeaders = new MessageHeaders(headers);
        Message<?> inputMessage = this.converter.toMessage(input, messageHeaders);

        inputDestination.send(inputMessage, "data-process-topic");
    }
}
```  

### 테스트

`Kafka` 와 `Zookeeper` 의 구성이 있는데 `docker-compose.yaml` 을 아래 명령으로 실행한다.  

```bash
$ docker-compose up --build
[+] Running 2/2
 ⠿ Container zookeeper  Created                                                                                                                                                                                                    0.0s
 ⠿ Container broker     Created                                                                                                                                                                                                    0.0s
Attaching to broker, zookeeper

...
```  

앞서 구현한 `Stream Application` 들은 `jar` 로 빌드해 실행해도 되고, `Docker Image` 를 만들어서 실행해도 된다.  

먼저 `Source Application` 인 `DataSourceApplication` 을 실행한다. 
그럼 애플리케이션에서는 아래와 같은 로그가 출력되는 것을 확인 할 수 있다.  

```
.. DataSourceApplication 출력 로그 ..

INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=300, dataLevel=S)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=301, dataLevel=Q)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=302, dataLevel=V)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=303, dataLevel=I)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=304, dataLevel=J)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=305, dataLevel=R)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=306, dataLevel=J)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=307, dataLevel=L)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=308, dataLevel=G)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=309, dataLevel=G)
INFO 67408 --- [   scheduling-1] c.w.dataflow.datasource.DataSender       : DataModel(uid=310, dataLevel=G)
```  

그리고 `DataSourceApplication` 이 수행되면 `Kafka` 에 `data-source-topic` 이 생성된것과 해당 토픽에서도 데이터를 확인 할 수 있다.  

```bash
$ docker exec -it broker /bin/kafka-topics --bootstrap-server localhost:9092 --list
data-source-topic


$ docker exec -it broker /bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic data-source-topic --from-beginning 
{"uid":300,"dataLevel":"S"}
{"uid":301,"dataLevel":"Q"}
{"uid":302,"dataLevel":"V"}
{"uid":303,"dataLevel":"I"}
{"uid":304,"dataLevel":"J"}
{"uid":305,"dataLevel":"R"}
{"uid":306,"dataLevel":"J"}
{"uid":307,"dataLevel":"L"}
{"uid":308,"dataLevel":"G"}
{"uid":309,"dataLevel":"G"}
{"uid":310,"dataLevel":"G"}

...
```  

이어서 `Processor Application` 인 `DataProcessorApplication` 을 실행한다. 
그러면 애플리케이션에서는 아래와 같은 로그를 확인 할 수 있다. 

```
.. DataProcessorApplication 출력 로그 ..

INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=300, dataLevel=S, priority=2400)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=301, dataLevel=Q, priority=3010)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=302, dataLevel=V, priority=1510)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=303, dataLevel=I, priority=5454)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=304, dataLevel=J, priority=5168)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=305, dataLevel=R, priority=2745)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=306, dataLevel=J, priority=5202)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=307, dataLevel=L, priority=4605)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=308, dataLevel=G, priority=6160)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=309, dataLevel=G, priority=6180)
INFO 68642 --- [container-0-C-1] c.w.d.dataprocessor.DataProcessor        : DataPriorityModel(uid=310, dataLevel=G, priority=6200)
```  

그리고 `DataProcessorApplication` 이 수행되면 `Kafka` 에 `data-process-topic` 이 생성된 것과 해당 토픽에 결과 데이터를 확인 할 수 있다.  

```bash
$ docker exec -it broker /bin/kafka-topics --bootstrap-server localhost:9092 --list
data-process-topic
data-source-topic

$ docker exec -it broker /bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic data-process-topic --from-beginning 
{"uid":300,"dataLevel":"S","priority":2400}
{"uid":301,"dataLevel":"Q","priority":3010}
{"uid":302,"dataLevel":"V","priority":1510}
{"uid":303,"dataLevel":"I","priority":5454}
{"uid":304,"dataLevel":"J","priority":5168}
{"uid":305,"dataLevel":"R","priority":2745}
{"uid":306,"dataLevel":"J","priority":5202}
{"uid":307,"dataLevel":"L","priority":4605}
{"uid":308,"dataLevel":"G","priority":6160}
{"uid":309,"dataLevel":"G","priority":6180}
{"uid":310,"dataLevel":"G","priority":6200}

...
```  

마지막으로 `Sink Application` 인 `DataSinkLogApplication` 을 실행하면 `data-process-topic` 에서 데이터를 받아와 애플리케이션에서 최종 로그가 출력되는 것을 확인 할 수 있다.  

```
.. DataSinkLogApplication 출력 로그 ..

INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=300, dataLevel=S, priority=2400)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=301, dataLevel=Q, priority=3010)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=302, dataLevel=V, priority=1510)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=303, dataLevel=I, priority=5454)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=304, dataLevel=J, priority=5168)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=305, dataLevel=R, priority=2745)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=306, dataLevel=J, priority=5202)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=307, dataLevel=L, priority=4605)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=308, dataLevel=G, priority=6160)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=309, dataLevel=G, priority=6180)
INFO 71457 --- [container-0-C-1] c.w.dataflow.datasinklog.DataLog         : sink log : DataPriorityModel(uid=310, dataLevel=G, priority=6200)
```  


---  
## Reference
[Stream Application Development](https://dataflow.spring.io/docs/stream-developer-guides/streams/standalone-stream-sample/)  