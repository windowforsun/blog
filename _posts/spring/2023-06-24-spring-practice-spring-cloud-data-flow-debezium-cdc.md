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

정리하면 `Index Processor` 인 `debezium-index-processor` 는 아래와 같은 내용을 수행한다. 

- `Debezium CDC Source` 의 `DB`, `Table` 기반 동적  `Elasticsearch Index name` 설정 제공
- 날짜기반 `Rolling index` 제공
- `Elasticsearch Index` 에서 사용될 `Time field` 인 `timestamp` 필드 추가

  
<br>

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
    private String prefix;
    private boolean applyDate;
}
```  

- `DebeziumMessage`
  - `debezium-source` 에서 전달되는 메시지 대략적인 형태를 클래스로 정의

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DebeziumMessage {
    private Map<String, Object> schema;
    private Map<String, Object> payload;
    // 커스텀 필드, ES 에서 타임 필드용
    private String timestamp;
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
    public Function<Message<DebeziumMessage>, Message<DebeziumMessage>> processor() {
        return mapMessage -> {
            log.info("origin message : {}", mapMessage);
            DebeziumMessage payload = mapMessage.getPayload();
            MessageHeaders headers = mapMessage.getHeaders();
            Map<String, Object> newHeaders = new HashMap<>();
            LocalDateTime timestamp = LocalDateTime.ofInstant(Instant.ofEpochMilli(headers.getTimestamp()), ZoneId.of("UTC"));
            Map<String, Object> payloadSource = (Map<String, Object>) payload.getPayload().getOrDefault("source", Collections.emptyMap());
            String db = (String) payloadSource.getOrDefault("db", "");
            String table = (String) payloadSource.getOrDefault("table", "");
            StringBuilder indexName = new StringBuilder();

            // index name by prefix
            if (StringUtils.hasText(this.indexProperties.getPrefix())) {
                indexName
                        .append(this.indexProperties.getPrefix())
                        .append("-");
            }

            // index name by db, table
            indexName
                    .append(db)
                    .append("-")
                    .append(table);

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
                newHeaders.put("INDEX_NAME", indexName.toString());
            }

            // create time field for elasticsearch
            payload.setTimestamp(timestamp + "Z");

            Message<DebeziumMessage> transMessage = MessageBuilder
                    .withPayload(payload)
                    .copyHeaders(headers)
                    .copyHeaders(newHeaders)
                    .build();
            log.info("trans message : {}", transMessage);

            return transMessage;
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
@SpringBootTest(properties = {"index.prefix=test", "index.apply-date=true"})
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
        DebeziumMessage message = DebeziumMessage.builder()
                .schema(Map.of("key1", "value1"))
                .payload(Map.of(
                        "key1", "value1", "source",
                        Map.of("db", "testDb", "table", "testTable"))
                )
                .build();
        Map<String, Object> headers = Map.of(
                "timestamp", System.currentTimeMillis()
        );
        Message<?> sourceMessage = this.converter.toMessage(message, new MessageHeaders(headers));

        inputDestination.send(sourceMessage, "output");

        Message<byte[]> processMessage = this.outputDestination.receive(10000, "input");
        Map<String, Object> result = (Map<String, Object>) this.converter
                .fromMessage(processMessage, Map.class);

        assertThat(processMessage.getHeaders().get("INDEX_NAME").toString(), is("test-testDb-testTable-" + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy.MM.dd"))));
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


```bash
mysql> create database user;
Query OK, 1 row affected (0.02 sec)

mysql> use user;
Database changed

mysql> create table user_account (
    ->    uid int,
    ->    name varchar(255)
    -> );
Query OK, 0 rows affected (0.04 sec)

mysql> create table user_role (
    -> account_id int,
    -> roll varchar(255)
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> insert into user_account(uid, name) values(1, 'jack');
Query OK, 1 row affected (0.01 sec)

mysql> insert into user_role(account_id, role) values(1, 'normal');
Query OK, 1 row affected (0.01 sec)

mysql> select * from user_account;
+------+------+
| uid  | name |
+------+------+
|    1 | jack |
+------+------+
1 row in set (0.00 sec)

mysql> select * from user_role;
+------------+--------+
| account_id | role  |
+------------+--------+
|          1 | normal |
+------------+--------+
1 row in set (0.00 sec)
```



### Stream 구성 
스트림 구성에 사용할 `Stream Application` 은 아래와 같다. 

spring-cloud-data-flow-debezium-cdc-1.png

애플리케이션 페이지에서 보이지 않는다면 `ADD APPLICATION` 을 통해 애플리케이션을 추가해준다. 

- `debezium-source`(Source) : `docker:springcloudstream/debezium-source-kafka:4.0.0-RC1`
- `debezium-cdc-index-processor`(Processor) : `docker:windowforsun/debezium-cdc-index-processor:v1`
- `elasticsearch`(Sink) : `docker:springcloudstream/elasticsearch-sink-kafka:4.0.0-RC1`


<details><summary>Elasticsearch Kubernetes 구성</summary>
<div markdown="1">


```yaml
# es-deploy.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: es-single
  labels:
    app: es-single

data:
  elasticsearch.yml: |-
    xpack.security.enabled=false
    node.name=${NODE_NAME}
    cluster.name=${CLUSTER_NAME}
    discovery.type=single-node

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: es-deployment
  labels:
    app: es-single
spec:
  replicas: 1
  selector:
    matchLabels:
      app: es-single
  template:
    metadata:
      labels:
        app: es-single
    spec:
      containers:
        - name: es-single
          image: docker.elastic.co/elasticsearch/elasticsearch:7.15.1
          ports:
            - name: rest
              containerPort: 9200
            - name: transport
              containerPort: 9300
          env:
            - name: CLUSTER_NAME
              value: es-single
            - name: NODE_NAME
              value: "es-single"
            - name: discovery.type
              value: single-node
            - name: xpack.security.enabled
              value: "false"
            - name: "ES_JAVA_OPTS"
              value: "-Xms512m -Xmx512m"
---
apiVersion: v1
kind: Service
metadata:
  name: es-single
spec:
  selector:
    app: es-single
  ports:
    - protocol: TCP
      port: 9200
      targetPort: 9200
```  

```yaml
# kibana-deploy.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana
  labels:
    app: kibana

data:
  kibana.yml: |-
    server.host: 0.0.0.0

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: kibana
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:7.15.1
          ports:
            - name: view
              containerPort: 5601
          volumeMounts:
            - name: config
              mountPath: /usr/share/kibana/config/kibana.yml
              readOnly: true
              subPath: kibana.yml
          env:
            - name: ELASTICSEARCH_HOSTS
              value: 'http://es-single:9200'
      volumes:
        - name: config
          configMap:
            name: kibana
            items:
              - key: kibana.yml
                path: kibana.yml

---
apiVersion: v1
kind: Service
metadata:
  name: kibana
spec:
  selector:
    app: kibana
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 5601
      targetPort: 5601
```


</div>
</details>

필요한 애플리케이션 추가를 완료했으면, `Streams` 에서 `CREATE STREAM` 을 통해 아래와 같이 `Debezium CDC` 를 위한 스트림을 생성한다.  

spring-cloud-data-flow-debezium-cdc-2.png

그리고 아래 `CREATE STREAM` 을 눌러 `scdf-debeizum-cdc` 라는 이름으로 스트림을 생성한다. 

spring-cloud-data-flow-debezium-cdc-3.png

스트림이 생성됐으면 `DEPLOY STREAM` 을 눌러 배포 설정을 해주는데, 
배포 설정은 `FreeText` 를 사용하고 그 내용은 아래와 같다. 


```properties
# debezium-source 설정
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

# debezium-cdc-index-processor 설정
app.debezium-cdc-index-processor.index.prefix=debezium-test
app.debezium-cdc-index-processor.index.applyDate=true

# elasticsearch 설정
app.elasticsearch.spring.data.elasticsearch.client.reactive.endpoints=${ES_SINGLE_SERVICE_HOST}:${ES_SINGLE_PORT_9200_TCP_PORT}
app.elasticsearch.spring.data.elasticsearch.client.reactive.password=irteam123!@#
app.elasticsearch.spring.data.elasticsearch.client.reactive.username=irteam
app.elasticsearch.spring.elasticsearch.uris=http://${ES_SINGLE_SERVICE_HOST}:${ES_SINGLE_PORT_9200_TCP_PORT}
app.elasticsearch.elasticsearch.consumer.index=ignored-index
app.elasticsearch.elasticsearch.consumer.id=headers.id

# 배포 설정
deployer.debezium-cdc-index-processor.kubernetes.readiness-http-probe-path=/actuator/health
deployer.debezium-cdc-index-processor.kubernetes.readiness-http-probe-port=8080
deployer.*.kubernetes.limits.cpu=3
deployer.*.kubernetes.limits.memory=1000Mi
version.debezium-cdc-index-processor=v1
version.elasticsearch=4.0.0-RC1
version.debezium-source=4.0.0-RC1
spring.cloud.dataflow.skipper.platformName=default
```  

`DEPLOY STREAM` 을 눌러 배포를 시작한다. 
그리고 스트림이 `DEPLOYED` 상태가 될때까지 가디란다.  

이후 `SCDF` 스트림 상세 정보에서 `debezium-cdc-index-processor` 로그를 확인하면, 
아래와 같이 2개의 `MySQL` 로우에 대한 출력이 있는 것을 확인 할 수 있다.  

```
2023-07-01 16:11:48.635  INFO 6 --- [container-0-C-1] c.w.s.p.DebeziumCdcIndexProcessor        : origin message : GenericMessage [payload=DebeziumMessage(schema={type=struct, fields=[{type=struct, fields=[{type=int32, optional=false, field=uid}, {type=string, optional=true, field=name}], optional=true, name=my-debezium-cdc.user.user_account.Value, field=before}, {type=struct, fields=[{type=int32, optional=false, field=uid}, {type=string, optional=true, field=name}], optional=true, name=my-debezium-cdc.user.user_account.Value, field=after}, {type=struct, fields=[{type=string, optional=false, field=version}, {type=string, optional=false, field=connector}, {type=string, optional=false, field=name}, {type=int64, optional=false, field=ts_ms}, {type=string, optional=true, name=io.debezium.data.Enum, version=1, parameters={allowed=true,last,false,incremental}, default=false, field=snapshot}, {type=string, optional=false, field=db}, {type=string, optional=true, field=sequence}, {type=string, optional=true, field=table}, {type=int64, optional=false, field=server_id}, {type=string, optional=true, field=gtid}, {type=string, optional=false, field=file}, {type=int64, optional=false, field=pos}, {type=int32, optional=false, field=row}, {type=int64, optional=true, field=thread}, {type=string, optional=true, field=query}], optional=false, name=io.debezium.connector.mysql.Source, field=source}, {type=string, optional=false, field=op}, {type=int64, optional=true, field=ts_ms}, {type=struct, fields=[{type=string, optional=false, field=id}, {type=int64, optional=false, field=total_order}, {type=int64, optional=false, field=data_collection_order}], optional=true, name=event.block, version=1, field=transaction}], optional=false, name=my-debezium-cdc.user.user_account.Envelope, version=1}, payload={before=null, after={uid=1, name=jack}, source={version=2.3.0.CR1, connector=mysql, name=my-debezium-cdc, ts_ms=1688195507000, snapshot=first, db=user, sequence=null, table=user_account, server_id=0, gtid=null, file=mysql-bin.000003, pos=37100403, row=0, thread=null, query=null}, op=r, ts_ms=1688195507990, transaction=null}, timestamp=null), headers={debezium_key=[B@32e84ff7, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, kafka_receivedMessageKey=[B@261933e0, kafka_receivedTopic=scdf-debezium-cdc.debezium-source, target-protocol=kafka, kafka_offset=2, debezium_destination=my-debezium-cdc.user.user_account, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@70c3206b, id=161f1a60-b2cd-5a1a-b1a2-cebc4b2d8c9c, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTimestamp=1688195508538, kafka_groupId=scdf-debezium-cdc, timestamp=1688195508635}]
2023-07-01 16:11:48.642  INFO 6 --- [container-0-C-1] c.w.s.p.DebeziumCdcIndexProcessor        : trans message : GenericMessage [payload=DebeziumMessage(schema={type=struct, fields=[{type=struct, fields=[{type=int32, optional=false, field=uid}, {type=string, optional=true, field=name}], optional=true, name=my-debezium-cdc.user.user_account.Value, field=before}, {type=struct, fields=[{type=int32, optional=false, field=uid}, {type=string, optional=true, field=name}], optional=true, name=my-debezium-cdc.user.user_account.Value, field=after}, {type=struct, fields=[{type=string, optional=false, field=version}, {type=string, optional=false, field=connector}, {type=string, optional=false, field=name}, {type=int64, optional=false, field=ts_ms}, {type=string, optional=true, name=io.debezium.data.Enum, version=1, parameters={allowed=true,last,false,incremental}, default=false, field=snapshot}, {type=string, optional=false, field=db}, {type=string, optional=true, field=sequence}, {type=string, optional=true, field=table}, {type=int64, optional=false, field=server_id}, {type=string, optional=true, field=gtid}, {type=string, optional=false, field=file}, {type=int64, optional=false, field=pos}, {type=int32, optional=false, field=row}, {type=int64, optional=true, field=thread}, {type=string, optional=true, field=query}], optional=false, name=io.debezium.connector.mysql.Source, field=source}, {type=string, optional=false, field=op}, {type=int64, optional=true, field=ts_ms}, {type=struct, fields=[{type=string, optional=false, field=id}, {type=int64, optional=false, field=total_order}, {type=int64, optional=false, field=data_collection_order}], optional=true, name=event.block, version=1, field=transaction}], optional=false, name=my-debezium-cdc.user.user_account.Envelope, version=1}, payload={before=null, after={uid=1, name=jack}, source={version=2.3.0.CR1, connector=mysql, name=my-debezium-cdc, ts_ms=1688195507000, snapshot=first, db=user, sequence=null, table=user_account, server_id=0, gtid=null, file=mysql-bin.000003, pos=37100403, row=0, thread=null, query=null}, op=r, ts_ms=1688195507990, transaction=null}, timestamp=2023-07-01T07:11:48.635Z), headers={debezium_key=[B@32e84ff7, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, INDEX_NAME=debezium-test-user-user_account-2023.07.01, kafka_receivedMessageKey=[B@261933e0, kafka_receivedTopic=scdf-debezium-cdc.debezium-source, target-protocol=kafka, kafka_offset=2, debezium_destination=my-debezium-cdc.user.user_account, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@70c3206b, id=7345430a-b829-8745-2e03-816eff17007b, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTimestamp=1688195508538, kafka_groupId=scdf-debezium-cdc, timestamp=1688195508642}]
2023-07-01 16:11:48.666  INFO 6 --- [container-0-C-1] c.w.s.p.DebeziumCdcIndexProcessor        : origin message : GenericMessage [payload=DebeziumMessage(schema={type=struct, fields=[{type=struct, fields=[{type=int32, optional=false, field=account_id}, {type=string, optional=false, field=user_role}], optional=true, name=my-debezium-cdc.user.user_role.Value, field=before}, {type=struct, fields=[{type=int32, optional=false, field=account_id}, {type=string, optional=false, field=user_role}], optional=true, name=my-debezium-cdc.user.user_role.Value, field=after}, {type=struct, fields=[{type=string, optional=false, field=version}, {type=string, optional=false, field=connector}, {type=string, optional=false, field=name}, {type=int64, optional=false, field=ts_ms}, {type=string, optional=true, name=io.debezium.data.Enum, version=1, parameters={allowed=true,last,false,incremental}, default=false, field=snapshot}, {type=string, optional=false, field=db}, {type=string, optional=true, field=sequence}, {type=string, optional=true, field=table}, {type=int64, optional=false, field=server_id}, {type=string, optional=true, field=gtid}, {type=string, optional=false, field=file}, {type=int64, optional=false, field=pos}, {type=int32, optional=false, field=row}, {type=int64, optional=true, field=thread}, {type=string, optional=true, field=query}], optional=false, name=io.debezium.connector.mysql.Source, field=source}, {type=string, optional=false, field=op}, {type=int64, optional=true, field=ts_ms}, {type=struct, fields=[{type=string, optional=false, field=id}, {type=int64, optional=false, field=total_order}, {type=int64, optional=false, field=data_collection_order}], optional=true, name=event.block, version=1, field=transaction}], optional=false, name=my-debezium-cdc.user.user_role.Envelope, version=1}, payload={before=null, after={account_id=1, user_role=normal}, source={version=2.3.0.CR1, connector=mysql, name=my-debezium-cdc, ts_ms=1688195507000, snapshot=last, db=user, sequence=null, table=user_role, server_id=0, gtid=null, file=mysql-bin.000003, pos=37100403, row=0, thread=null, query=null}, op=r, ts_ms=1688195507997, transaction=null}, timestamp=null), headers={debezium_key=[B@70429187, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, kafka_receivedMessageKey=[B@7f2fcf4f, kafka_receivedTopic=scdf-debezium-cdc.debezium-source, target-protocol=kafka, kafka_offset=3, debezium_destination=my-debezium-cdc.user.user_role, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@70c3206b, id=2078290d-6db4-1eea-5fa9-a927d1bd5aa2, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTimestamp=1688195508557, kafka_groupId=scdf-debezium-cdc, timestamp=1688195508666}]
2023-07-01 16:11:48.667  INFO 6 --- [container-0-C-1] c.w.s.p.DebeziumCdcIndexProcessor        : trans message : GenericMessage [payload=DebeziumMessage(schema={type=struct, fields=[{type=struct, fields=[{type=int32, optional=false, field=account_id}, {type=string, optional=false, field=user_role}], optional=true, name=my-debezium-cdc.user.user_role.Value, field=before}, {type=struct, fields=[{type=int32, optional=false, field=account_id}, {type=string, optional=false, field=user_role}], optional=true, name=my-debezium-cdc.user.user_role.Value, field=after}, {type=struct, fields=[{type=string, optional=false, field=version}, {type=string, optional=false, field=connector}, {type=string, optional=false, field=name}, {type=int64, optional=false, field=ts_ms}, {type=string, optional=true, name=io.debezium.data.Enum, version=1, parameters={allowed=true,last,false,incremental}, default=false, field=snapshot}, {type=string, optional=false, field=db}, {type=string, optional=true, field=sequence}, {type=string, optional=true, field=table}, {type=int64, optional=false, field=server_id}, {type=string, optional=true, field=gtid}, {type=string, optional=false, field=file}, {type=int64, optional=false, field=pos}, {type=int32, optional=false, field=row}, {type=int64, optional=true, field=thread}, {type=string, optional=true, field=query}], optional=false, name=io.debezium.connector.mysql.Source, field=source}, {type=string, optional=false, field=op}, {type=int64, optional=true, field=ts_ms}, {type=struct, fields=[{type=string, optional=false, field=id}, {type=int64, optional=false, field=total_order}, {type=int64, optional=false, field=data_collection_order}], optional=true, name=event.block, version=1, field=transaction}], optional=false, name=my-debezium-cdc.user.user_role.Envelope, version=1}, payload={before=null, after={account_id=1, user_role=normal}, source={version=2.3.0.CR1, connector=mysql, name=my-debezium-cdc, ts_ms=1688195507000, snapshot=last, db=user, sequence=null, table=user_role, server_id=0, gtid=null, file=mysql-bin.000003, pos=37100403, row=0, thread=null, query=null}, op=r, ts_ms=1688195507997, transaction=null}, timestamp=2023-07-01T07:11:48.666Z), headers={debezium_key=[B@70429187, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, INDEX_NAME=debezium-test-user-user_role-2023.07.01, kafka_receivedMessageKey=[B@7f2fcf4f, kafka_receivedTopic=scdf-debezium-cdc.debezium-source, target-protocol=kafka, kafka_offset=3, debezium_destination=my-debezium-cdc.user.user_role, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@70c3206b, id=0d11fd6d-7f59-f436-2031-57efd7696deb, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTimestamp=1688195508557, kafka_groupId=scdf-debezium-cdc, timestamp=1688195508667}]
```  

`Kafka` 에 생성된 토픽을 확인하면 아래와 같이, 
`scdf-debezium-cdc` 스트림에서 사용하는 2개의 토픽이 생성된 것을 확인 할 수 있다.  

```bash
$ kafka-topics --bootstrap-server localhost:29092 --list
__consumer_offsets
scdf-debezium-cdc.debezium-cdc-index-processor
scdf-debezium-cdc.debezium-source
```  

```bash
.. debezium-source(producer) - debezium-cdc-index-processor (consumer) ..
$ kafka-console-consumer --bootstrap-server localhost:29092 --topic scdf-debezium-cdc.debezium-source --from-beginning
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"uid"},{"type":"string","optional":true,"field":"name"}],"optional":true,"name":"my-debezium-cdc.user.user_account.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"uid"},{"type":"string","optional":true,"field":"name"}],"optional":true,"name":"my-debezium-cdc.user.user_account.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"my-debezium-cdc.user.user_account.Envelope","version":1},"payload":{"before":null,"after":{"uid":1,"name":"jack"},"source":{"version":"2.3.0.CR1","connector":"mysql","name":"my-debezium-cdc","ts_ms":1688195507000,"snapshot":"first","db":"user","sequence":null,"table":"user_account","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":37100403,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1688195507990,"transaction":null}}
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"account_id"},{"type":"string","optional":false,"field":"user_role"}],"optional":true,"name":"my-debezium-cdc.user.user_role.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"account_id"},{"type":"string","optional":false,"field":"user_role"}],"optional":true,"name":"my-debezium-cdc.user.user_role.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"my-debezium-cdc.user.user_role.Envelope","version":1},"payload":{"before":null,"after":{"account_id":1,"user_role":"normal"},"source":{"version":"2.3.0.CR1","connector":"mysql","name":"my-debezium-cdc","ts_ms":1688195507000,"snapshot":"last","db":"user","sequence":null,"table":"user_role","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":37100403,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1688195507997,"transaction":null}}

.. debezium-cdc-index-processor(producer) - elasticsearch(consumer)
$ kafka-console-consumer --bootstrap-server localhost:29092 --topic scdf-debezium-cdc.debezium-cdc-index-processor --from-beginning
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"uid"},{"type":"string","optional":true,"field":"name"}],"optional":true,"name":"my-debezium-cdc.user.user_account.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"uid"},{"type":"string","optional":true,"field":"name"}],"optional":true,"name":"my-debezium-cdc.user.user_account.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"my-debezium-cdc.user.user_account.Envelope","version":1},"payload":{"before":null,"after":{"uid":1,"name":"jack"},"source":{"version":"2.3.0.CR1","connector":"mysql","name":"my-debezium-cdc","ts_ms":1688195507000,"snapshot":"first","db":"user","sequence":null,"table":"user_account","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":37100403,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1688195507990,"transaction":null},"timestamp":"2023-07-01T07:11:48.635Z"}
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"account_id"},{"type":"string","optional":false,"field":"user_role"}],"optional":true,"name":"my-debezium-cdc.user.user_role.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"account_id"},{"type":"string","optional":false,"field":"user_role"}],"optional":true,"name":"my-debezium-cdc.user.user_role.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"my-debezium-cdc.user.user_role.Envelope","version":1},"payload":{"before":null,"after":{"account_id":1,"user_role":"normal"},"source":{"version":"2.3.0.CR1","connector":"mysql","name":"my-debezium-cdc","ts_ms":1688195507000,"snapshot":"last","db":"user","sequence":null,"table":"user_role","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":37100403,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1688195507997,"transaction":null},"timestamp":"2023-07-01T07:11:48.666Z"}
```  

이제 마지막으로 구성한 스트림의 엔드인 `Elasticsearch` 에 설정한 대로 인덱스와 데이터가 잘 들어갔는지 확인만 하면 된다. 
`Kubernetes` 에 구성한 `Elasticsearch` 확인은 `Kibana` 를 통해 진행하는데, `Kibana` 접속은 `Service` 를 조회한 뒤 `EXTERNAL-IP` 를 통해 접속 할 수 있다.  

```bash
$ kubectl get svc
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
kibana                                             LoadBalancer   xxx.xx.xxx.xxx   xx.xxx.xx.xxx   5601:45719/TCP   6d22h
```  

웹 브라우저에 `http://<kibana-external-ip>:5601` 로 접속한다. 
그러면 `Kibana` 홈 화면이 나오는데 좌측 메뉴 `Management > Stack Management` 로 들어간다. 
그리고 좌측 메뉴 `Kibana > Index patterns` 에 들어가고 `Create index pattern` 을 누른다.  

`Name` 에 `app.debezium-cdc-index-processor.index.prefix=debezium-test` 입력한 값인, 
`debezium-test` 를 입력해주면 아래와 같이 2개 테이블에 대한 2개의 인덱스가 날짜기반 `Rolling Index` 가 된 것을 확인 할 수 있다.  

spring-cloud-data-flow-debezium-cdc-4.png

생성된 각 인덱스의 필드와 대상 테이블이 다르기 때문에 각각 따로 인덱스 패턴을 지정해 주고, `Timestamp field` 또한 별도로 추가한 `timestamp` 필드로 지정한다.
모두 완료됐으면 하단에 `Create index pattern` 을 눌러 생성을 완료한다.

spring-cloud-data-flow-debezium-cdc-5.png

spring-cloud-data-flow-debezium-cdc-6.png
 

다시 좌측 메뉴에서 `Analytics > Discover` 로 접속한 뒤, 
생성한 인덱스 패턴이 잘 설정 됐는지 확인하면 아래와 같다.  

spring-cloud-data-flow-debezium-cdc-7.png

spring-cloud-data-flow-debezium-cdc-8.png



---  
## Reference
[]()  