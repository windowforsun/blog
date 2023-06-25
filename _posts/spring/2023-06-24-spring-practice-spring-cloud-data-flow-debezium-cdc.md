--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Data Flow
    - SCDF
    - Debezium
    - CDC
    - MySQL
    - Elasticsearch
toc: true
use_math: true
---  

## CDC for Spring Cloud Data Flow
`SCDF` 에서는 `Spring Cloud Stream` 에서 제공하는 기본 애플리케이션을 사용해서 `CDC` 스트림을 구성할 수 있다.
[Kafka Connect Debezium CDC]({{site.baseurl}}{% link _posts/kafka/2023-01-19-kafka-practice-kafka-connect-debezium-mysql-cdc-source-connector.md %})
에서는 `Kafka Connect` 을 사용한 `CDC` 를 알아보았다. 
`CDC` 를 위한 작업에 있어서는 약간 차이가 있을 수 있지만, 필요한 `Debezium` 설정은 큰 차이가 없다.  

[Spring Stream Application](https://github.com/spring-cloud/stream-applications)
의 현시점 최신 릴리즈 버전인 `2021.1.2` 버전까지는 
[CDC Debezium](https://github.com/spring-cloud/stream-applications/blob/v2021.1.2/applications/source/cdc-debezium-source/README.adoc)
라는 `CDC` 를 위한 `Source Application` 이 있다. 
해당 애플리케이션을 사용해도 `CDC` 구성은 문제없지만, 
본 포스트에서는 아직 정식 릴리즈는 되지 않은 `v4.0.0-RC1` 버전에 있는 
[Debezium Source](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/source/debezium-source/README.adoc)
를 사용해서 예제를 진행할 계획이다.  

`CDC` 를 구현할 전체 구성도는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/spring-cloud-data-flow-debezium-cdc.drawio.png)

- `debezium-source` : `MySQL` 의 변경사항을 `Kafka` 를 통해 `debezium-index-processor` 로 전달 한다. 
- `debezium-index-processor` : `debezium-source` 로 부터 받은 데이터에 `Elasticsearch` 의 인덱스관련 처리를 한휘 `elasticsearch-sink` 로 전달한다. 
- `elastiacsearch-sink` : 전달 받은 데이터를 설정된 인덱스의 데이터로 추가한다. 

예제 진행을 위해서는 `SCDF` 환경이 필요한데, 관련해서는 
[관련 글 1]({{site.baseurl}}{% link _posts/spring/2023-05-11-spring-practice-spring-cloud-data-flow-installation.md %}), 
[관련 글 2]({{site.baseurl}}{% link _posts/spring/2023-06-04-spring-practice-spring-cloud-data-flow-mysql.md %}), 
을 참고해서 구성할 수 있다.  

### Index Processor 구현
`debezium-index-processor` 는 `Elasticsearch` 에서 인덱스 관련 추가 처리를 하기위해 별도로 구현하는 애플리케이션이다.  

`MySQL` 변경사항들을 모두 `Elasticsearch` 에 저장하기 위해서 `elasticsearch-sink` 애플이케이션을 사용한다. 
`Elasticsearch` 에 데이터를 저장해 장기적으로 관리하기 위해 보편적으로 사용하는 방법은 
`Rolling index` 이다. 
인덱스 한개에 전체 데이터를 관리하는 것이 아니라, 날짜 혹은 특정 구분값을 기준으로 인덱스를 나눠서 데이터를 관리하는 방식이다.  

하지만 `elasticsearch-sink` 애플리케이션을 사용 했을때 `Rolling index` 를 적용할 만한 방법이 없어, 
추가적인 `Processor` 를 개발하기로 했다. 
그리고 해당 애플리케이션에서는 `Elasticsearch` 에서 시계열 방식으로 데이터를 관리하기 위해 `timestamp` 필드로 추가한다.  

구현할 애플리케이션은 `Spring Cloud Stream` 기반으로
[Spring Cloud Stream Application]({{site.baseurl}}{% link _posts/spring/2023-05-25-spring-practice-spring-cloud-data-flow-develop-deploy-application.md %}),
에서 진행한 예제와 비슷한 방식으로 구현된다.  

`Rolling Index` 를 위해서는 `elasticsearch-sink` 애플리케이션에서 헤더 중 `INDEX_NAME` 의 값이 있다면, 
해당 값으로 인덱스를 사용한다는 스펙을 이용한다. 
그리고 시계열 데이터 관리를 위해서는 `payload` 에 `timestamp` 새로운 필드를 추가한다.  

- `build.gradle`

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.6.4'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
sourceCompatibility = '11'
version 'v1'

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
    annotationProcessor "org.springframework.boot:spring-boot-configuration-processor"

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

jib {
    from {
        image = "openjdk:11-jre-slim"
        // for mac m1
        platforms {
//            platform {
//                architecture = "arm64"
//                os = "linux"
//            }
            platform {
                architecture = "amd64"
                os = "linux"
            }
        }
    }
    to {
        image = "debezium-cdc-index-processor"
        tags = ["${project.version}".toString()]
    }
    container {
        mainClass = "com.windowforsun.stream.processor.DebeziumCdcIndexProcessorApplication"
        ports = ["8080"]
    }

}
```  

- `DebeziumCdcIndexProcessorApplication`

```java
@SpringBootApplication
@EnableConfigurationProperties(IndexProperties.class)
public class DebeziumCdcIndexProcessorApplication {
    public static void main(String... args) {
        SpringApplication.run(DebeziumCdcIndexProcessorApplication.class, args);
    }
}
```  

- `IndexProperties`

```java
@Data
@ConfigurationProperties("index")
public class IndexProperties {
    private String name;
    private boolean applyDate;
}
```  

- `DeeziumCdcIndexProcessor`

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class DebeziumCdcIndexProcessor {
    private final IndexProperties indexProperties;

    @Bean
    public Function<Message<Map<String, Object>>, Message<Map<String, Object>>> processor() {
        return mapMessage -> {
            log.info("message : {}", mapMessage);
            Map<String, Object> payload = mapMessage.getPayload();
            MessageHeaders headers = mapMessage.getHeaders();
            Map<String, Object> newHeaders = new HashMap<>();
            StringBuilder indexName = new StringBuilder(this.indexProperties.getName());
            LocalDateTime timestamp = LocalDateTime.ofInstant(Instant.ofEpochMilli(headers.getTimestamp()), ZoneId.of("UTC"));

            // create elasticsearch rolling index with date
            if (this.indexProperties.isApplyDate()) {
                indexName
                        .append("-")
                        .append(timestamp
                                .atZone(ZoneId.of("UTC"))
                                .withZoneSameInstant(ZoneId.of("Asia/Seoul"))
                                .format(DateTimeFormatter.ofPattern("yyyy.MM.dd")));
            }

            // apply index name
            if (StringUtils.hasText(indexName)) {
                newHeaders.put("INDEX_NAME", indexName);
            }

            // create timestamp for elasticsearch
            payload.put("timestamp", timestamp + "Z");

            return MessageBuilder
                    .withPayload(payload)
                    .copyHeaders(headers)
                    .copyHeaders(newHeaders)
                    .build();
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
          brokers: localhost:9092
      function:
        bindings:
          processor-in-0: input
          processor-out-0: output
      bindings:
        input:
          destination: output
        output:
          destination: input
```  

- `DebeziumCdcIndexProcessorTest`

```java
@SpringBootTest(properties = {"index.name=test", "index.apply-date=true"})
@Import(TestChannelBinderConfiguration.class)
public class DebeziumCdcIndexProcessorTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private InputDestination inputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void test() {
        Map<String, Object> payload = Map.of(
                "schema", Map.of("key1", "value1"),
                "payload", Map.of("key1", "value1")
        );
        Map<String, Object> headers = Map.of(
                "timestamp", System.currentTimeMillis()
        );
        Message<?> sourceMessage = this.converter.toMessage(payload, new MessageHeaders(headers));

        inputDestination.send(sourceMessage, "output");

        Message<byte[]> processMessage = this.outputDestination.receive(10000, "input");
        Map<String, Object> result = (Map<String, Object>) this.converter
                .fromMessage(processMessage, Map.class);

        assertThat(processMessage.getHeaders().get("INDEX_NAME").toString(), is("test-" + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy.MM.dd"))));
    }
}
```  

- 빌드, 푸시

```bash
$ ./gradlew jibDockerBuild
$ docker tag debezium-cdc-index-processor:v1 windowforsun/debezium-cdc-index-processor:v1
$ docker push windowforsun/debezium-cdc-index-processor:v1
```  

### 데이터 구성
`CDC` 예제 진행에 사용할 `MySQL` 은
[SCDF with MySQL]({{site.baseurl}}{% link _posts/spring/2023-06-04-spring-practice-spring-cloud-data-flow-mysql.md %})
을 바탕으로 환경 구성을 했다면 실행중에 있는 `MySQL` 에 `CDC` 용 데이터베이스와 테이블을 생성해 사용한다. 
그리고 데이터는 
[Kafka Connect CDC 데이터 구성](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-connect-debezium-mysql-cdc-source-connector/#%EC%84%A4%EC%A0%95-%EB%B0%8F-%ED%85%8C%EC%8A%A4%ED%8A%B8)
와 동일한 데이터베이스, 테이블, 데이터로 진행 한다.  



### Stream 구성 
스트림 구성에 사용할 `Stream Application` 은 아래와 같다. 

spring-cloud-data-flow-debezium-cdc-1.png

애플리케이션 페이지에서 보이지 않는다면 `ADD APPLICATION` 을 통해 애플리케이션을 추가해준다. 

- `debezium-source`(Source) : `docker:springcloudstream/debezium-source-kafka:4.0.0-RC1`
- `debezium-cdc-index-processor`(Processor) : `docker:windowforsun/debezium-cdc-index-processor:v1`
- `elasticsearch`(Sink) : `docker:springcloudstream/elasticsearch-sink-kafka:4.0.0-RC1`

















```properties
# ???????? Caused by: java.lang.ClassCastException: class java.lang.StringBuilder cannot be cast to class java.lang.String

app.debezium-source.debezium.properties.database.server.id=111111
app.debezium-source.debezium.properties.connector.class=io.debezium.connector.mysql.MySqlConnector
app.debezium-source.debezium.properties.topic.prefix=my-debezium-cdc
app.debezium-source.debezium.properties.name=my-debezium-cdc
app.debezium-source.debezium.properties.database.server.name=mysql
app.debezium-source.debezium.properties.schema=true
app.debezium-source.debezium.properties.key.converter.schemas.enable=true
app.debezium-source.debezium.properties.value.converter.schemas.enable=true
app.debezium-source.debezium.properties.include.schema.changes=false
app.debezium-source.debezium.properties.database.user=root
app.debezium-source.debezium.properties.database.password=root
app.debezium-source.debezium.properties.database.hostname=${MYSQL_SERVICE_HOST}
app.debezium-source.debezium.properties.database.port=${MYSQL_SERVICE_PORT}
app.debezium-source.debezium.properties.database.include.list=user
app.debezium-source.debezium.properties.table.include.list=user.user_account
app.debezium-source.debezium.properties.database.allowPublicKeyRetrieval=true
app.debezium-source.debezium.properties.database.characterEncoding=utf8
app.debezium-source.debezium.properties.database.connectionTimeZone=Asia/Seoul
app.debezium-source.debezium.properties.database.useUnicode=true
app.debezium-source.debezium.properties.bootstrap.servers=kafka-service:9092
app.debezium-source.debezium.spring.kafka.bootstrap.servers=kafka-service:9092
app.debezium-source.debezium.bootstrap.servers=kafka-service:9092
app.debezium-source.debezium.properties.offset.storage=org.apache.kafka.connect.storage.MemoryOffsetBackingStore
app.debezium-source.debezium.properties.database.history=io.debezium.relational.history.MemoryDatabaseHistory
app.debezium-source.debezium.properties.schema.history.internal=io.debezium.relational.history.MemorySchemaHistory
app.elasticsearch.spring.data.elasticsearch.client.reactive.endpoints=${ES_SINGLE_SERVICE_HOST}:${ES_SINGLE_PORT_9200_TCP_PORT}
app.elasticsearch.spring.data.elasticsearch.client.reactive.password=irteam123!@#
app.elasticsearch.spring.data.elasticsearch.client.reactive.username=irteam
app.elasticsearch.spring.elasticsearch.uris=http://${ES_SINGLE_SERVICE_HOST}:${ES_SINGLE_PORT_9200_TCP_PORT}
app.elasticsearch.elasticsearch.consumer.index=ignored-index
app.elasticsearch.elasticsearch.consumer.id=headers.id
app.debezium-cdc-index-processor.index.name=my-debezium-cdc-rolling-index
app.debezium-cdc-index-processor.index.applyIndex=true
deployer.debezium-cdc-index-processor.kubernetes.readiness-http-probe-path=/actuator/health
deployer.debezium-cdc-index-processor.kubernetes.readiness-http-probe-port=8080
deployer.*.kubernetes.limits.cpu=3
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default
version.debezium-cdc-index-processor=v1
version.elasticsearch=4.0.0-RC1
version.debezium-source=4.0.0-RC1
```

```


./gradlew debezium-cdc-index-processor:jibDockerBuild

docker tag debezium-cdc-index-processor:v1 windowforsun/debezium-cdc-index-processor:v1

docker push windowforsun/debezium-cdc-index-processor:v1

```


---  
## Reference
[]()  