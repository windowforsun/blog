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

- `debezium-source` : 
- `debezium-index-processor` : 
- `elastiacsearch-sink` : 









```properties
app.debezium-source.debezium.properties.database.server.id=12345
app.debezium-source.debezium.properties.connector.class=io.debezium.connector.mysql.MySqlConnector
app.debezium-source.debezium.properties.topic.prefix=my-debezium-cdc
app.debezium-source.debezium.properties.name=my-debezium-cdc
app.debezium-source.debezium.properties.database.server.name=mysql
app.debezium-source.debezium.properties.schema=true
app.debezium-source.debezium.properties.key.converter.schemas.enable=true
app.debezium-source.debezium.properties.value.converter.schemas.enable=true
app.debezium-source.debezium.properties.include.schema.changes=false

app.debezium-source.debezium.properties.database.user=weather
app.debezium-source.debezium.properties.database.password=7dnjf15dlf
app.debezium-source.debezium.properties.database.hostname=10.113.133.163
app.debezium-source.debezium.properties.database.port=13306
app.debezium-source.debezium.properties.database.include.list=weather
app.debezium-source.debezium.properties.table.include.list=weather.wt_wetr_warn_timeline

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
version.debezium-source=4.0.0-M2
```

```


./gradlew debezium-cdc-index-processor:jibDockerBuild

docker tag debezium-cdc-index-processor:v1 windowforsun/debezium-cdc-index-processor:v1

docker push windowforsun/debezium-cdc-index-processor:v1

```


---  
## Reference
[]()  