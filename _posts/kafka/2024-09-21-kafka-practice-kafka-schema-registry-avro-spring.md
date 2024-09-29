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
toc: true
use_math: true
---  

## Avro & Schema Registry
[Schema Registry]({{site.baseurl}}{% link _posts/kafka/2023-02-25-kafka-practice-kafka-schema-registry.md %})
와
[Schema Registry Avro Gradle]({{site.baseurl}}{% link _posts/kafka/2023-03-04-kafka-practice-kafka-schema-registry-avro-gradle.md %})
에서 `Schema Registry` 와 `Avro` 를 `Gradle` 환경에서 사용할 수 있는 방법에 대해 알아보았다. 
이번 포스팅에서는 이를 종합해 `Spring Boot` 에서 활용하는 예시를 알아보고자 하는데, 
그에 앞서 먼저 `Schema Registry` 와 `Avro` 에 대해 다시 한번 정리해보려 한다.  

`Kafka Broker` 를 기반으로 `Producer` 와 `Consumer` 가 스트리밍을 처리할 때, 
메시지의 직렬화/역직렬화는 필수적인 요소이고, 애플리케이션은 중단 없이 계속해서 메시지 처리가 가능해야 한다. 
그리고 그 메시지에는 다양한 데이터가 포함 될 수 있고, 필요에 따라 추가/삭제 또한 가능할 것이다. 
이러한 요구사항을 만족 시킬 수 있는것이 바로 스키마이다. 
스키마를 적용하면 `Producer` 가 직렬화를 통해 토픽에 쓴 메시지가 토픽의 `Consumer` 가 소비하고 역직렬화를 해서 처리 할 수 있도록 보장한다. 
이렇게 `Producer` 와 `Consumer` 는 스키마를 알아야 하고 스키마 변경 관리를 위해서는 `Schema Registry` 가 필요하다. 
여기에 `Apache Avro` 를 사용하면 버전 간 호환성, 타입 처리를 제공하는 직렬화 프레임워크로 끊김 없는 스트림 처리에서 
여러 버전 메시지 스키마르 호환성 높게 사용 할 수 있도록 한다.  


### Avro
`Apache Avro` 는 데이터 직렬화 시스템으로 분산 컴퓨팅 환경에서 효과적이다. 
`Avro` 는 `JSON` 형식의 스키마를 사용하여 스키마를 정의하고, 
스키마를 바탕으로 데이터를 직렬화/역직렬화한다. 
먼저 정의된 스키마 데이터는 이를 저장하고 관리하는 저장소에 보관된다. 
즉 실제 데이터와 스키마가 분리되어 있어 다른 시스템들과 비교했을 때 좀 더 신뢰성과 효율성에 이점이 있다.  

그리고 `Avro` 는 데이터를 바이너르 형태로 직렬화해 저장하기 때문에 
`JSON`, `XML` 과 같은 텍스트 기반 데이터 포멧과 비교해서 크기가 작다. 
이를 토해 네트워크 전송 시간 및 저장공간을 절약할 수 있다.  

다른 특징으로는 스키마의 진화를 지원한다. 
여기서 스키마 진화는 시간이 지남에 따라 스키마에 변경이 발생하는 것을 의미한다. 
구버전 데이터와 신버전 데이터가 호환될 수 있도록 하여 스트리밍 처리환경에서 활용도가 높다.   


### Schema Registry
앞서 메시지를 정의하는 스키마를 사용하면 메시지 변경에 따른 호환성을 높일 수 있다는 것에 대해 알아보았다. 
스키마를 사용하는 방법 중 가장 간단한 방법은 메시지에 해당 메시지와 대응되는 스키마 내용을 포함시키는 것이다. 
하지만 이는 모든 메시지의 크기를 증가시켜, `Producer`, `Consumer`, `Broker` 가 필요로하는 `CPU` 와 메모리를 증가시켜 처리량에 직접적인 영향을 줄 수 있다.  

이러한 문제점을 해결하면서 메시지 스키마를 사용할 수 있는 방법이 바로 `Schema Registry` 이다. 
`Schema Registry` 는 앞서 나열한 구성들과 독립적으로 구성되어 네트워크 통신을 통해 메시지의 스키마를 관리하고 정보를 조회할 수 있다.  


.. 그림 ..

`Kafka` 진영에서 가장 보편적인 `Schema Registry` 는 `Confluent Schema Registry` 이다. 
스키마를 등록하고 조회하기 위한 `REST API` 를 제공한다. 
그러므로 `Producer` 실행 전 명시적으로 사용하는 스키마를 `REST API` 를 사용해서 등록해주거나, 
`CI/CD` 파이프라인의 일부에서 스키마를 등록하는 시나리오를 바탕으로 데이터 스트리밍 환경에서 데이터 일관성과 통합성을 보장 할 수 있다.   

`Schema Registry` 는 필수적인 구성요소는 아니고 구성하지 않더라도 메시지 스트리밍 처리는 구현할 수 있다. 
그렇다면 있을 경우와 그렇지 않은 경우에 어떠한 차이를 보이는지에 대해 좀 더 자세히 알아보자. 

- `Schema Registry` 가 있는 경우
  - 스키마 진화 관리 : `Schema Registry` 는 스키마의 버전 관리를 지원한다. 스키마가 변할 경우 이전 버전과의 호환성을 유지할 수 있다. 이는 스키마가 변경 됐을 때 기존 스키마와 어떻게 호환될지를 정의하고 관리할 수 있다. 
  - 중앙 집중식 스키마 관리 : 모든 스키마 정보를 한 곳에 저장함으로써 모든 애플리케이션에서 동일한 스키마를 사용하고 있음을 보장할 수 있다. 여러 서비스 및 여러 팀 간의 일관성 유지에 큰 도움이 된다. 
  - 자동 스키마 검증 : `Producer` 가 메시지를 전송할 때, `Schema Registry` 는 전송된 메시지가 등록된 스키마와 일치하는지 검증한다. 
  - 효율적인 데이터 역직렬화 : `Consumer` 는 `Schema Registry` 에서 검색하여 받은 스키마로 메시지를 역직렬화 할 수 있다. 이는 스키마가 매번 메시지에 포함되지 않기 때문에 메시지 크기가 작아지고 네트워크 효율성이 높아진다. 
- `Schema Registry` 가 없는 경우
  - 스키마 포함 메시지 : 스키마가 메시지에 포함되어 메시지 크기가 커져 네트워크와 저장공간을 더 많이 사용한다. 
  - 스키마 관리의 복잡성 증가 : 각 애플리케이션 또는 팀에서 독립적으로 스키마를 관리하게 되면, 시스템 전체에서 스키마의 일관성을 유지하기 어렵다. 
  - 스키마 진화의 어려움 : 스키마 변경 시 모든 관련 애플리케이션을 업데이트하고, 변경된 스키마에 대한 호환성을 각각 검증해야 한다. 

### Registering Schemas
`Confluent Schema Registry` 는 내부적으로 `Kafka` 를 저장소로 사용한다. 
이는 `Schema Registry` 에 등록되는 스키마를 `_schemas` 라는 압축된 토픽에 저장하는 것을 의미한다. 
만약 토픽이름을 변경하고 싶다면 `kafkastore.topic` 옵션 수정을 통해 가능하다.  

`Confluent Schema Registry` 는 앞서 언급한 것과 같이 `REST API` 를 사용해서 스키마를 등록한다. 
이는 수동으로 `REST API` 를 호출해서 등록하는 것도 가능하고, `Schema Registry Client` 를 사용하거나, 
`Maven`, `Gradle` 플러그인을 통해 수행하도록 구현할 수도 있다. 
또한 `Kafka Avro` 라이브러리를 사용한다면 `Producer` 가 메시지를 보내기 전에 `Avro Serilizer` 가 메시지의 스키마를 `Schema Registry` 에 확인한다. 
스키마가 존재하지 않을 경우 스키마를 등록하고, 존재한다면 스키마 아이디를 사용해서 메시지를 직렬화한다.  

최종적으로 `Schema Registry` 에 등록된 스키마는 `_schemas` 토픽에 쓰여지고, 
`Shcema Registry` 는 내부 `Consumer` 를 통해 이를 소비해서 로컬 캐시에 저장한다.  


.. 그림 ..


1. `Avro Schema` 는 `REST POST` 를 통해 `Schema Registry` 에 등록된다. 
2. `Schema Registry` 는 새로운 스키마를 내부 `Producer` 를 통해 `_schemas` 토픽에 저장한다. 
3. `Schema Registry` 는 내부 `Consumer` 로 `_schemas` 에 등록된 새로운 스키마를 소비한다. 
4. 소비된 새로운 스키마는 로컬 캐시에 저장된다. 

이후 스키마를 필요로하는 `Producer`, `Consumer` 가 스키마 아이디 혹은 스키마를 조회할 때 로컬 캐시를 통해 응답하게 된다. 
`Confluent Schema Registry` 는 데이터 저장소로 `Kafka` 를 사용하는 만큼 `Kafka` 에서 제공하는 모든 장점을 얻을 수 있다. 
단편적인 예시로 특정 `Broker` 가 실패하더라도, 복제 파티션을 통해 `Scahme Registry` 의 사용은 지속적으로 가능하다. 
또한 `Schema Registry` 가 실패해 재시작 되는 경우에는 `_schemas` 토픽 구독을 통해 등록된 기존 스키마를 로컬 캐시에 재구축하게 된다.  

### Serialization/Deserialization
`Kafka Avro` 라이브러리는 메시지 키, 메시지 값을 직렬화/역직렬화하는 
`KafkaAvroSerilizer` 과 `KafkaAvroDeseializer` 라는 클래스를 제공한다. 
직렬화가 필요한 `Producer` 는 `KafkaAvroSerilizer` 를 역직렬화가 필요한 `Consumer` 는 `KafkaAvroDeserializer` 를 등록하기만 하면 된다. 
이후 실제 직렬화/역직렬화 처리와 `Schema Registry` 와의 상호작용은 `Kafka Avro` 에서 수행해주기 때문에 추가적인 구현은 필요하지 않다. 
아래 그림은 `Kafka Avro` 를 사용하는 `Producer` 와 `Consumer` 그리고 `Schema Registry` 가 가지는 일반적인 흐름을 도식화 한 것이다.  


.. 그림 ..

1. 메시지가 만들어지면 `Producer` 의 `Avro Serializer` 는 메시지의 스키마가 가지는 스키마 `ID` 를 `Schema Registry` 로 부터 조회한다. 
2. 메시지는 `Avro` 로 직렬화 되고, 검증된다. 
3. 메시지는 `Kafka Topic` 으로 전송된다. 
4. 메시지는 `Kafka Topic` 을 구독하는 `Consumer` 에게 소비된다. 
5. `Consumer` 의 `Avro Deserializer` 는 메시지에 있는 스키마 `ID` 를 통해 `Schema Registry` 에서 스키마를 조회한다. 
6. 메시지는 조회된 스키마를 통해 역직렬화되고 검증된다.  

아래 다이어그램은 `Producer` 측면의 상세한 흐름을 표현한 것이다. 
`KafkaAvroSerilizer` 는 먼저 `Reflection` 을 통해 메시지 클래스에서 스키마를 획득한다. 
그리고 `kafka-schema-registry-client` 라이브러리에서 제공되는 
`CachedSchemaRegistryClient` 를 사용해 스키마 `ID` 를 조회한다. 
스키마 `ID` 가 캐싱돼있지 않았다면, 클라이언트는 `REST API` 를 호출해 스키마 `ID` 를 얻고 캐싱해둔다. 
`REST API` 가 필요한 시점은 스키마 `ID` 를 얻어오는 과정과 같은 초기 작업뿐이다. 
이제 `KafkaAvroSerializer` 는 메시지를 바이트 배열로 직렬화하는데, 
`magic byte` 라 불리는 직렬화 포맷 버전 번호로 시작한다. 
그리고 이어 스키마 ID가 위치하고 그 다음으로 `Avro` 의 이진 인코딩으로 처리된 데이터가 위치한다. 
최종적으로 해당 데이터가 `Kafka Topic` 으로 전송된다.  


.. 그림 ..

아래는 `Consumer` 측면의 상세한 흐름을 표현한 것이다. 
`Kafka Topic` 의 메시지가 `Consumer` 에게 전달되면 
`Consumer` 는 역직렬화 처리를 `KafkaAvroDeserilizer` 에게 위임한다. 
먼저 메시지의 1~4번 바이트에 위치하는 스키마 `ID` 를 추출한다. 
그리고 `CachedSchemaRegistryClient` 를 사용해서 로컬 캐시에 해당하는 스키마가 있는지 확인한다. 
만약 존재하지 않다면 클라이언트는 `REST API` 를 사용해 스키마를 조회한다. 
조회된 스키마는 로컬 캐시에 저장되어 이후 역직렬화에서도 사용된다. 
이제 `KafkaAvroDeserializer` 는 바이트 배열의 메시지를 `Avro` 클래스로 역직렬화 가능하다. 
그리고 역직렬화된 메시지는 `Consumer` 가 처리할 수 있도록 전달된다.   


## Spring Boot Demo
구현하는 `Spring Boot Demo` 는 간단한 이벤트 처리를 구현한 내용이다. 
자세한 코드 내용은 [여기]()
에서 확인 할 수 있다. 
`send-event` 라는 `Inbound Topic` 에서 메시지를 받아 메시지 관련 처리를 수행하도록 트리거한다. 
이러한 트리거 동작은 `REST API` 를 통해 가능하도록 구현돼 있다. 
그리고 처리된 메시지는 `my-event` 라는 `Outbound Topic` 에 이벤트를 생성하고, 
해당 이벤트 발생이 성공하면 메시지 처리의 전과정이 모두 성공적으로 처리되었다고 판단한다.  

`Avro Schem` 로 정의된 이벤트는 `MyEvent` 와 `SendEvent` 두 가지가 있다. 
데모 애플리케이션은 `Schema Registry` 를 사용해서 위 2개 `Avro` 메시지를 직렬화/역직렬화하기 위해 스키마를 등록하고 조회하게 된다.  

아래 그림은 `SendEvent` 메시지가 소비되는 시점부터 시작해서 
`MyEvent` 가 최종적으로 발송되는 전체 처리 흐름을 보여주고 있다.  


.. 그림 .. 

1. `Inbound Topic` 에서 `SendEvent` 가 소비된다. 
2. 소비된 메시지 바이트 배열 시작 부분에 위치하는 `Schema Id` 를 추출한 후, `Schema Registry` 에서 스키마를 조회하고, 결과 스키마를 캐시한다. 
3. 조회한 스키마를 바탕으로 메시지를 역직렬화한다. 
4. 이후 메시지 처리에 필요한 동작을 수행해준다. (`Third party call`, ..)
5. `Outbound Topic` 에 결과 메시지인 `MyEvent` 를 발송하기 전, `Schema Registry` 에 스키마 등록/ID조회를 한다. 그리고 `Schema ID` 는 캐시 후, 메시지 클래스를 통해 스키마를 획득한다. 
6. 메시지가 직렬화되고, `Schema Id` 는 메시지 시작 부분에 포함된다. 
7. 최종적으로 `Outbound Topic` 에 `MyEvent` 의 직렬화 데이터가 발행된다. 


### Avro Schema & Generating
데모 프로젝트는 `Gradle` 빌드 툴을 사용한다. 
그러므로 공식적인 `Avro` 관련 플러그인은 지원되지 않아 
[Gradle Avro Plugin](https://github.com/davidmc24/gradle-avro-plugin)
을 사용해서 빌드 과정에 `Avro Schema` 를 통해 `Java Class` 파일이 생성될 수 있도록 구성한다. 
관련해서 보다 상세한 내용은 [Kafka Avro Gradle]({{site.baseurl}}{% link _posts/kafka/2023-03-04-kafka-practice-kafka-schema-registry-avro-gradle.md %})
에서 확인 할 수 있다.  

`Avro Schema` 파일은 `scr/main/avro` 디렉토리 하위에 `send-event.avsc`, `my-event.avsc` 와 같이 2개가 존재한다. 

```json
{
     "type": "record",
     "namespace": "com.windowforsun.avro",
     "name": "MyEvent",
     "version": "1",
     "fields": [
        { "name": "eventId", "type": "string" },
        { "name": "number", "type": "long" }
     ]
}
```  

```json
{
     "type": "record",
     "namespace": "com.windowforsun.avro",
     "name": "SendEvent",
     "version": "1",
     "fields": [
        { "name": "eventId", "type": "string" },
        { "name": "number", "type": "long" }
     ]
}
```  

스키마 정의 후 `Gradle` 의 `build` 명령을 수행해주면 `Java Class` 가 생성된다.  


### Registering Avro Schema
`Avro Schema` 와 `Confluent Schema Registry` 를 사용 할 때 스키마를 등록하는 것은 여러 방법이 있고, 
사용하는 빌드 툴 및 라이브러리에 따라 달라질 수 있다. 
먼저 `Maven` 을 사용하는 경우는 `kafka-schema-registry-maven-plugin` 을 사용하면 빌드 프로세스 중 `Avro Schema` 를 
`Schema Registry` 에 자동으로 등록할 수 있다.  

```xml
<plugin>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-schema-registry-maven-plugin</artifactId>
  <version>${confluent.version}</version>
  <configuration>
     <schemaRegistryUrls>
        <param>http://localhost:8081</param>
     </schemaRegistryUrls>
     <subjects>
        <send-event-value>${project.basedir}/src/main/resources/avro/send-event.avsc</send-event-value>
        <my-event-value>${project.basedir}/src/main/resources/avro/myeent.avsc</my-event-value>
     </subjects>
  </configuration>
  <goals>
     <goal>register</goal>
  </goals>
</plugin>
```  

`schemaRegistryUrls` 에 사용 중인 `Schema Registry` 의 `URL` 을 지정한다. 
그리고 `subjects` 에 등록할 스키마 파일과 해당 스키마가 등록될 `subject` 를 지정한다. 
여기서 `subject` 는 `Schema Registry` 에서 스키마를 식별하는데 사용되며, 여러 버전의 스키마를 관리할 수 있다. 
`Maven` 의 `install` 혹은 `deploy` 명령과 함께 실행 할 수 있고, `maven schema-registry:register` 과 같이 등록 동작만 별도로 수행할 수 있다.  

다음은 `Gradle` 인 경우를 알아본다. 
`Gradle` 의 경우 `Maven` 과 같이 `Schema Registry` 등록을 지원해주는 플러그인이 존재하지 않는다. 
그러므로 필요하다면 직접 커스텀 태스크를 작성해서 빌드과정에 수행될 수 있도록 구성해줘야 한다. 

```groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

task registerAllAvroSchemas {
    doLast {
        file('src/main/avro').eachFileMatch(~/.+\.avsc$/) { File schemaFile ->
            def schemaText = schemaFile.text
            def subject = schemaFile.name - '.avsc'  // 파일 이름에서 .avsc 제거
            def url = "http://localhost:8081/subjects/${subject}/versions"
            def json = JsonOutput.toJson(["schema": JsonOutput.toJson(schemaText)])

            def conn = new URL(url).openConnection() as HttpURLConnection
            conn.requestMethod = 'POST'
            conn.doOutput = true
            conn.setRequestProperty('Content-Type', 'application/vnd.schemaregistry.v1+json')
            conn.outputStream.withWriter('UTF-8') { it.write(json) }
            conn.connect()

            if (conn.responseCode == HttpURLConnection.HTTP_OK || conn.responseCode == HttpURLConnection.HTTP_CREATED) {
                println "Schema ${subject} successfully registered."
                def responseParsed = new JsonSlurper().parseText(conn.inputStream.text)
                println "ID of registered schema for ${subject}: ${responseParsed.id}"
            } else {
                println "Failed to register schema ${subject}. Server responded with status code: ${conn.responseCode}"
                println conn.errorStream.text
            }
            conn.disconnect()
        }
    }
}

// Optionally, ensure this task runs after Avro classes are generated
registerAllAvroSchemas.dependsOn 'generateAvroJava'
```  

그리고 빌드 프로세스의 일부로 자동적으로 시행되도록 설정하고 싶다면 아래 설정을 추가해주면 된다.  

```groovy
build.dependsOn registerAllAvroSchemas
```  

마지막으로 프로젝트에서 `kafka-avro-serializer` 의존성을 사용한다면 아래와 같이 `Producer` 및 `Consumer` 설정에 `Schema Registry` 
설정값을 추가해주면 스키마 등록 과 같은 `Schema Registry` 와의 상호동작을 자동으로 수행해준다.  

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());
props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");
props.put(KafkaAvroSerializerConfig.AUTO_REGISTER_SCHEMAS, "true");

KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props);
```  

### Schema Registry REST API
`Schema Registry` 에서 관리되는 모든 스키마는 `REST API` 로 조회하거나 등록/삭제를 수행 할 수 있다. 
관련 자세한 내용은
[Confluent Schema Registry]({{site.baseurl}}{% link _posts/kafka/2023-02-25-kafka-practice-kafka-schema-registry.md %})
에서 확인 가능하다.  


### Consumer Config
데모 애플리케이션에서 구현된 `Consumer` 는 아래와 같이 일반적인 `Kafka Consumer` 로 `Avro` 및 `Schema Registry` 를 추가로 사용한다고 해서, 
달라지는 부분은 없다.  

```java
@KafkaListener(topics = "send-event", 
        groupId = "demo-consumer-group", 
        containerFactory = "kafkaListenerContainerFactory")
public void listen(@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String key, @Payload final SendEvent sendEvent) {
    this.counter.getAndIncrement();
    log.debug("Received message [{}] - key : {}", this.counter.get(), key);

    try {
        this.myEventService.process(key, sendEvent);
    } catch (Exception e) {
        log.error("Error processing message : {}", e.getMessage());
    }

}
```  

이는 `Avro` 및 `Schema Registry` 가 `Consumer` 의 구성과는 `Decoupling` 관계로 직접적인 영향은 주지 않는다. 
`Consumer` 설정에서 `containerFactory` 를 명시적으로 사용하고 있는데, 
일반적인 `Consumer` 에서 추가로 필요하는 `Avro` 와 `Schema Registry` 의 설정은 `Kafka Consumer Bean` 설정 내용에 포함된다. 

```java
@Bean
public ConcurrentKafkaListenerContainerFactory kafkaListenerContainerFactory(final ConsumerFactory consumerFactory) {
    final ConcurrentKafkaListenerContainerFactory factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);

    return factory;
}

@Bean
public ConsumerFactory consumerFactory() {
    final Map<String, Object> config = new HashMap<>();
    config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    config.put(ConsumerConfig.GROUP_ID_CONFIG, "demo-consumer-group");
    config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    config.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
    config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, KafkaAvroDeserializer.class);
    config.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, this.schemaRegistryUrl);
    config.put(KafkaAvroDeserializerConfig.AUTO_REGISTER_SCHEMAS, false);
    config.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);

    return new DefaultKafkaConsumerFactory<>(config);
}
```  

`Kafka Consumer` 의 메시지 역직렬화와 관련된 설정과 설명을 정리하면 아래와 같다.  

properties|value|desc
---|--|---
VALUE_DESERIALIZER_CLASS_CONFIG|ErrorHandlingDeserializer.class|역직렬화 과정에서 발생할 수 있는 예외를 잡아내 처리하는 `Deserializer Wrapper`
KEY_DESERIALIZER_CLASS_CONFIG|ErrorHandlingDeserializer.class|역직렬화 과정에서 발생할 수 있는 예외를 잡아내 처리하는 `Deserializer Wrapper`
KEY_DESERIALIZER_CLASS|StringDeserializer.class|메시지의 키는 `String` 타입으로 역직렬화 한다. 
VALUE_DESERIALIZER_CLASS|KafkaAvroDeserializer.class|메시지의 값은 `Avro` 타입으로 역직렬화 한다. 
SCHEMA_REGISTRY_URL_CONFIG|e.g. http://localhost:8081|`Confluentd Schema Registry` 의 URL
AUTO_REGISTER_SCHEMAS|false|`Schema Registry` 에 존지하지 않는 스키마를 자동으로 등록할지 여부 설정
SPECIFIC_AVRO_READER_CONFIG|true|`GenericRcord` 타입을 사용하지 않고 `Avro` 스키마로 생성된 클래스로 역직렬화 할지에 대한 여부 설정

데모 애플리케이션의 설정에서는 메시지의 값만 `Avro` 스키마를 사용한다. 
하지만 메시지의 키 또한 필요하다면 `Avro` 를 사용해서 직렬화/역직렬화를 해서 사용할 수 있다.  

그리고 `AUTO_REGISTER_SCHEMAS` 는 실제 리얼 환경보다는 개발/테스트 환경에서 편의성을 위해 사용하는 것이 좋다. 
우선 스키마의 변경 관리는 철저한 프로세르르 바탕으로 통제된 과정으로 수행되는 것이 좋다. 
이는  스키마의 변경 관리가 엄격하지 않을 경우 자동으로 등록된 스키마가 기존 데이터와 호환되지 않는 상태를 포함할 수 있기 때문이다.  

