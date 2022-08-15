--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Stream Serialize/Deserialize 커스텀 메시지 사용"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '커스텀 메시지를 사용하는 경우 Spring Cloud Stream 에 Serialize/Deserialize 처리를 수행하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Stream
    - StreamBridge
    - Kafka
    - MSA
    - Serialize
    - Deserialize
toc: true
use_math: true
---  

## Spring Cloud Stream Serialize/Deserialize
[Spring Cloud Stream 기본 사용]()
에서는 `Spring Cloud Strema` 에 대한 간단한 설명과 메시징 애플리케이션의 구현 방법에 대해서 알아보았다. 
앞선 포스트에서는 단순히 문자열, 숫자형에 대한 메시지를 주고 받았다면, 
사용자 정의 형식의 메시지를 `Spring Cloud Stream` 에서 사용하는 방법에 대해 알아볼 것이다.  

물론 이후 설명하는 방법이 아니더라도, 
문자열 타입으로 설정해두고 해당 문자열을 `Producer`, `Consumer` 의 구현체에서 각각 `Serialize`, `Deserialize` 처리하는 로직을 넣어도 구현은 가능하다. 
하지만 이번 포스트에서는 `Spring Cloud Stream` 에서 제공하는 기능을 사용해서 이러한 동작 수행 방법에 대해 알아보는 것이 목적이다.  

여기서 `Producer` 애플리케이션에서는 메시지를 `Serialize` 해서 `Kafka` 을 통해 방출하게 되고, 
`Consumer` 애플리케이션에서는 `Kafka` 로 부터 수신한 메시지를 `Deserialize` 하게 된다.  

`Kafka` 구현, `build.gradle` 는 모두 이전 포스트와 동일하기 때문에 관련 설명은 동일하기 때문에 사용된 실제 코드만 기재한다.  

- `kafka docker-compose.yaml`

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
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```  

- `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '2.6.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
group 'com.windowforsun.cloudstream'
version '1.0-SNAPSHOT'

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
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    compileOnly "org.projectlombok:lombok"
    testCompileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
    testAnnotationProcessor "org.projectlombok:lombok"

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

### Producer(Serialize)

#### MyMessage
`MyMessage` 는 `Spring Cloud Stream` 메시지으 커스텀 타입이다. 

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class MyMessage {
    private String str;
    private Integer num;
    private List<String> strList;
}
```  

#### MyMessageSerializer
`MyMessage` 의 직렬화 로직이 있는 구현체는 아래와 같다. 
예제에서는 `ObjectMapper` 를 사용하고 있지만, 
애플리케이션 특정에 맞춰 원하는 방식으로 자유롭게 사용할 수 있다. 

```java
import org.apache.kafka.common.serialization.Serializer;

@Slf4j
public class MyMessageSerializer implements Serializer<MyMessage> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public byte[] serialize(String topic, MyMessage data) {
        try {
            log.info("serialize : {}, {}", topic, data);
            return this.objectMapper.writeValueAsBytes(data);
        } catch(Exception e) {
            throw new SerializationException(e);
        }
    }
}
```  

#### SerializeProducer
`SerializeProducer` 에서 방출 된 메시지는 `MyMessageSerializer` 로 전달되어 직렬화가 수행된다. 
즉 직렬화 로직이 변경된다고 하더라도 역할에 따라 분리된 `MyMessageSerializer` 만 수정하거나 추가로 개발해주면 되고, 
실제 `Producer` 에 대한 변경은 발생하지 않는 다는 점이 있다. 

```java
@Service
@RequiredArgsConstructor
public class SerializeProducer {
    private final StreamBridge streamBridge;
    private static MyMessage LAST_MY_MESSAGE = null;

    public void produce(MyMessage myMessage) {
        LAST_MY_MESSAGE = myMessage;
        this.streamBridge.send("myMessageProducer", myMessage);
    }

    public static MyMessage getLastMyMessage() {
        return LAST_MY_MESSAGE;
    }
}
```  

#### SerializeProducerController
이번 예제에서도 동일하게 웹요청을 통해 테스트를 진행할 계획이므로 테스트에 필요한 엔드포인트를 등록해 준다.  

```java
@RestController
@RequiredArgsConstructor
public class SerializeProducerController {
    private final SerializeProducer serializeProducer;

    @GetMapping("/produce")
    public MyMessage produce(@RequestParam("str") String str,
                             @RequestParam("num") Integer num,
                             @RequestParam("strList") String strList) {
        MyMessage myMessage = MyMessage.builder()
                .str(str)
                .num(num)
                .strList(Arrays.asList(strList.split(",")))
                .build();

        this.serializeProducer.produce(myMessage);

        return myMessage;
    }

    @GetMapping("/myMessage")
    public MyMessage getMeMessage() {
        return SerializeProducer.getLastMyMessage();
    }
}
```  

#### application.yaml
`spring.cloud.stream.bindings.<bindingName>.producer.use-native-encoding` 의 값을 `true` 값으로 설정해서, 
커스텀한 `Serialize` 가 사용될 수 있도록 설정한다. 
그리고 `spring.cloud.stream.kafka.bindings.<bindingName>.producer.configuration.value.serializer` 에 
`Serialize` 구현이 있는 패키지명을 포함한 클래스 이름을 등록해 준다.  

```yaml
server:
  port: 8071

spring:
  cloud:
    stream:
      bindings:
        myMessageProducer:
          destination: my-message-topic
          producer:
            # custom serializer 활성화
            use-native-encoding: true
      kafka:
        binder:
          brokers: localhost:9092
        # producer 에서 사용할 serializer, deserializer 설정
        bindings:
          myMessageProducer:
            producer:
              configuration:
                value:
                  serializer: com.windowforsun.cloudstream.serializeproducer.MyMessageSerializer

```  

#### SerializeProducerTest
아래와 같은 방법으로 `Custom Serialize` 가 추가된 `Proudcer` 도 `Unit Test` 가 가능한데, 
한가지 주의해야 할 점은 아래 처럼 테스트를 할 경우 테스트 실행 과정에서는 `Serializer` 가 호출 되지 않는 다는 점이다. 


```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class SerializeProducerTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private SerializeProducer serializeProducer;

    @Test
    public void produce() {
        // given
        MyMessage myMessage = MyMessage.builder()
                .str("hello!")
                .num(1234)
                .strList(List.of("a", "1", "b", "2"))
                .build();

        // when
        this.serializeProducer.produce(myMessage);

        // then
        Message<byte[]> result = this.outputDestination.receive(100, "my-message-topic");
        MyMessage actual = (MyMessage) ((Object)result.getPayload());
        assertThat(actual, notNullValue());
        assertThat(actual.getStr(), is("hello!"));
        assertThat(actual.getNum(), is(1234));
        assertThat(actual.getStrList(), contains("a", "1", "b", "2"));
    }
}
```  

### Consumer(Deserialize)

#### MyMessage
`Consumer` 에도 `Producer` 와 동일한 메시지 클래스는 정의돼야 한다. 

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class MyMessage {
    private String str;
    private Integer num;
    private List<String> strList;
}
```  

#### MyMessageDeserializer
`Producer` 에서 `MyMessage` 를 `ObjectMapper` 로 직렬화 수행했기 때문에, 
`Consumer` 에서도 호환 될 수 있도록 동일하게 `ObjectMapper` 를 사용해서 역직렬화를 수행해 준다.  


```java
@Slf4j
public class MyMessageDeserializer implements Deserializer<MyMessage> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public MyMessage deserialize(String topic, byte[] data) {
        try {
            log.info("deserialize : {}, {}", topic, data);
            return this.objectMapper.readValue(new String(data), MyMessage.class);
        } catch(Exception e) {
            throw new SerializationException(e);
        }
    }
}
```  

#### SerializeConsumer
`Consumer` 는 `MyMessageDeserializer` 에서 역질렬화 돼서 오브젝트로 변환된 객체가 `SerializeConsumer` 로 전달되고 소비된다. 

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SerializeConsumer {
    private static MyMessage LAST_MY_MESSAGE = null;

    @Bean
    public Consumer<MyMessage> myMessageConsumer() {
        return myMessage -> {
            LAST_MY_MESSAGE = myMessage;

            log.info("myMessageConsumer : {}", LAST_MY_MESSAGE);
        };
    }

    public static MyMessage getLastMyMessage() {
        return LAST_MY_MESSAGE;
    }
}
```  

#### SerializeConsumerController
`Consumer` 에도 이후 테스트를 위해 가장 마지막 메시지 조회용 엔드포인트를 등록해 준다.  

```java
@RestController
@RequiredArgsConstructor
public class SerializeConsumerController {
    private final SerializeConsumer serializeConsumer;

    @GetMapping("/myMessage")
    public MyMessage getMeMessage() {
        return SerializeConsumer.getLastMyMessage();
    }
}
```  

#### application.yaml
`Consumer` 설정은 `Producer` 설정과 대부분 비슷하다. 
`spring.cloud.stream.bindings.<bindingName>.producer.use-native-encoding` 의 값을 `true` 값으로 설정해서,
커스텀한 `Deserialize` 가 사용될 수 있도록 설정한다.
그리고 `spring.cloud.stream.kafka.bindings.<bindingName>.producer.configuration.value.deserializer` 에
`Deserialize` 구현이 있는 패키지명을 포함한 클래스 이름을 등록해 준다.  

차이점은 `Consumser` 는 빈 등록으로 사용되기 때문에 `spring.cloud.function.definition` 에 빈이름을 등록해주고, 
이후에 사용하는 `bindingName` 또한 `myMessageConsumer-in-0` 과 같은 정해진 규칙에 맞는 방식으로 지어 줘야 한다는 점이다.  

```yaml
server:
  port: 8072

spring:
  cloud:
    function:
      # consumer bean 이름
      definition: myMessageConsumer
    stream:
      bindings:
        # consumer bean 이름을 기반으로 생성할 바인딩 이름 <빈이름>-<in/out>-<index>
        myMessageConsumer-in-0:
          destination: my-message-topic
          consumer:
            # custom deserializer 활성화
            use-native-decoding: true
      kafka:
        binder:
          brokers: localhost:9092
        # consumer 에서 사용할 serializer, deserializer 설정
        bindings:
          myMessageConsumer-in-0:
            consumer:
              configuration:
                value:
                  deserializer: com.windowforsun.cloudstream.serializeconsumer.MyMessageDeserializer
```  

#### SerializeConsumerTest
`Consumer` 테스트 또한 `Unit Test` 수행시에는 등록된 `Deserializer` 가 실행되지 않는다. 
방법은 `inputDestination` 을 통해 원하는 메시지를 구성해서 방출하고 구현한 `Consumer` 동작을 확인해 주면 된다.  

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class SerializeConsumerTest {
    @Autowired
    private InputDestination inputDestination;

    @Test
    public void consumer() {
        // given
        MyMessage myMessage = MyMessage.builder()
                .str("hello!")
                .num(1234)
                .strList(List.of("a", "1", "b", "2"))
                .build();

        // when
        this.inputDestination.send(new GenericMessage<>(myMessage), "my-message-topic");

        // then
        MyMessage actual = SerializeConsumer.getLastMyMessage();
        assertThat(actual, notNullValue());
        assertThat(actual.getStr(), is("hello!"));
        assertThat(actual.getNum(), is(1234));
        assertThat(actual.getStrList(), contains("a", "1", "b", "2"));
    }
}
```  

### 통합 테스트
테스트는 `Kafka` 를 실행하고 `Producer`, `Consumer` 를 모두 실행한 상태에서 진행한다.

먼저 `/myMessage` 요청을 통해 현재 `Producer`, `Consumer` 에 마지막으로 설정된 값을 확인하면 아래와 같다.
참고로 `Producer` 는 `8071` 포트, `Consumer` 는 `8072` 포트를 사용한다.  

```bash
$ curl localhost:8071/myMessage
$ curl localhost:8072/myMessage
```  

아직 아무런 메시지가 설정되지 않은 상태이기 때문에 `null` 이 응답이 오는 상태이다.  

이제 `Producer` 를 사용해서 특정 `MyMessage` 메시지를 방출하고 나서 다시 확인하면 아래와 같다.  

```bash
$ curl http://localhost:8071/produce?str=myStr&num=11&strList=111111,a,2,b
{
  "str": "myStr",
  "num": 11,
  "strList": [
    "111111",
    "a",
    "2",
    "b"
  ]
}

$ curl localhost:8071/myMessage
{
  "str": "myStr",
  "num": 11,
  "strList": [
    "111111",
    "a",
    "2",
    "b"
  ]
}
$ curl localhost:8072/myMessage
{
  "str": "myStr",
  "num": 11,
  "strList": [
    "111111",
    "a",
    "2",
    "b"
  ]
}
```  

메시지를 모두 정상적으로 수신한 것을 확인 할 수 있다. 
이때 각 애플리케이션의 로그를 확인하면 아래와 같다. 

```
.. producer ..
INFO 13268 --- [nio-8071-exec-5] c.w.c.s.MyMessageSerializer              : serialize : my-message-topic, MyMessage(str=myStr, num=11, strList=[111111, a, 2, b])

.. consumer ..
INFO 29368 --- [container-0-C-1] c.w.c.s.MyMessageDeserializer            : deserialize : my-message-topic, [123, 34, 115, 116, 114, 34, 58, 34, 109, 121, 83, 116, 114, 34, 44, 34, 110, 117, 109, 34, 58, 49, 49, 44, 34, 115, 116, 114, 76, 105, 115, 116, 34, 58, 91, 34, 49, 49, 49, 49, 49, 49, 34, 44, 34, 97, 34, 44, 34, 50, 34, 44, 34, 98, 34, 93, 125]
INFO 29368 --- [container-0-C-1] c.w.c.s.SerializeConsumer                : myMessageConsumer : MyMessage(str=myStr, num=11, strList=[111111, a, 2, b])
```  

---  
## Reference
[Spring Cloud Stream](https://spring.io/projects/spring-cloud-stream)  
[Spring Cloud Stream With Kafka](https://refactorfirst.com/spring-cloud-stream-with-kafka-communication.html)  
[Streaming with Spring Cloud](https://medium.com/walmartglobaltech/streaming-with-spring-cloud-24a001ad307a)  
[Unit Testing a Spring Cloud Stream Producer / Publisher](https://medium.com/@sumant.rana/unit-testing-a-spring-cloud-stream-producer-publisher-ecf39d29ea13)  
