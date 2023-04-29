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

### DataProcessor

### DataSinkLog

### 테스트




---  
## Reference
[]()  