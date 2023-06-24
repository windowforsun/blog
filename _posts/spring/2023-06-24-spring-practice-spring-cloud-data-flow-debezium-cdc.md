--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Data Flow(SCDF) MySQL 연동"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'SCDF 의 데이터를 외부 저장소 MySQL 과 연동해 저장하고 관리해보자'
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
    - MySQL
toc: true
use_math: true
---  


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