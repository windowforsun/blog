--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Stream 기본 사용"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '메시징 기반 애플리케이션 구현을 지원해주는 Spring Cloud Stream 과 사용법에 대해 알아보자'
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
toc: true
use_math: true
---  

## Spring Cloud Stream
[Spring Cloud Stream](https://spring.io/projects/spring-cloud-stream) 
은 `Spring Cloud` 프로젝트에서 제공하는 여러 프레임워크 중 하나로, 
이벤트(메시지) 중심 마이크로 서비스를 구축하기 위한 프레임워크라고 할 수 있다. 
몇개의 어노테이션과 설정 작업과 같은 아주 간단한 구성으로 통해 이벤트 기능을 애플리케이션에 추가할 수 있다. 
그리고 `Kafka`, `RabbitMQ` 등과 같은 메시징 플랫폼을 지원한다. 그리고 이러한 세부 구현은 모두 추상화된 형태로 제공 되기 때문에, 
특정 플랫폼에 종속되지 않고 중립적 인터페이스를 통해 `Publisher-Consumer` 구조를 구현할 수 있도록 한다.  

이외 자세한 내용은 [여기 Reference](https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/spring-cloud-stream.html#spring-cloud-stream-overview-introducing)
에서 확인 할 수 있다.  

위와 같은 특징을 갖는 `Spring Cloud Stream` 은 다양한 구조와 상황에서 활용 가능할 걸로 보인다. 
`MSA` 구조에서 역할로 분리된 애플리케이션들 끼리 상호작용이 필요한 경우라던가, 
특정 이벤트에 맞춰 애플리케이션의 설정 혹은 동작 변경이 필요한 경우 등이 있을 것이다. 


## Spring Cloud Stream 메시징 구현
직접 `Spring Cloud Stream` 을 사용해서 애플리케이션을 구현해본다. 
구현할 내용은 아래와 같다. 

- `Producer` : 메시지를 생산하는 애플리케이션
- `Consumer` : 생산된 메시지를 소비해 필요한 처리를 수행하는 애플리케이션
- `Kafka` : `Producer` 와 `Consumer` 사이에서 메시지를 전달해주는 브로커 역할

`Producer` 와 `Consumer` 는 `Spring Boot Applictaion` 으로 구현하고, 
`Kafka` 는 `Docker` 와 `docker-compose` 를 사용해서 구성한다.  

애플리케이션은 최종적으로 웹요청을 통해 메시지를 생성하고 생성된 메시지는 잘 받는지에 대해서 테스트를 진행할 계획이다.  

`Spring Cloud Stream` 은 버전 `3.0` 이후와 이전로 사용 방법이 크게 다르다. 
`3.0` 버전이전에는 `@EnableBiding`, `@Output`, `@Input`, `@StreamListener` 과 같은 어노테이션을 사용해서 주된 설정이 이뤄졌다. 
하지만 본 포스트에서 다룰 버전인 `3.0` 버전은 위와 같은 어노테이션은 사용하지 않고, 
주로 `properties` 설정과 `StreamBridge` 를 사용해서 `Fundtional Integerface` 를 사용하는 방법으로 구현이 이뤄진다.  

가장 간단하면서 일반적인 `Producer`, `Consumer` 구현은 아래와 같이 `Functional Integerace` 구현체를 빈으로 등록하는 것이다. 
이렇게 `Producer` 와 `Consumer` 를 빈으로 등록해 사용할 경우 `spring.cloud.stream.function.definition` 에 빈 이름을 미리 설정해줘야 한다. 

```java
@Bean
public Supplier<String> producer() {
    return () -> "myMessage!!";
}

@Bean
public Consumer<String> consumer() {
    return message -> System.out.println("received " + message);
}
```  

`Producer` 의 경우에는 `StreamBridge` 를 사용해서 구현하는 방법도 있는데 이는 아래 예제에서 살펴보도록 한다.  

### Producer

#### build.gradle
`Spring Cloud Stream`, `Spring Boot Web`
그리고 `Spring Cloud Stream` 에 대한 `Junit` 테스트를 수행할 수 있도록 하는 의존성이 추가 돼 있다.  

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
    // spring cloud stream
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

#### ExamProducer
`ExamProducer` 는 `StreamBridge` 를 사용해서 미리 등록된 바인딩 이름으로 메시지를 방출하는 역할을 수행한다. 
그리고 가장 최근 방출한 메시지를 변수로 저장하고 이후 확인하는 용도로 사용할 계획이다.  

```java
@Service
@RequiredArgsConstructor
public class ExamProducer {
    private final StreamBridge streamBridge;
    private static String LAST_STR = "none";
    private static int LAST_NUM = -1;

    public void strProducer(String str) {
        LAST_STR = str;
        // 메시지 방출
        this.streamBridge.send("strProducer", str);
    }

    public void numProducer(int num) {
        LAST_NUM = num;
        // 메시지 방출
        this.streamBridge.send("numProducer", num);
    }

    public static String getLastStr() {
        return LAST_STR;
    }

    public static int getLastNum() {
        return LAST_NUM;
    }
}
```  

### ExamProducerController
`ExamProdcerController` 는 아래와 같은 엔드포인트와 그에 해당하는 역할을 갖는다. 

- `/str/{str}` : 문자열 메시지를 방출한다. 
- `/num/{num}` : 숫자형 메시지를 방출한다. 
- `/str` : 가장 최근에 방출된 문자열 메시지를 확인한다. 
- `/num` : 가장 최근에 방출된 숫자형 메시지를 확인한다. 

```java
@RestController
@RequiredArgsConstructor
public class ExamProducerController {
    private final ExamProducer examProducer;

    @GetMapping("/str")
    public String getStr() {
        return ExamProducer.getLastStr();
    }

    @GetMapping("/num")
    public int getNum() {
        return ExamProducer.getLastNum();
    }

    @GetMapping("/str/{str}")
    public String produceStr(@PathVariable String str) {
        this.examProducer.strProducer(str);

        return "done";
    }

    @GetMapping("/num/{num}")
    public String produceNum(@PathVariable Integer num) {
        this.examProducer.numProducer(num);

        return "done";
    }
}
```  

#### application.yaml
`Producer` 애플리케이션이 `8081` 포트로 웹요청을 받는 다는 설정을 제외하고는 모두 `Spring Cloud Stream` 에 대한 설정이다. 
`Producer` 를 빈으로 등록해서 사용하는 경우 `spring.coud.stream.function.definition` 에 설정이 필요하지만, 
현재 구현에서는 `StreamBrdige` 로만 사용하고 있기 때문에 관련 설정은 필요없다.  

주목해야 할 설정은 `spring.cloud.stream.bindings` 하위에 코드에서 `StreamBridge` 에 사용한 `bindingName` 인 `strProducer`, `numProducer` 를 
`Kafka` 의 토픽 이름와 매칭시켜 주는 부분이다. 
현재 `strProducer` 는 `str-topic`, `numProducer` 는 `num-topic` 과 매핑된 설정이다.  


```yaml
server:
  port: 8081

spring:
  cloud:
    # producer 빈 이름(빈으로 등록해서 사용하는게 없기 때문에 주석처리)
#    function:
#      definition: 
    stream:
      bindings:
        # producer bean 이름을 기반으로 생성할 바인딩 이름 <빈이름>-<in/out>-<index>
        strProducer:
          destination: str-topic
        numProducer:
          destination: num-topic
      kafka:
        binder:
          brokers: localhost:9092
```  

#### Producer Test
`Spring Cloud Stream` 에서 `Producer` 를 `Unit Test` 수행하는 방법이다. 
`TestChannelBinderConfiguration` 를 `@Import` 로 추가해 주고나서, 
`OutputDestination` 를 통해 `Producer` 에서 방출된 메시지를 토픽 이름을 명시해서 확인해 볼 수 있다.  

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class ExamProducerTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private ExamProducer examProducer;

    @Test
    public void strProducer() {
        // given
        this.examProducer.strProducer("hello!");

        // when
        Message<byte[]> result = outputDestination.receive(100, "str-topic");

        // then
        assertThat(result, is(notNullValue()));
        assertThat(new String(result.getPayload()), is("hello!"));
    }

    @Test
    public void numProducer() {
        // given
        this.examProducer.numProducer(1234);

        // when
        Message<byte[]> result = outputDestination.receive(100, "num-topic");

        // then
        assertThat(result, is(notNullValue()));
        assertThat(Integer.parseInt(new String(result.getPayload())), is(1234));
    }
}
```  


### Consumer

#### build.gradle
`Consumer` 의 `build.gradle` 는 `Producer` 와 동일하다.  

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
    // spring cloud stream
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

#### ExamConsumer
`Consumer` 는 `strConsumer`, `numConsumer` 를 빈으로 등록해서 사용한다. 
그리고 마지막에 수신한 메시지를 변수에 저장해서 추후 확인하는 용도로 사용할 계획이다.  

빈으로 등록한 `Consumer` 의 경우 `Spring Cloud Stream` 에서 빈과 토픽 바인등을 위해 `application.yaml` 에서 추가적인 설정이 필요하다.  

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ExamConsumer {
    private static String LAST_STR = "none";
    private static Integer LAST_NUM = -1;

    @Bean
    public Consumer<String> strConsumer() {
        return str -> {
            LAST_STR = str;

            log.info("strConsumer : {}", str);
        };
    }

    @Bean
    public Consumer<Integer> numConsumer() {
        return num -> {
            LAST_NUM = num;

            log.info("numConsumer : {}", num);
        };
    }

    public static String getLastStr() {
        return LAST_STR;
    }

    public static Integer getLastNum() {
        return LAST_NUM;
    }
}
```  

#### ExamConsumerController
`ExamConsumerController` 는 아래와 같은 엔드포인트와 그에 해당하는 역할을 갖는다.

- `/str` : 가장 최근에 수신한 문자열 메시지를 확인한다.
- `/num` : 가장 최근에 수신한 숫자형 메시지를 확인한다.

```java
@RestController
@RequiredArgsConstructor
public class ExamConsumerController {
    private final ExamConsumer examConsumer;

    @GetMapping("/str")
    public String getStr() {
        return ExamConsumer.getLastStr();
    }

    @GetMapping("/num")
    public Integer getNum() {
        return ExamConsumer.getLastNum();
    }
}
```  

#### application.yaml
`Consumer` 웹애플리케이션은 `8082` 포트를 사용한다는 것을 빼곤 모두 `Spring Cloud Stream` 에 대한 설정이다.  

`Consumer` 는 빈으로 등록된 구현체를 사용하기 때문에 `spring.cloud.stream.definition` 에 컨슈머 빈 이름이 설정 된 것을 확인 할 수 있다. 
그리고 컨슈머 빈 이름을 바인딩 해줘야 하는데 이때는 아래와 같은 규칙을 따라줘야 한다. 

```
<Bean Name>-<in(Consumer)/out(Producer)>-<index>
```  

위 규칙을 만들어진 바인딩 이름이 바로 `strConsumer-in-0` 과 `numConsumer-in-0` 이다. 
여기서 `<index>` 는 다수의 토픽과 바인딩이 필요할 때 별도의 값을 지정해줘야 하고 일반적으로는 0을 지정해주면 된다. 
그리고 최종적으로 바인딩 이름의 `destination` 설정을 사용해서 토픽이름을 매핑 해주면 된다. 

빈이름|바인딩이름|토픽이름
---|---|---
strConsumer|strConsumer-in-0|str-topic
numConsumer|numConsumer-in-0|num-topic

`Consumer` 뿐만아니라, `Producer` 도 위 처럼 빈으로 등록하는 방법이 가능 하기 때문에 관련 설정 방법과 관계는 인지할 필요가 있다.  

```yaml
server:
  port: 8082

spring:
  cloud:
    function:
      # consumer bean 이름
      definition: strConsumer;numConsumer
    stream:
      bindings:
        # consumer bean 이름을 기반으로 생성할 바인딩 이름 <빈이름>-<in/out>-<index>
        strConsumer-in-0:
          destination: str-topic
        numConsumer-in-0:
          destination: num-topic
      kafka:
        binder:
          brokers: localhost:9092
```  

#### Consumer Test
`Consumer` `Unit Test` 는 `InputDestionation` 설정된 `destination` 에 원하는 메시지를 방출하는 방법으로, 
아래와 같이 진행할 수 있다.  

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class ExamConsumerTest {
    @Autowired
    private InputDestination inputDestination;

    @Test
    public void strConsumer() {
        // when
        this.inputDestination.send(new GenericMessage<>("hello!"), "str-topic");

        // then
        assertThat(ExamConsumer.getLastStr(), is("hello!"));
    }

    @Test
    public void numConsumer() {
        // when
        this.inputDestination.send(new GenericMessage<>(1234), "num-topic");

        // then
        assertThat(ExamConsumer.getLastNum(), is(1234));
    }
}
```  

### Kafka 구성
`Kafka` 는 `docker-compose` 를 사용해서 구성하는데, 필요한 `docker-compose.yaml` 내용은 아래와 같다.  

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

`Bash` 에서 아래 명령을 사용해서 `Kafka` 를 실행 할 수 있다.  

```bash
$ docker-compose up --build
Creating myKafka     ... done
Creating myZookeeper ... done
Attaching to myZookeeper, myKafka                                     
myZookeeper  | ZooKeeper JMX enabled by default                       
myZookeeper  | Using config: /opt/zookeeper-3.4.13/bin/../conf/zoo.cfg
myKafka      | [Configuring] 'port' in '/opt/kafka/config/server.properties'
myKafka      | [Configuring] 'advertised.host.name' in '/opt/kafka/config/server.properties'

...
```  

### 통합 테스트
테스트는 `Kafka` 를 실행하고 `Producer`, `Consumer` 를 모두 실행한 상태에서 진행한다.  

먼저 `/str`, `/num` 요청을 통해 현재 `Producer`, `Consumer` 에 마지막으로 설정된 값을 확인하면 아래와 같다. 
참고로 `Producer` 는 `8081` 포트, `Consumer` 는 `8082` 포트를 사용한다.  

```bash
.. producer ..
$ curl 172.28.96.1:8081/str
none
$ curl 172.28.96.1:8081/num
-1

.. consumer ..
$ curl 172.28.96.1:8082/str
none
$ curl 172.28.96.1:8082/num
-1
```  

`Producer` 의 `/str/{str}`, `/num/{num}` 을 사용해서 각각 메시지를 방출하고 확인하면 아래와 같다.  

```bash
$ curl 172.28.96.1:8081/num/1234
done
$ curl 172.28.96.1:8081/num
1234
$ curl 172.28.96.1:8082/num
1234

$ curl 172.28.96.1:8081/str/hello
done
$ curl 172.28.96.1:8081/str
hello
$ curl 172.28.96.1:8082/str
hello
```  



---  
## Reference
[Spring Cloud Stream](https://spring.io/projects/spring-cloud-stream)  
[Spring Cloud Stream With Kafka](https://refactorfirst.com/spring-cloud-stream-with-kafka-communication.html)  
[Streaming with Spring Cloud](https://medium.com/walmartglobaltech/streaming-with-spring-cloud-24a001ad307a)  
[Unit Testing a Spring Cloud Stream Producer / Publisher](https://medium.com/@sumant.rana/unit-testing-a-spring-cloud-stream-producer-publisher-ecf39d29ea13)  
