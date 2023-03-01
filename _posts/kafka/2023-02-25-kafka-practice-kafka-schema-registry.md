--- 
layout: single
classes: wide
title: "[Kafka] Kafka 와 Confluent Schema Registry"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Confluent Schema Registry 에 대한 설명과 이를 활용해 안정적인 Kafka 메시지 관리에 대해 알아보자'
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
toc: true
use_math: true
---  

## Confluent Schema Registry
`Schema Registry` 는 `Kafka` 흐름에서 생산되고 소비되는 메시지의 형태를 안정적으로 변경하고 관리 할 수 있도록 도와준다. 
`Kafka` 를 활용하는 애플리케이션의 기본적인 구조는 아래와 같이, 
`Producer`, `Kafka`, `Consumer` 로 구성되고 이어지는 흐름일 것이다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-1.drawio.png)  


사용자는 위 흐름에 데이터를 실어 원하는 비지니스를 구현한다.
위 상황에서 계속해서 변하는 비지니스의 요구사항에 맞춰 메시지의 구조 또한 함꼐 변화할 것이다. 
필드를 추가하거나, 삭제, 타입 변경들 그 변경사항은 특정 지을 수 없다. 
이러한 변경사항이 적용되다 보면 아래와 같이 `Producer` 와 `Consumer` 간의 데이터 포멧의 불일치가 발생 할 수 있다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-2.drawio.png)

기존 `name, age` 로 구성된 데이터에서 `age` 가 삭제되고 `address` 가 추가 되었다.
이런 상태라면 `Consumer` 는 정상동작 할 수 없거나 오류가 발생할 수 있다.  

이렇게 메시지의 형태를 지속적으로 안전한 방향으로 변경하며 호환성을 유지할 수 있도록 하는 것이 바로 `Schema Registry` 이다.  

참고로 `Schema Registry` 는 `Confluent` 에서 제공하는 것만 있는 것이 아니다. 
`Hortonworks Schema Registry`, `Spring Cloud Stream Schema Registry` 등 
다양한 구현체들이 존재하지만 본 포스팅에서 알아볼 것은 `Kafka` 와 함께 많이 사용하는 `Confluent Schema Registry` 에 대한 내용이다.  


### 구조와 배경

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-3.drawio.png)

`Schema Registry` 는 `Kafka` 를 통해 주고 받는 메시지 스키마를 중앙에서 관리하기 위한 애플리케이션이다. 
이런 관리 애플리케이션까지 나온 배경은 아래와 같다.  

- `Kafka Topic` 에 저장되는 데이터는 저장후 수정하는 것은 불가능하다. 
- `Producer-Consumer` 는 `메시지 포맷` 에 강한 결합도를 가지고 있다. 
- 강한 결합도로 인해 메시지 포맷이 맞지 않는다면 정상동작을 보장 할 수 없다. 
- `Producer` 에서 생산하는 메시지에 스키마 정보를 넣는 방법으로 호환성을 해결 할 수 있지만, 데이터의 크기가 방대해 진다. 

### 스키마 진화 전략
[Schema Evolution and Compatibility](https://docs.confluent.io/platform/current/schema-registry/avro.html#compatibility-types)
에 나와 있는 내용으로 `Schema Registry` 에서 변화하는 스키마 정보를 관리하고 호환성을 유지할 수 있다.  

여러 호환 타입이 있는데 그 중 대표적인 몇가지만 정리하면 아래와 같다.  

호환 타입|설명|가능한 변경|비교 버전|선 변경
---|---|---|---|---
BACKWAWRD|(기본값)새로운 스키마로 이전 데이터를 읽는 것이 가능하다.|- 필드 삭제<br>- 기본 값이 설정된 필드 추가|마지막 버전|Consumer
FORWARD|이전 스키마에서 새로운 데이터를 읽는 것이 가능하다.|- 필드 추가<br>- 기본 값이 설정된 필드 삭제|마지막 버전|Producer
FULL|BACKWARD 와 FORWARD 모두 만족 한다.|- 기본 값이 설정된 필드 추가<br>- 기본 값이 설정된 필드 삭제|마지막 버전|Any order
NONE|BACKWARD 와 FORWARD 모두 만족하지 않는다.(어떠한 변경이든 허용)|모두 가능|판단필요

`Consumer` 를 항상 선 변경 할 수 있다면 `BACKWARD` 기본 값을 사용하면 된다. 
하지만 `Producer` 가 선 변경해야 하는 경우가 있다면 `FULL` 을 사용하는 것이 좋다.  

위의 내용을 정리하면 `Schema Registry` 를 사용해서 스키마를 관리할 때 아래와 같은 사항들은 고려해야 한다. 

- 삭제가 될 수 있는 필드라면(키와 같은 필드가 아닌) 기본 값을 설정해야 한다. 
- 새로 추가하는 필드라면 기본 값을 추가해야 한다.
- `Enum` 과 같이 타입성의 필드는 변경 가능성이 없는 경우에만 사용해야 한다. 
- 필드 이름 변경은 최대한 피해야 한다. 


### 장단점
장점을 나열하면 아래와 같다. 

- 메시지 스키마 변경을 보다 안정적으로 할 수 있다. 
- 잘못된 메시지가 `Consumer` 로 까지 전달되지 않는다. 
- 메시지 스키마 정보는 `Schema Registry` 에서 가져오고 `Kafka` 에는 `Binary Data` 를 전송하기 때문에 상대적으로 적은 용량을 전송한다.

단점을 나열하면 아래와 같다. 

- `Schema Registry` 의 장애가 메시지 생산, 소비의 장애로 이어진다.
- `Kafka` 와 함께 안정성을 보장해야 할 운영 포인트가 증가한다.


### 스키마 포맷
`Confluent Schema Registry` 에서 보편적으로 사용하는 `Schema Format` 은 [Apache Avro](https://avro.apache.org/)
이다. 

다른 스키마 포맷도 사용 가능한데, 그 종류는 아래와 같다. 

Format|Producer|Consumer|desc
---|---|---|---
Avro|io.confluent.kafka.serializers.KafkaAvroSerializer|io.confluent.kafka.serializers.KafkaAvroDeserializer|[설명](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/serdes-avro.html)
ProtoBuf|io.confluent.kafka.serializers.protobuf.KafkaProtobufSerializer|io.confluent.kafka.serializers.protobuf.KafkaProtobufDeserializer|[설명](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/serdes-protobuf.html)
JSON Schema|io.confluent.kafka.serializers.json.KafkaJsonSchemaSerializer|io.confluent.kafka.serializers.json.KafkaJsonSchemaDeserializer|[설명](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/serdes-json.html)

이후 진행되는 예제에서는 `Avro` 를 사용해서 진행한다.  


### 설치와 사용해보기
`Schema Registry` 의 동작에 대한 내용은 간단하게 `REST API` 를 통해 확인 해 볼 수 있다.
`Docker` 와 `Docker Compose` 기반으로 구성할 예제인데 전체 구성 정보가 작성된 `docker-compose.yaml` 파일 내용은 아래와 같다.



```yaml
version: '3'

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
    depends_on:
      - zookeeper

  schemaRegistry:
    container_name: schemaRegistry
    image: confluentinc/cp-schema-registry:5.1.0
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: zookeeper:2181
      SCHEMA_REGISTRY_HOST_NAME: schemaRegistry
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    depends_on:
      - zookeeper
      - kafka
```  

그리고 아래 명령으로 전체 구성을 실행해 준다.  

```bash
$ docker-compose up --build

[+] Running 3/2
 ⠿ Container myZookeeper     Created                                                                               0.1s
 ⠿ Container myKafka         Created                                                                               0.0s
 ⠿ Container schemaRegistry  Created                                                                               0.0s
Attaching to myKafka, myZookeeper, schemaRegistry

```  

그리고 아래 요청으로 `Schema Registry` 가 정상 동작중인지 확인 한다. 
이작 등록된 스키마가 없기 때문에 빈 배열이 응답으로 온다면 정상 동작 중인 것이다.  

```bash
$ curl -X GET http://localhost:8081/subjects | jq

[]
```  

#### 스키마 추가

`Schema Registry` 는 `REST API` 로 스키마 확인 등록 삭제 등이 가능하다. 
먼저 `myTest` 라는 이름의 `Avro` 스키마를 등록해 볼껀데 스키마 정의는 아래와 같다.  

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

`POST` 요청을 통해 아래와 같이 등록할 수 있다.  

```bash
$ curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
--data '{
"schema":"{\"namespace\":\"com.windowforsun.test\",\"type\":\"record\",\"name\":\"myTest\",\"fields\":[{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"}]}"
}' \
http://localhost:8081/subjects/myTest/versions | jq

{
  "id": 1
}
```  

#### 스키마 이름 조회

```bash
$ curl -X GET http://localhost:8081/subjects | jq

[
  "myTest"
]
```  

#### 스키마 버전 조회

```bash
$ curl -X GET http://localhost:8081/subjects/myTest/versions/ | jq

[
  1
]
```  

#### 스키마 정보 조회

```bash
$ curl -X GET http://localhost:8081/subjects/myTest/versions/1 | jq

{
  "subject": "myTest",
  "version": 1,
  "id": 1,
  "schema": "{\"type\":\"record\",\"name\":\"myTest\",\"namespace\":\"com.windowforsun.test\",\"fields\":[{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"}]}"
}
```  

#### 스키마 변경
`Schema Registry` 의 기본 호환 타입은 `BACKWARD` 이다. 
먼저 호환 타입과 맞지 않는 변경을 수행하면 아래와 같은 메시지가 응답 된다. 
`BACKWARD` 호환 타입에서 허용하지 않는 변경인 기본 값이 설정되지 않은 필드 추가를 수행해 본다.  

```bash
$ curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
--data '{
"schema":"{\"namespace\":\"com.windowforsun.test\",\"type\":\"record\",\"name\":\"myTest\",\"fields\":[{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"},{\"name\":\"newName\",\"type\":\"string\"}]}"
}' \
http://localhost:8081/subjects/myTest/versions | jq

{
  "error_code": 409,
  "message": "Schema being registered is incompatible with an earlier schema"
}
```  

위와 같이 `409` 응답으로 변경이 거부 된다.  

`BACKWARD` 에서 호환하는 범위의 변경으로 아래와 같이 스키마를 변경한다. 
`age` 를 삭제하고 기본 값이 설정된 `address` 필드가 추가된다.  

```json
{
  "namespace":"com.windowforsun.test",
  "type":"record",
  "name":"myTest",
  "fields":[
    {"name":"name","type":"string"},
    {"name":"address","type":"string","default":""}
  ]
}
```  

```bash
$ curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
> --data '{
quote> "schema":"{\"namespace\":\"com.windowforsun.test\",\"type\":\"record\",\"name\":\"myTest\",\"fields\":[{\"name\":\"name\",\"type\":\"str
ing\"},{\"name\":\"address\",\"type\":\"string\",\"default\":\"\"}]}"
quote> }' \
> http://localhost:8081/subjects/myTest/versions | jq

{
  "id": 2
}

$ curl -X GET http://localhost:8081/subjects/myTest/versions/ | jq

[
  1,
  2
]
```  

위와 같이 `myTest` 스키마에 새로운 버전이 추가 된 것을 확인 할 수 있다.  


#### 스키마 호환 타입 조회 및 변경
현재 `myTest` 스키마는 별도로 호환 타입 설정을 해주지 않았기 때문에 기본 호환 타입을 따른다.  

```bash
$ curl -X GET http://localhost:8081/config | jq

{
  "compatibilityLevel": "BACKWARD"
}
```  

테스트를 위해 아래 요청으로 호환 타입을 `NONE` 으로 변경해 준다.  

```bash
$ curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
--data '{"compatibility": "NONE"}' \
http://localhost:8081/config/myTest | jq

{
  "compatibility": "NONE"
}
```  

그리고 `myTest` 의 호환 타입을 조회하면 `NONE` 으로 조회되는 것을 확읺 할 수 있다.  

```bash
$ curl -X GET http://localhost:8081/config/myTest | jq

{
  "compatibilityLevel": "NONE"
}
```  

`NONE` 이기 때문에 어떠한 변경도 가능하다. 
기존에 있던 모든 필드를 다 지우고, 기본 값이 없는 새로운 필드를 추가해 본다.  

```bash
$ curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
--data '{
"schema":"{\"namespace\":\"com.windowforsun.test\",\"type\":\"record\",\"name\":\"myTest\",\"fields\":[{\"name\":\"test1\",\"type\":\"string\"},{\"name\":\"test2\",\"type\":\"string\"}]}"
}' \
http://localhost:8081/subjects/myTest/versions | jq

{
  "id": 3
}
```  


### Producer, Consumer 연동
이번에는 간단한 `Producer`, `Consumer` 애플리케이션을 `Java` 기반으로 구현해서 
`Schema Registry` 와 연동해 메시지를 주고 받는 예제를 진행해 본다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-schema-registry-4.drawio.png)

공통으로 사용되는 `buid.gradle` 내용은 아래와 같다.  

```groovy
plugins {
    id 'java'
}

version '1.0-SNAPSHOT'

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
    implementation 'io.confluent:kafka-avro-serializer:7.0.1'
    implementation "org.apache.avro:avro:1.10.2"
}

test {
    useJUnitPlatform()
}
```  

`Producer` 애플리케이션 코드는 아래와 같다. 
`avro-employee` 라는 토픽을 사용하고 `Schema Registry` 에 스키마를 등록하고, 
3초 간격으로 총 10개의 데이터를 생산한다.  

```java
public class AvroProducer {
    private final static String TOPIC = "avro-employee";
    private final static String FIRST_SCHEMA =
                    "  {" +
                    "    \"namespace\": \"com.windowforsun.exam\"," +
                    "    \"type\": \"record\"," +
                    "    \"name\": \"Employee\"," +
                    "    \"fields\": [" +
                    "        {\"name\": \"firstName\", \"type\": \"string\"}," +
                    "        {\"name\": \"lastName\", \"type\": \"string\"}," +
                    "        {\"name\": \"age\", \"type\": \"int\"}" +
                    "    ]" +
                    "  }";


    public static Producer<Long, GenericRecord> createProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "AvroProducer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, LongSerializer.class.getName());
        // config KafkaAvroSerializer
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());
        // schema registry location
        props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        return new KafkaProducer<>(props);
    }

    public static void main(String... args) {
        Producer<Long, GenericRecord> producer = createProducer();

        try {
            for (int i = 0; i < 10; i++) {
                // send v1 schema data
                producer.send(getFirstSchemaData());

                TimeUnit.SECONDS.sleep(3);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.flush();
            producer.close();
        }
    }

    public static ProducerRecord<Long, GenericRecord> getFirstSchemaData() {
        Random keyRandom = new Random();
        Schema.Parser parser = new Schema.Parser();
        Schema schema = parser.parse(FIRST_SCHEMA);
        GenericRecord employeeAvroRecord = new GenericData.Record(schema);

        employeeAvroRecord.put("firstName", "Bob");
        employeeAvroRecord.put("lastName", "Jones");
        employeeAvroRecord.put("age", 20);

        return new ProducerRecord<>(TOPIC, keyRandom.nextLong(), employeeAvroRecord);
    }
}
```  

> 구동 환경인 `Mac M1 Pro podman docker` 환경에서는 Java 애플리케이션에서 `localhost 로 `Kafka` 에 접속하기 위해서 아래와 같은 처리를 해주었다. 
>
> ```
> vi /etc/hosts
> 127.0.0.1 kafka
> ```

`Consumer` 코드는 아래와 같다. 
`avro-employee` 토픽의 데이터를 읽어 `Schema Registry` 에서 스키마 정보를 얻어와 최종 데이터로 변환해 출력한다.  

```java
public class AvroConsumer {
    private final static String TOPIC = "avro-employee";

    public static Consumer<Long, GenericRecord> createConsumer() {
        Properties prop = new Properties();
        prop.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        prop.put(ConsumerConfig.GROUP_ID_CONFIG, String.format("AvroConsumer-%s", UUID.randomUUID().toString()));
        prop.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, LongDeserializer.class.getName());
        prop.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        // kafka avro deserializer
        prop.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class.getName());
        // specific record or else get avro GenericRecord
//        prop.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, "true");
        prop.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        return new KafkaConsumer<>(prop);
    }

    public static void main(String... args) {
        Consumer<Long, GenericRecord> consumer = createConsumer();
        consumer.subscribe(List.of(TOPIC));

        try {
            while(true) {
                ConsumerRecords<Long, GenericRecord> records = consumer.poll(Duration.ofSeconds(10));

                if(records.isEmpty()) {
                    break;
                }

                for(ConsumerRecord<Long, GenericRecord> record : records) {
                    System.out.printf("consume offset=%d, key=%d, value=%s\n", record.offset(), record.key(), record.value());
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```  

`Produer` 를 실행하고 이어서 `Consumer` 를 실행하면, 
`Consumer` 애플리케이션 출력에서 아래와 같이 `Producer` 의 메시지가 정상적으로 `Consumer` 까지 전달 된 것을 확인 할 수 있다.  

```
consume offset=0, key=4377525503399672603, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=1, key=-6580271613910437641, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=2, key=-6222654254634233669, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=3, key=-3639595488384231473, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=4, key=7477665076072471481, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=5, key=-7572765000403818358, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=6, key=679796825054442659, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=7, key=-5007053102214549516, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=8, key=-8505523601764608954, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=9, key=-7532690658205931149, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
```  

`Schema Registry` 에 등록된 스키마를 조회하면 아래와 같이 `employee` 스키마가 등록된 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:8081/subjects | jq

[
  "avro-employee-value",
  "myTest"
]

$ curl -X GET http://localhost:8081/subjects/avro-employee-value/versions | jq

[
  1
]
```  

`avro-employee` 토픽의 실제 데이터는 `Deserialize` 동작 없이는 확인 할 수 없다.   

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic avro-employee --from-beginning

Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
Bob
Jones(
```  

이번에는 `Producer` 에서 새로운 스키마와 기존 스키마를 동시에 사용하는 상황을 구현해 본다.  

```java
public class AvroProducer {
    private final static String TOPIC = "avro-employee";
    private final static String FIRST_SCHEMA =
                    "  {" +
                    "    \"namespace\": \"com.windowforsun.exam\"," +
                    "    \"type\": \"record\"," +
                    "    \"name\": \"Employee\"," +
                    "    \"fields\": [" +
                    "        {\"name\": \"firstName\", \"type\": \"string\"}," +
                    "        {\"name\": \"lastName\", \"type\": \"string\"}," +
                    "        {\"name\": \"age\", \"type\": \"int\"}" +
                    "    ]" +
                    "  }";

    // 추가
    // age 삭제, 기본 값이 ""인 phoneNumber 추가
    private final static String SECOND_SCHEMA =
                    "  {" +
                    "    \"namespace\": \"com.windowforsun.exam\"," +
                    "    \"type\": \"record\"," +
                    "    \"name\": \"Employee\"," +
                    "    \"fields\": [" +
                    "        {\"name\": \"firstName\", \"type\": \"string\"}," +
                    "        {\"name\": \"lastName\", \"type\": \"string\"}," +
                    "        {\"name\": \"phoneNumber\", \"type\": \"string\", \"default\": \"\"}" +
                    "    ]" +
                    "  }";


    public static Producer<Long, GenericRecord> createProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "AvroProducer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, LongSerializer.class.getName());
        // config KafkaAvroSerializer
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());
        // schema registry location
        props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        return new KafkaProducer<>(props);
    }

    public static void main(String... args) {
        Producer<Long, GenericRecord> producer = createProducer();

        try {
            for (int i = 0; i < 10; i++) {
                // send v1 schema data
                producer.send(getFirstSchemaData());
                // 추가
                // send v2 schema data
                producer.send(getSecondSchemaData());

                TimeUnit.SECONDS.sleep(3);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.flush();
            producer.close();
        }
    }

    public static ProducerRecord<Long, GenericRecord> getFirstSchemaData() {
        Random keyRandom = new Random();
        Schema.Parser parser = new Schema.Parser();
        Schema schema = parser.parse(FIRST_SCHEMA);
        GenericRecord employeeAvroRecord = new GenericData.Record(schema);

        employeeAvroRecord.put("firstName", "Bob");
        employeeAvroRecord.put("lastName", "Jones");
        employeeAvroRecord.put("age", 20);

        return new ProducerRecord<>(TOPIC, keyRandom.nextLong(), employeeAvroRecord);
    }

    // 추가
    public static ProducerRecord<Long, GenericRecord> getSecondSchemaData() {
        Random keyRandom = new Random();
        Schema.Parser parser = new Schema.Parser();
        Schema schema = parser.parse(SECOND_SCHEMA);
        GenericRecord employeeAvroRecord = new GenericData.Record(schema);

        employeeAvroRecord.put("firstName", "Bob");
        employeeAvroRecord.put("lastName", "Jones");
        employeeAvroRecord.put("phoneNumber", "01012341234");

        return new ProducerRecord<>(TOPIC, keyRandom.nextLong(), employeeAvroRecord);
    }
}
```  

`Consumer` 는 코드 수정 없이 그대로 사용하면 된다. 
이전과 같이 `Producer` 실행 후 `Consumer` 를 실행하면 `Consumer` 출력에서 2개의 스키마가 함께 사용되는 것을 확인 할 수 있다.  

```
consume offset=10, key=5162990227792807001, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=11, key=7198787782085740299, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=12, key=3682103131243450030, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=13, key=-5597383093141652803, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=14, key=-1407532733639755087, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=15, key=-8588484599451141121, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=16, key=4910995462763244864, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=17, key=2790224350919717534, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=18, key=-7175551501632317176, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=19, key=4150782920890993423, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=20, key=-8164567881896546062, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=21, key=5560543272937157978, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=22, key=-1030167455531010845, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=23, key=-1780545764043312743, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=24, key=-2503234676346411331, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=25, key=-3836746708290016110, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=26, key=-6568557259707343094, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=27, key=9003268337515304566, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
consume offset=28, key=-672179222903661939, value={"firstName": "Bob", "lastName": "Jones", "age": 20}
consume offset=29, key=6918897648799820622, value={"firstName": "Bob", "lastName": "Jones", "phoneNumber": "01012341234"}
```  

#### specific.avro.reader

`Consumer` 코드에서 주의해야할 옵션이 있는데 주석처리돼 있는 아래 코드가 있다. 

```java

// specific record or else get avro GenericRecord
prop.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, "true");
```  

`specific.avro.reader` 옵션의 기본 값은 `false` 인데, 
현재 구현채에서 해당 옵션 값을 `true` 로 줄 경우 아래와 같은 에러가 발생한다.  

```
Exception in thread "main" org.apache.kafka.common.errors.RecordDeserializationException: Error deserializing key/value for partition avro-employee-0 at offset 0. If needed, please seek past the record to continue consumption.
	at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1429)
	at org.apache.kafka.clients.consumer.internals.Fetcher.access$3400(Fetcher.java:134)
	at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.fetchRecords(Fetcher.java:1652)
	at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.access$1800(Fetcher.java:1488)
	at org.apache.kafka.clients.consumer.internals.Fetcher.fetchRecords(Fetcher.java:721)
	at org.apache.kafka.clients.consumer.internals.Fetcher.fetchedRecords(Fetcher.java:672)
	at org.apache.kafka.clients.consumer.KafkaConsumer.pollForFetches(KafkaConsumer.java:1304)
	at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1238)
	at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1211)
	at com.windowforsun.schemaregistry.AvroConsumer.main(AvroConsumer.java:38)
Caused by: org.apache.kafka.common.errors.SerializationException: Could not find class com.windowforsun.exam.Employee specified in writer's schema whilst finding reader's schema for a SpecificRecord.
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.getSpecificReaderSchema(AbstractKafkaAvroDeserializer.java:281)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.getReaderSchema(AbstractKafkaAvroDeserializer.java:252)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.getDatumReader(AbstractKafkaAvroDeserializer.java:196)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer$DeserializationContext.read(AbstractKafkaAvroDeserializer.java:391)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.deserialize(AbstractKafkaAvroDeserializer.java:114)
	at io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer.deserialize(AbstractKafkaAvroDeserializer.java:88)
	at io.confluent.kafka.serializers.KafkaAvroDeserializer.deserialize(KafkaAvroDeserializer.java:55)
	at org.apache.kafka.common.serialization.Deserializer.deserialize(Deserializer.java:60)
	at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1420)
	... 9 more
```  

`false` 인 경우 `Consumer` 에서 `Procuder` 가 메시지를 생산할 때 사용한 스키마 정보를 그대로 사용한다는 의미이다. 
반대로 `true` 인 경우에는 `Consumer` 에서 지정된 버전의 스키마 정보만 사용한다는 의미이다. 
즉 구현한 애플리케이션에서는 지정된 스키마 버전이 없기 때문에 위와 같은 에러가 발생하는 것이다.  

추가적으로 `specific.avro.reader` 가 `true` 인 경우에는 내부적으로 `GenericRecord` 가 아닌, 
`SpecificRecord` 를 사용하게 되는데 `SpecificRecord` 는 조회한 스키마는 캐시에 담아두고 재사용하게 된다. 
히자만 반대로 `specific.avro.reader` 가 `false` 일 때는 `GenericRecord` 를 사용하고 캐시도 사용하지 않는다. 





---  
## Reference
[Kafka, Avro Serialization, and the Schema Registry](https://dzone.com/articles/kafka-avro-serialization-and-the-schema-registry)  
[AVRO - Schemas](https://www.tutorialspoint.com/avro/avro_schemas.htm)  
[Avro Specification](https://avro.apache.org/docs/1.11.1/specification/)  
[Why Avro for Kafka Data?](https://www.confluent.io/blog/avro-kafka-data/)  
[Introduction to Schema Registry in Kafka](https://medium.com/slalom-technology/introduction-to-schema-registry-in-kafka-915ccf06b902)  
[UNDERSTANDING APACHE AVRO: AVRO INTRODUCTION FOR BIG DATA AND DATA STREAMING ARCHITECTURES](http://cloudurable.com/blog/avro/index.html)  
[Schema Registry - Formats, Serializers, and Deserializers](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/index.html)  
[Avro Schema Serializer and Deserializer](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/serdes-avro.html)  
[Kafka Schema Registry & Avro: Introduction](https://www.lydtechconsulting.com/blog-kafka-schema-registry-intro.html)  
[Introduction to Schemas in Apache Kafka with the Confluent Schema Registry](https://medium.com/@stephane.maarek/introduction-to-schemas-in-apache-kafka-with-the-confluent-schema-registry-3bf55e401321)  
[Kafka Ecosystem at LinkedIn](https://engineering.linkedin.com/blog/2016/04/kafka-ecosystem-at-linkedin)  
