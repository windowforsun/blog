--- 
layout: single
classes: wide
title: "[Kafka] "
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
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Topics
`Kafka Streams` 애플리케이션은 `Kafka Broker` 의 다양한 `Topic` 들과 지속적으로 데이터를 주고 받으면 데이터 스트림을 구현한다. 
이는 사용자가 필요에 의해 정의한 명시적인 `Topic` 도 포함되지만 `Kafka Streams` 에서 필요에 의해서 생성되는 `Topic` 도 포함된다. 
이번 포스팅에서는 `Kafka Streams` 애플리케이션을 구현하고 사용할 때 어떠한 유형의 `Topic` 들이 사용되고 필요하지에 대해 알아본다. 
크게 아래 2가지의 `Topic` 으로 구분될 수 있다.  

- `User Topics`
- `Internal Topics`


### User Topics
`User Topics` 는 `Kafka Streams Application` 에서 외적으로 사용하는 `Topic` 으로, 
애플리케이션에 의해 읽히거나 기록되는 토픽을 의미한다. 
즉 사용자의 필요에 따라 생성되고 관리되는 `Topic` 이라고 할 수 있다. 
그 상세한 종류는 아래와 같다.  

- `Input Topics` : 애플리케이션이 데이터를 읽어오는 소스 토픽이다. 일반적으로 외부 시스템이나 다른 애프릴케이션에서 생성한 데이터가 포함된다. 스트림 애플리케이션의 시작점 역할을 하게 된다. (e.g. `StreamsBuilder.stream()`, `StreamsBuilder.table()`, `Topology.addSource()`)
- `Output Topics` : 애플리케이션이 처리 결과를 쓰는 목적지 토픽이다. 처리된 데이터나 분석 결과가 저장된다. 다른 시스템이나 애플리케이션에서 이 토픽 데이터를 소비할 수 있다. (e.g. `KStream.to()`, `KTable.to()`, `Topology.addSink()`)
- `Intermediate Topics` : 애플리케이션 내에서 입력/출력 모두로 사용되는 토픽이다. 복잡한 처리 파이프라인에서 중간 결과를 저장하는 데 사용된다. 다단계 처리나 여러 애플리케이션 간의 데이터 전달에 활용할 수도 있다. (e.g. `KStream.through()`)

> `Kafka Streams 3.0` 이하인 경우 `KStream.through()` 를 사용해 `Intermediate Topics` 를 생성해 사용할 수 있었다. 
> 하지만 `3.0` 버전 이후 부터는 `Depreated` 되어 관련된 처리는 `KStream.repartition()` 이 새로 생겼다. 
> 그 상세한 내용을 정리하면 아래와 같다. 
> 간략한 내용은 두 가지 모두 사용 목적은 `중간 처리 단계 데이터 저장` 이지만, 관리 방식이 좀 더 용의하게 변경됐다. 
> 
> | 특성               | `repartition()` 토픽             | `through()` 토픽                |
> |-------------------|----------------------------------|---------------------------------|
> | **관리 방식**      | 자동 관리                        | 수동 관리                       |
> | **토픽 종류**      | 내부 토픽 (internal topic)      | 사용자 토픽 (user topic)       |
> | **가시성**         | 내부적으로 숨겨짐                | 명시적으로 보임                 |
> | **네이밍**         | 자동 이름 생성                   | 개발자가 이름 지정             |
> | **역할**           | 중간 처리 단계의 데이터 저장    | 중간 처리 단계의 데이터 저장    |


`User Topics` 사용/관리에 권장하는 방법은 사전에 직접 생성하고 수동으로 관리해야 한다는 것이다. (`kafka-topics` 툴 사용 등)
즉 `Kafka Broker` 의 토픽 자동 생성 기능으로 생성하도록 하는 것은 권장하지 않는다는 의미한데, 그 이유는 아래와 같다.  

- `Kafka Broker` 에서 토픽 자동 생성이 비활성화 돼 있을 수 있다. 
- `auto.create.topics.enable=true` 로 설정된 경우 생성되는 토픽은 기본 토픽 설정을 바탕으로 구성된다. 기본 설정으로 토픽이 생성될 경우 필요한 설정과는 차이가 있으 수 있기 때문이다. 


### Internal Topics
`Internal Topics` 는 `Kafka Streams` 애플리케이션이 실행 중 내부적으로 필요에 의해 생성/사용하는 `Topic` 을 의미한다. 
`State Store` 의 `changelog` 가 이에 해당한다. 
이러한 `Internal Topics` 는 `Kafka Streams` 애플리케이션에 의해 생성되고, 내부에서만 사용하는 `Topic` 이다.  

> 설명하는 `Internal Topics` 는 `Kafka Streams` 애플리케이션 관점에 대한 내용이다. 
> `Kafka Broker` 관점에서는 `State Store` 의 `changelog` 토픽도 일반적인 토픽과 차이가 없다. 
> 즉 `Kafka Broker` 관점에서 `Internal Topics` 라는 것은 `__consumer_offsets`, `__transaction_state` 과 같은 토픽을 의미한다.  

`Kafka Broker` 에 보안 설정이 활성화 돼 있는 경우, `Kafka Streams` 애플리케이션은 `Internal Topics` 생성을 위해 
관리자 권한을 부여해야 한다.  

> `Kafka Streams` 애플리케이션에 관리자 권한을 부여하는 상세한 방법은 추후에 다시 알아보고 간단한 절차에 대해서만 알아본다. 
> - `ACL` 설정 : `streams-app` 사용자에게 모든 토픽에 대한 `Create`, `Write`, `Read`, `Describe` 권한을 부여한다. 
> 
> ```bash
> bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config admin.properties \
> --add --allow-principal User:streams-app \
> --operation Create --operation Write --operation Read --operation Describe \
> --topic '*' --resource-pattern-type prefixed
> ```  
> 
> - `Streams` 애플리케이션 설정
> 
> ```java
> properties.put("security.protocol", "SASL_PLAINTEXT");
> properties.put("sasl.mechanism", "PLAIN");
> properties.put("sasl.jaas.config", "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"streams-app\" password=\"streams-password\";");
> ```  
> 
> 자세한 내용은 [여기](https://kafka.apache.org/38/documentation/streams/developer-guide/security.html#streams-developer-guide-security)
> 에서 확인 가능하다. 

그리고 `Internal Topics` 의 이름은 `<application.id>-<operationName>-<suffix>` 와 같이 구성될 수 있다. 
이는 추후 릴리즈에서 변경될 수 있다. 
[여기](https://docs.confluent.io/platform/current/streams/developer-guide/config-streams.html#internal-topic-parameters)
내용 처럼 필요에 따라 `Internal Topics` 에 대한 설정은 `topic.` 이라는 `prefix` 를 사용해 여타 다른 `Topic` 들과 동일하게 설정할 수 있다.  


```java
// or StreamsConfig.TOPIC_PREFIX + StreamsConfig.TOPIC_PREFIX
props.put(StreamsConfig.topicPrefix(StreamsConfig.TOPIC_PREFIX), "internal-topics");
props.put(StreamsConfig.topicPrefix(StreamsConfig.REPLICATION_FACTOR_CONFIG), "10");
```  

`Kafka Broker` 의 `auto.create.topics.enable=false` 로 설정돼 토픽 자동 생성 기능이 비활성화 돼있더라도, 
`Kafka Streams` 의 `Internal Topics` 는 자동 생성이 가능하다. 
그 이유는 `Kafka Streams` 는 내부적으로 `Admin Client` 를 사용하기 때문이다. 
자동 생성을 할 때는 `StreamsConfig` 에 설정된 값을 바탕으로 `Topic` 을 생성한다. 
`Internal Topics` 를 생성할 때 기본으로 사용되는 주요 옵션은 아래와 같다.  


| 토픽 유형                           | 설정 | 값 |
|---------------------------------|------|-----|
| Internal Topics                 | message.timestamp.type | CreateTime |
| Repartition Topics              | 압축 정책 | delete |
|                                 | 보존 시간 | -1 (무한) |
| KeyValueStore Changelog Topics  | 압축 정책 | compact |
| WindowStore Changelog Topics    | 압축 정책 | delete,compact |
|                                 | 보존 시간 | 24시간 + 윈도우 저장소 설정 시간 |
| VersionedStore Changelog Topics | 정리 정책 | compact |
|                                 | min.compaction.lag.ms | 24시간 + 저장소의 historyRetentionMs 값 |


---  
## Reference
[Confluent Kafka Manage Kafka Streams Application Topics in Confluent Platform](https://docs.confluent.io/platform/current/streams/developer-guide/manage-topics.html)  
[Apache Kafka MANAGING STREAMS APPLICATION TOPICS](https://kafka.apache.org/38/documentation/streams/developer-guide/manage-topics.html)  



