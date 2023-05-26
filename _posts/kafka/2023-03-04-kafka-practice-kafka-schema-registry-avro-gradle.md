--- 
layout: single
classes: wide
title: "[Kafka] Kafka Schema Registry, Java Gradle with Avro Serialize/Deserialize"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Avro Gradle Plugin 을 사용해서 Avro 스키마 의 정의와 사용을 좀더 간편하게 해서 Kafka 와 Schema Registry 를 사용해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Schema Registry
    - Confluent Schema Registry
    - Avro
    - Gradle
    - Java
toc: true
use_math: true
---  


## Avro Serialize/Deserialize
[Kafka 와 Confluent Schema Registry]({{site.baseurl}}{% link _posts/kafka/2023-02-25-kafka-practice-kafka-schema-registry.md %})
에서는 `Confluent Schema Registry` 에 대한 개념과 기본적인 사용 방법에 대해서 알아보 았다.  

전 포스트에서 `Avro` 포맷으로 `Serialize/Deserialize` 하는 방법은 아래와 같은 `avro` 포맷 형식을 
코드내 문자열로 만들어 직접 스키마를 구성하는 방법이였다.  

```json
{
  "namespace":"com.windowforsun.test",
  "type":"record",
  "name":"my-test",
  "fields":[
    {"name":"name","type":"string"},
    {"name":"age","type":"int"}
  ]
}
```  

하지만 이러한 방법으로 실제 애플리케이션을 구현하기에는 위 스키마를 클래스 구현체와 매핑해야 한다는 점이다. 
직접 `SpecificRecordBase`, `SpecificRecord` 를 상속 받고 구현해서 매핑지을 수 있지만, 
매번 사용이 필요한 스키마에 대해서 해당 작업을 해주기는 번거러운 작업이다.  


그래서 이번 포스트에서는 `Gradle` 기반으로 `Avro` 스키마만 정의해 주면 `Schema Registry` 에 
등록도 해주고 매핑되는 클래스 또한 생성해주는 플러그인과 사용 방법에 대해 알아 본다.  


[specific.avro.reader](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-schema-registry/#producer-consumer-%EC%97%B0%EB%8F%99)
이전 포스팅에서 해당 옵션에 `true` 값을 주면 역직렬화 관련 에러가 발생했다. 
이번 포스팅에서 알아보는 `Gradle` 기반 방식을 사용하면 `Producer`, `Consumer` 에서 
특정 `Avro` 스키마 버전을 명시해서 보다 안정적으로 애플리케이션 구현이 가능하다.  

이전 포스팅에서는 `Producer` 가 생산한 스키마와 상관없이 `Consumer` 는 `GenericRecord` 를 사용해서 

### Gradle Avro Serialize/Deserialize Plugin
`Gradle` 기반으로 `Avro` 스키마에 해당하는 클래스를 자동 생성하기 위해서는 [davidmc24/gradle-avro-plugin](https://github.com/davidmc24/gradle-avro-plugin)
이라는 `Gradle Plugin` 을 사용한다. 

플러그인 사용을 위해서 `setting.gradle` 파일에 아래 저장소 설정을 추가해 준다.  

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```  

다음으로 `build.gradle` 내용은 아래와 같다.  

```groovy
plugins {
    id 'java'
    id "com.github.davidmc24.gradle.plugin.avro" version "1.3.0"
}

repositories {
    mavenCentral()
    maven {
        url "https://packages.confluent.io/maven/"
    }
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
    implementation 'org.apache.kafka:kafka-clients:2.6.3'
    implementation 'org.slf4j:slf4j-simple:1.7.30'
    implementation 'io.confluent:kafka-avro-serializer:6.0.0'
    implementation "org.apache.avro:avro:1.11.0"
    implementation "org.apache.avro:avro-tools:1.11.0"
}

test {
    useJUnitPlatform()
}
```  

`Procuder`, `Consumer` 애플리케이션의 `src/main/avro` 경로에 
`Avro` 스키마 정의 파일을 `value.avsc` 파일로 생성해 준다.  

```json
{
  "namespace": "com.windowforsun.avro.exam",
  "type": "record",
  "name": "Employee",
  "fields": [
      {"name": "firstName", "type": "string"},
      {"name": "lastName", "type": "string"},
      {"name": "age", "type": "int"}
  ]
}
```  

그리고 `./gradlew build` 명령으로 `Producer`, `Consumer` 애플리케이션을 빌드 해주면 
아래와 같이 `Employee` 라는 `Avro` 스키마와 매핑되는 클래스가 `build/generated-main-avro-java` 경로에 생성 된다.  

```
build
└── generated-main-avro-java
    └── com
        └── windowforsun
            └── avro
                └── exam
                    └── Employee.java
```  

`Producer` 애플리케이션의 구현코드는 아래와 같다. 
`GenericRecord` 객체를 사용해서 `key`, `value` 형태로 스키마 데이터를 생성하는 방식이 아니라, 
`Employee` 클래스를 바탕으로 `Kafka` 로 전송할 객체를 생성하는 것을 확인 할 수 있다.  

```java
public class AvroGradleProducer {
    private final static String TOPIC = "avro-gradle-employee";

    public static Producer<Long, Employee> createProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "AvroGradleProducer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, LongSerializer.class.getName());
        // config KafkaAvroSerializer
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());
        // schema registry location
        props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        return new KafkaProducer<>(props);
    }

    public static void main(String... args) {
        Producer<Long, Employee> producer = createProducer();

        try {
            for (int i = 0; i < 10; i++) {
                producer.send(getSchemaData());

                TimeUnit.SECONDS.sleep(3);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.flush();
            producer.close();
        }
    }

    public static ProducerRecord<Long, Employee> getSchemaData() {
        Random keyRandom = new Random();
        Employee employee = Employee.newBuilder()
                .setFirstName("Bob")
                .setLastName("Jones")
                .setAge(20)
                .build();

        return new ProducerRecord<>(TOPIC, keyRandom.nextLong(), employee);
    }
}
```  

`Consumer` 애플리케이션 구현 코드는 아래와 같다. 
`ConsumerRecord` 의 `Value` 객체가 `GenericRecord` 가 아닌 정의한 `Avro` 스키마와 매핑되는 `Employee` 객체이다. 
그리고 `KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG`('specific.avro.reader`) 
옵션 또한 `true` 로 설정해서 선언한 `Avro` 스키마를 사용해서 `Consumer` 가 `Deserialize` 를 수행하고 애플리케이션이 수행될 수 있도록 했다. 
테스트를 위해 `Consumer` 애플리케이션은 `GROUP_ID_CONFIG` 를 유니크 하게 생성하도록 해서 실행 때마다 토픽의 모든 데이터를 읽도록 했다.  

```java
public class AvroGradleConsumer {
    private final static String TOPIC = "avro-gradle-employee";

    public static Consumer<Long, Employee> createConsumer() {
        Properties prop = new Properties();
        prop.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        prop.put(ConsumerConfig.GROUP_ID_CONFIG, String.format("AvroConsumer-%s", UUID.randomUUID().toString()));
        prop.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, LongDeserializer.class.getName());
        prop.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        // kafka avro deserializer
        prop.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class.getName());
        // specific record
        prop.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, "true");
        prop.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        return new KafkaConsumer<>(prop);
    }

    public static void main(String... args) {
        Consumer<Long, Employee> consumer = createConsumer();
        consumer.subscribe(List.of(TOPIC));

        try {
            while(true) {
                ConsumerRecords<Long, Employee> records = consumer.poll(Duration.ofSeconds(10));

                if(records.isEmpty()) {
                    break;
                }

                for(ConsumerRecord<Long, Employee> record : records) {
                    System.out.printf("consume offset=%d, key=%d, value=%s\n", record.offset(), record.key(), record.value());
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```  

마지막으로 구동을 위해 `Docker` 와 `Docker Compose` 기반으로 환경을 구성한다. 
환경 구성에 사용되는 `docker-compose.yaml` 파일 내용은 아래와 같다. (이전 포스팅의 `docker-compose.yaml` 파일을 그대로 사용해도 동일하다.)

```yaml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:6.2.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:6.2.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost

  schema-registry:
    image: confluentinc/cp-schema-registry:6.2.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - broker
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker:29092'
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
```  

`docker-compose up --build` 명령으로 위 구성을 실행하고 `Producer` 와 `Consumer` 순으로 
실행한다.  

```bash
$ docker-compose up --build
[+] Running 4/0
 ⠿ Network exam2_default      Created                                                                                                                                                                                                              0.0s
 ⠿ Container zookeeper        Created                                                                                                                                                                                                              0.0s
 ⠿ Container broker           Created                                                                                                                                                                                                              0.0s
 ⠿ Container schema-registry  Created                                                                                                                                                                                                              0.0s
Attaching to broker, schema-registry, zookeeper

```  

`Consumer` 애플리케이션 콘솔 출력을 통해 애플리케이션에 정의된 `value.avsc` 라는 `Avro` 스키마 정의파일이 
`gradle plugin` 을 통해 클래스로 만들어지고 해당 클래스를 바탕으로 메시지를 생산하고 소비한 결과를 확인 할 수 있다.  

```
consume offset=0, key=7711826870628396238, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=1, key=421165421102176529, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=2, key=-6966530516040386184, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=3, key=7476769790129067786, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=4, key=1580369344782630981, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=5, key=1671071163925824873, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=6, key=8908208290679904106, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=7, key=3681635501498229080, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=8, key=-1818972359259312973, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=9, key=6594268417272188164, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
```  

#### 스키마 업데이트
[스키마 진화 전략](https://windowforsun.github.io/blog/kafka/kafka-practice-kafka-schema-registry/#%EC%8A%A4%ED%82%A4%EB%A7%88-%EC%A7%84%ED%99%94-%EC%A0%84%EB%9E%B5)
에서 `Avro` 스키마를 업데이트하고 변경하는 조건과 전략에 대해서 알아 보았다. 
만약 해당 전략에 부합하지 않는 방식으로 변경하면 아래와 같은 메시지를 확인 할 수 있다. 
아래는 기존 `Employee` 스키마의 호환 타입인 `BACKWARD` 가 부합하지 않는 기본 값이 설정되지 않은 `age2` 라는 필드를 추가한 결과이다.  

```
.. Consumer ..
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing Avro message for id 1
Caused by: org.apache.avro.AvroTypeException: Found com.windowforsun.avro.exam.Employee, expecting com.windowforsun.avro.exam.Employee, missing required field age2
	at org.apache.avro.io.ResolvingDecoder.doAction(ResolvingDecoder.java:308)
	at org.apache.avro.io.parsing.Parser.advance(Parser.java:86)
	at org.apache.avro.io.ResolvingDecoder.readFieldOrder(ResolvingDecoder.java:127)
	at org.apache.avro.generic.GenericDatumReader.readRecord(GenericDatumReader.java:240)
	at org.apache.avro.specific.SpecificDatumReader.readRecord(SpecificDatumReader.java:123)
	at org.apache.avro.generic.GenericDatumReader.readWithoutConversion(GenericDatumReader.java:180)
	at org.apache.avro.generic.GenericDatumReader.read(GenericDatumReader.java:161)
	at org.apache.avro.generic.GenericDatumReader.read(GenericDatumReader.java:154)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer$DeserializationContext.read(AbstractKafkaAvroDeserializer.java:327)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.deserialize(AbstractKafkaAvroDeserializer.java:97)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.deserialize(AbstractKafkaAvroDeserializer.java:76)
	at io.confluent.kafka.serializers.KafkaAvroDeserializer.deserialize(KafkaAvroDeserializer.java:55)
	at org.apache.kafka.common.serialization.Deserializer.deserialize(Deserializer.java:60)
	at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1365)
	at org.apache.kafka.clients.consumer.internals.Fetcher.access$3400(Fetcher.java:130)
	at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.fetchRecords(Fetcher.java:1596)
	at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.access$1700(Fetcher.java:1432)
	at org.apache.kafka.clients.consumer.internals.Fetcher.fetchRecords(Fetcher.java:684)
	at org.apache.kafka.clients.consumer.internals.Fetcher.fetchedRecords(Fetcher.java:635)
	at org.apache.kafka.clients.consumer.KafkaConsumer.pollForFetches(KafkaConsumer.java:1308)
	at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1237)
	at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1210)
	at com.windowforsun.schemaregistry.AvroGradleConsumer.main(AvroGradleConsumer.java:39)
	
	
	
.. Producer ..
Path in schema: --> age2
	at org.apache.avro.generic.GenericData.getDefaultValue(GenericData.java:1205)
	at org.apache.avro.data.RecordBuilderBase.defaultValue(RecordBuilderBase.java:138)
	at com.windowforsun.avro.exam.Employee$Builder.build(Employee.java:460)
	at com.windowforsun.schemaregistry.AvroGradleProducer.getSchemaData(AvroGradleProducer.java:56)
	at com.windowforsun.schemaregistry.AvroGradleProducer.main(AvroGradleProducer.java:38)
```  

`Employee` 스키마의 호환 타입인 `BACKWARD` 에 부합할 수 있는 조건인 기존 필드인 `age` 필드 삭제와 
기본 값이 설정된 `phoneNumber` 필드 추가를 위해 `value.avsc` 파일을 아래와 같이 수정하고, 
`Producer`, `Consumer` 모두 다시 `./gradlew build` 명령으로 빌드 한다.  

```json
{
  "namespace": "com.windowforsun.avro.exam",
  "type": "record",
  "name": "Employee",
  "fields": [
      {"name": "firstName", "type": "string"},
      {"name": "lastName", "type": "string"},
      {"name": "phoneNumber", "type" : "string", "default": ""}
  ]
}
```  

`Producer` 의 스키마 데이터 생성 메소드는 아래와 같이 수정이 필요하다.  

```java
public static ProducerRecord<Long, Employee> getSchemaData() {
    Random keyRandom = new Random();
    Employee employee = Employee.newBuilder()
            .setFirstName("Bob")
            .setLastName("Jones")
            // 삭제
//            .setAge(30)
            // 추가
            .setPhoneNumber("01012345678")
            .build();

    return new ProducerRecord<>(TOPIC, keyRandom.nextLong(), employee);
}
```  

그리고 `Consumer`, `Producer` 순으로 다시 애플리케이션을 실행하면 새로운 스키마 데이터가 `Producer` 를 통해 생산되고, 
`Consumer` 는 이전 버전과 현재 버전 파싱이 모두 가능한 것을 콘솔 출력을 통해 확인 할 수 있다.  

```
consume offset=0, key=7711826870628396238, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=1, key=421165421102176529, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=2, key=-6966530516040386184, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=3, key=7476769790129067786, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=4, key=1580369344782630981, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=5, key=1671071163925824873, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=6, key=8908208290679904106, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=7, key=3681635501498229080, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=8, key=-1818972359259312973, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=9, key=6594268417272188164, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": ""}
consume offset=10, key=5469801176880890252, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=11, key=-2801838287309200461, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=12, key=3322856687502275735, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=13, key=9123566577538030979, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=14, key=-6828971205224721375, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=15, key=1722047268368527770, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=16, key=-4458625612544035587, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=17, key=-5259612762035821675, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=18, key=4514876730759371651, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
consume offset=19, key=-984294560683093556, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012345678"}
```  

#### 여러 스키마 정의
한 `Producer`, `Consumer` 애플리케이션에서 여러 스키마(토픽)의 데이터 처리를 수행하는 경우 
아래와 같이 여러 스키마를 정의해 돌 수 있다.  

```
src
└── main
    └── avro
        ├── value.avsc
        └── value2.avsc
```  

`value.avsc` 는 기존에 사용한 `Employee` 스키마 정의이다.  

```
{
  "namespace": "com.windowforsun.avro.exam",
  "type": "record",
  "name": "Employee",
  "fields": [
      {"name": "firstName", "type": "string"},
      {"name": "lastName", "type": "string"},
      {"name": "phoneNumber", "type" : "string", "default": ""}
  ]
}
```  

`value2.avsc` 는 새로 추가된 배열 형태로 `Test1`, `Test2` 라는 스키마가 정의된 파일이다.  

```
[
    {
      "namespace": "com.windowforsun.avro.exam",
      "type": "record",
      "name": "Test2",
      "fields": [
          {"name": "strTest2", "type": "string"},
          {"name": "intTest2", "type": "int"}
      ]
    },
    {
      "namespace": "com.windowforsun.avro.exam",
      "type": "record",
      "name": "Test1",
      "fields": [
          {"name": "strTest1", "type": "string"},
          {"name": "intTest1", "type": "int"}
      ]
    }
]
```  

`./gradlew build` 명령으로 빌드하면 아래와 같이 `build/generated-main-avro-java` 경로에 `Employee` 클래스외 
`Test1`, `Test2` 클래스가 추가된 것을 확인 할 수 있다.  


```
build
└── generated-main-avro-java
    └── com
        └── windowforsun
            └── avro
                └── exam
                    ├── Employee.java
                    ├── Test1.java
                    └── Test2.java
```  




---  
## Reference
[Kafka Clients in Java with Avro Serialization and Confluent Schema Registry](https://thecodinginterface.com/blog/gradle-java-avro-kafka-clients/)  
[Kafka, Avro Serialization, and the Schema Registry](https://dzone.com/articles/kafka-avro-serialization-and-the-schema-registry)  
[davidmc24/gradle-avro-plugin](https://github.com/davidmc24/gradle-avro-plugin)  
