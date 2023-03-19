--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Integration Application 구현하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Integration 과 구성요소에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Integration
    - pipe-and-filters
    - Message Endpoint
    - Filter
    - Message Gateway
    - Transformer
    - Splitter
    - Service Activator
    - Router
toc: true
use_math: true
---  

## Spring Integration Application
[이전 포스트]({{site.baseurl}}{% link _posts/spring/2023-03-11-spring-practice-spring-integration.md %})
에서는 `Spring Integration` 에 대한 개념과 주요 컴포넌트들에 대해 알아보았다.
이번 포스트에서는 외부 시스템은 연동하지 않은체 어떠한 방식으로 `Spring Integration` 을 사용해서 
메시징을 처리하는지에 대해 간단한 구현과 함꼐 알아본다.  

### Message 와 Message Channel
`Spring Integration` 의 `Message` 는 `Header` 와 `Payload` 로 구성된다. 
사용자는 구현에 따라 필요한 곳에 원하는 데이터를 넣고 아래와 같이 `MessageBuilder` 를 사용해서 메시지를 생성할 수 있다.  

```java
Message message = MessageBuilder
        .withPayload("myStringPayload")
        .setHeader("myHeader", "myHeaderValue")
        .build();
```  

생성한 `Message` 는 `MessageChannel` 를 통해 `Message Endpoint` 혹은 외부 시스템으로 전달 될 수 있다. 
생성 가능한 `Message Channel` 은 `PublishSubscribeChannel`, `QueueChannel`, `DirectChannel` 등이 있는데, 
더 자세한 목록은 [여기](https://docs.spring.io/spring-integration/docs/current/reference/html/channel.html#channel)
에서 확인 가능하다.  

```java
@Bean
public MessageChannel exampleChannel() {
    return new DirectChannel();
}
```  

### Message Endpoint
`Spring Integration` 에서 대부분의 코드는 `Message Endpoint` 에서 구현된다. 
본 포스트에서는 주요 예제 진행에 사용한 주요 `Message Endpoint` 에서만 구현 방법과 기본 개념에 대해서 설명한다. 
보다 자세한 개념적인 설명은 [여기](https://windowforsun.github.io/blog/spring/spring-practice-spring-integration/#message-endpoint)
에서 참고가능하다.  

`Message Endpoint` 에서 전달되는 메시지는 `payload` 자체만도 받거나 전달 할수 있고, `header` 를 포함한 `Message` 전체를 받거나 전달 할 수 있다. 
그 차이는 메서드 파라미터 혹은 리턴 값으로 정해 진다.  

```java
// payload 만 받고 전달
public String someMessageEndpoint(String payload) {
    return payload;
}

// header, payload 를 모두 포함한 Message 객체를 받고 전달
public Message<String> someMessageEndpoint(Message<String> message) {
    return message;
}

```  

#### Messaging Gateway
메시징 시스템의 진입점으로 외부 시스템으로 부터 메시지 `API` 를 숨겨 디커플링을 유지할 수 있다. 
요청과 응답 체널을 포함하고 양뱡향 통신으 제공한다. 
즉 `Spring Integration` 애플리케이션에서 외부에서 내부로 메시지 진입과 내부에서 외부로 메시지의 진출은 모두 `Messaging Gateway` 를 통해 수행된다고 할 수 있다. 
사용 가능한 `Messaging Gateway` 의 종류는 [여기](https://docs.spring.io/spring-integration/reference/html/endpoint-summary.html#endpoint-summary)
에서 확인 가능하다.  

```java
@MessagingGateway(name= "examGateway", defaultRequestChannel = "examChannel", errorChannel = "examErrorChannel")
public interface ExamGateway {
    @Gateway(requestChannel = "messageChannel")
    void processMessage(Message<List<String>> message);
    
    @Gateway(requestChannel = "payloadChannel")
    void processPayload(Message<List<String>> message);
    
    @Gateway
    void process(Message<List<String>> message);
}
```  

만약 `Message Endpoint` 에서 `outputChannel` 를 설정하지 않으면 `Spring Integration` 에서 
기본으로 제공하는 `nullChannel` 로 전달되며 해당 메시지는 버려진다. 
`nullChannel` 를 비롯한 기본 제공되는 채널은 [여기](https://docs.spring.io/spring-integration/docs/current/reference/html/channel.html#channel-special-channels)
에서 확인 가능하다.  

#### Transformer
메시지의 내용 또는 구조를 변환하고, 수정된 메시지를 반환한다. 
`inputChannel` 로 들어온 메시지를 변환 결과를 `outputChannel` 로 전달 한다. 

```java
@Transformer(inputChannel = "examInputChannel", outputChannel = "examOutputChannel")
public String someTransformer(String payload) {
    // some process ..
        
    return payload;    
}
``` 

#### Filter
어떤 메시지를 출력 체널로 전달 할지 결정한다. 
`inputChannel` 로 들어온 메시지에 대해서 검사 후 `true` 를 리턴하면 `outputChannel` 로 전달하고, 
`false` 를 리턴하면 `discardChannel` 로 전달한다.  

```java
@Filter(inputChannel = "examInputChannel", outputChannel = "examOutputChannel", discardChannel = "examDiscardChannel")
public boolean someFilter(String payload) {
    boolean result = check(payload);
    
    return result;
}
```  

#### Splitter
입력 채널로부터 메시지를 받아 메시지를 여러 개로 분할해 출력 채널로 전달한다. 
`inputChannel` 로 부터 전달 받은 `Message<List<String>>` 형태의 혹합 페이로드를 `List<String>` 형태로 리턴하면, 
`List` 의 각 원소 값인 `String` 이 개별로 `outputChannel` 로 전달 된다.  

```java
@Splitter(inputChannel = "examInputChannel", outputChannel = "examOutputChannel")
public List<String> someSplitter(Message<List<String>> message) {
    return message.getPayload();
}
```  

#### Service Activator
서비스를 메시징 시스템에 연결하기 위한 엔드포인트다. 
`inputChannel` 로 부터 전달된 메시지를 원하는 `outputChannel` 로 전달하거나, 
`outputChannel` 를 정의하지 않으면 해당 메지는 버려진다.  

```java
@ServiceActivator(inputChannel = "examInputChannel", outputChannel = "examOutputChannel")
public Message<String> someServiceActivator(Message<String> message) {
    return message;
}
```  

#### Router
메시지를 보고 조건에 따라 필요한 채널로 해당 메시지를 전달 할 수 있다.  
`inputChannel` 로 부터 전달된 메시지를 사용자가 원하는 검사로직을 통해 알맞는 채널 이름을 리턴하면, 
해당 메시지는 리턴된 이름의 `outputChannel` 로 전달 된다.  

```java
@Router(inputChannel = "examInputChannel")
public String someRouter(String payload) {
    if(someCheck1(payload)) {
        return "examOutputChannel1";    
    } else if(someCheck2(payload)){
        return "examOutputChannel2";
    } else {
        return "defaultOutputChannel";
    }
}
```  

### 예제 애플리케이션

![그림 1]({{site.baseurl}}/img/spring/spring-integration-basic-application-1.drawio.png)  

위 그림은 예제 애플리케이션에서 `Spring Integration` 을 사용해 구현할 메시지 흐름을 도식화 한 것이다. 
메시지의 `payload` 는 `String` 타입의 문자열로만 구성되고, 
사용자 정의 헤더도 설정해서 사용한다.

구현에 필요한 의존성 정보를 담고 있는 `build.gradle` 내용은 아래와 같다.  

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.6.4'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'

version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-integration';
    implementation 'org.springframework.boot:spring-boot-starter-test';
    testImplementation 'org.springframework.integration:spring-integration-test';
    compileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
}

test {
    useJUnitPlatform()
}
```  

구현하는 애플리케이션의 시작점인 `ExamApplication` 의 내용은 아래와 같다. 
애플리케이션이 실행되면 `CommandLineRunner` 를 통해 
문자와 숫자로 구성된 문자열 리스트를 `Messaging Gateway` 인 `ExamGateway` 로 전달 한다.  

```java
@EnableIntegration
@SpringBootApplication
public class ExamApplication {
    public static void main(String... args) {
        SpringApplication.run(ExamApplication.class, args);
    }

    @Autowired
    private ExamGateway examGateway;

    @Bean
    public CommandLineRunner runner() {
        return (args) ->
            this.examGateway.process(MessageBuilder
                    .withPayload(List.of("a", "b", "1", "2", "c", "d", "3", "4", "e", "f", "5", "6", "g", "h", "7", "8"))
                    .build());
    }
}
```  

애플리케이션에서 사용하는 모든 채널 설정을 담고 있는 `ChannelConfig` 의 구현 내용은 아래와 같다. 
사용하는 모든 채널은 `DirectChannel` 을 사용하고 채널 이름을 빈이름(메서드 이름)으로 작성해 빈으로 등록한다.  

```java
@Configuration
public class ChannelConfig {
    @Bean
    public MessageChannel entrypointChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel headerFilterChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel routerChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel stringFilterChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel numberFilterChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel stringTransformerChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel numberTransformerChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel discardChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel aggregatorChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageChannel resultChannel() {
        return new DirectChannel();
    }
}
```  

메시징 시스템의 시작점인 `ExamGateway`(`Messaging Gateway`) 의 구현 내용은 아래와 같다. 
`Message<List<String>>` 타입을 `process` 라는 메서드로 받아 요청 채널인 `entrypointChannel` 로 전달 한다.  

```java
@Component
@MessagingGateway(name = "examGateway", defaultRequestChannel = "entrypointChannel", errorChannel = "nullChannel")
public interface ExamGateway {
    @Gateway(requestChannel = "entrypointChannel")
    void process(Message<List<String>> message);
}
```  

애플리케이션에서 공통 `Message Endpoint` 구현 내용이 있는 `CommonEndpoint` 의 내용은 아래와 같다. 

```java
@Slf4j
@MessageEndpoint
public class CommonEndpoint {
    private static final AtomicInteger MESSAGE_INDEX = new AtomicInteger();
    
    // entrypointChannel 로 부터 전달된 List<String> 형태의 payload 를
    // 각 원소 String 으로 분리해 headerFilterChannel 로 전달한다. 
    @Splitter(inputChannel = "entrypointChannel", outputChannel = "headerFilterChannel")
    public List<Message<String>> splitList(Message<List<String>> message) {
        List<Message<String>> messages = new ArrayList<>();
        log.info("entrypointChannel splitList {}", message.getPayload());

        for (String str : message.getPayload()) {
            messages.add(MessageBuilder
                    .withPayload(str)
                    .setHeader("messageIndex", MESSAGE_INDEX.getAndIncrement())
                    .build());
        }

        return messages;
    }

    // headerFilterChannel 로 전달 된 메시지의 헤더 값중 messageIndex 값이 짝수 인 것만
    // routerChannel 로 전달하고 홀수 인 메시지는 discardChannel 로 전달해 이후 처리 플로우에서 제외한다. 
    @Filter(inputChannel = "headerFilterChannel", outputChannel = "routerChannel", discardChannel = "discardChannel")
    public boolean headerFilter(Message<String> message) {
        String messageIndex = message.getHeaders().getOrDefault("messageIndex", "").toString();
        log.info("headerFilterChannel headerFilter payload : {}, messageIndex : {}", message.getPayload(), messageIndex);

        return !Objects.equals(messageIndex, "") && Integer.parseInt(messageIndex) % 2 == 0;
    }

    // routerChannel 로 전달된 메시지의 타입을 보고 
    // 문자 타입이라면 stringFilterChannel 로 전달하고
    // 숫자 타입이라면 numberFilterChannel 로 전달한다. 
    @Router(inputChannel = "routerChannel")
    public String route(String payload) {
        log.info("routerChannel route : {}", payload);

        try {
            Integer.parseInt(payload);
            return "numberFilterChannel";
        } catch (NumberFormatException ignore) {
            return "stringFilterChannel";
        }
    }

    // discardChannel 로 전달된 애플리케이션에서 filter 를 통해 처리에서 제외된 메시지 로깅을 수행한다. 
    @ServiceActivator(inputChannel = "discardChannel")
    public void loggingDiscardMessage(Message<String> message) {
        log.info("discardChannel loggingDiscardMessage payload : {}, messageIndex : {}", message.getPayload(), message.getHeaders().get("messageIndex"));
    }

    // resultChannel 로 전달된 메시지 흐름의 최종 endpoint 로 결과 로딩을 수행한다. 
    @ServiceActivator(inputChannel = "resultChannel")
    public void loggingResult(Message<String> message) {
        log.info("resultChannel loggingResult payload : {}, messageIndex : {}", message.getPayload(), message.getHeaders().get("messageIndex"));
    }
}
```  

`router` 를 통해 라우팅되는 채널 중 문자열 메시지에 대한 `Message Endpoint` 내용이 있는 `StringMessageEndpoint` 는 아래와 같다.  

```java
@Slf4j
@MessageEndpoint
public class StringMessageEndpoint {
    
    // stringFilterChannel 로 전달되는 문자열 메시지 중 
    // c 보다 큰 문자열만 stringTransformerChannel 로 전달하고
    // 그외 메시지는 discardChannel 로 전달해 메시지 처리 흐름에서 제외한다. 
    @Filter(inputChannel = "stringFilterChannel", outputChannel = "stringTransformerChannel", discardChannel = "discardChannel")
    public boolean stringFilter(String payload) {
        log.info("stringFilterChannel stringFilter : {}", payload);

        return payload.compareTo("c") > 0;
    }

    // stringTransformerChannel 로 전달된 문자열 메시지를 
    // 대문자로 변경해 resultChannel 로 전달 한다. 
    @Transformer(inputChannel = "stringTransformerChannel", outputChannel = "resultChannel")
    public String uppercase(String payload) {
        log.info("stringTransformerChannel uppercase : {}", payload);

        return payload.toUpperCase();
    }

}
```  

`router` 를 통해 라우팅되는 채널 중 숫자 타입 메시지에 대한 `Message Endpint` 내용이 있는 `NumberMessageEndpoint` 의 내용은 아래와 같다.  

```java
@Slf4j
@MessageEndpoint
public class NumberMessageEndpoint {
    
    // numberFilterChannel 로 전달되는 숫자 타입 메시지 중
    // 3보다 큰 메시지만 numberTransformerChannel 로 전달하고 
    // 3보다 작은 메시지는 discardChannel 로 보내 메시지 처리 흐름에서 제외한다. 
    @Filter(inputChannel = "numberFilterChannel", outputChannel = "numberTransformerChannel", discardChannel = "discardChannel")
    public boolean numberFilter(String payload) {
        log.info("numberFilterChannel numberFilter : {}", payload);
        int number = Integer.parseInt(payload);

        return number > 3;
    }

    // numberTransformerChannel 로 전달되는 숫자 타입 메시지의
    // 제곱연산을 수행한 결과를 resultChannel 로 전달 한다. 
    @Transformer(inputChannel = "numberTransformerChannel", outputChannel = "resultChannel")
    public String square(String payload) {
        log.info("numberTransformerChannel square : {}", payload);
        int number = Integer.parseInt(payload);

        return String.valueOf(number * number);
    }

}
```  

구현된 애플리케이션을 실행하면 아래와 같은 로그를 확인 할 수 있다.  

```
entrypointChannel splitList [a, b, 1, 2, c, d, 3, 4, e, f, 5, 6, g, h, 7, 8]
headerFilterChannel headerFilter payload : a, messageIndex : 0
routerChannel route : a
stringFilterChannel stringFilter : a
discardChannel loggingDiscardMessage payload : a, messageIndex : 0
headerFilterChannel headerFilter payload : b, messageIndex : 1
discardChannel loggingDiscardMessage payload : b, messageIndex : 1
headerFilterChannel headerFilter payload : 1, messageIndex : 2
routerChannel route : 1
numberFilterChannel numberFilter : 1
discardChannel loggingDiscardMessage payload : 1, messageIndex : 2
headerFilterChannel headerFilter payload : 2, messageIndex : 3
discardChannel loggingDiscardMessage payload : 2, messageIndex : 3
headerFilterChannel headerFilter payload : c, messageIndex : 4
routerChannel route : c
stringFilterChannel stringFilter : c
discardChannel loggingDiscardMessage payload : c, messageIndex : 4
headerFilterChannel headerFilter payload : d, messageIndex : 5
discardChannel loggingDiscardMessage payload : d, messageIndex : 5
headerFilterChannel headerFilter payload : 3, messageIndex : 6
routerChannel route : 3
numberFilterChannel numberFilter : 3
discardChannel loggingDiscardMessage payload : 3, messageIndex : 6
headerFilterChannel headerFilter payload : 4, messageIndex : 7
discardChannel loggingDiscardMessage payload : 4, messageIndex : 7
headerFilterChannel headerFilter payload : e, messageIndex : 8
routerChannel route : e
stringFilterChannel stringFilter : e
stringTransformerChannel uppercase : e
resultChannel loggingResult payload : E, messageIndex : 8
headerFilterChannel headerFilter payload : f, messageIndex : 9
discardChannel loggingDiscardMessage payload : f, messageIndex : 9
headerFilterChannel headerFilter payload : 5, messageIndex : 10
routerChannel route : 5
numberFilterChannel numberFilter : 5
numberTransformerChannel square : 5
resultChannel loggingResult payload : 25, messageIndex : 10
headerFilterChannel headerFilter payload : 6, messageIndex : 11
discardChannel loggingDiscardMessage payload : 6, messageIndex : 11
headerFilterChannel headerFilter payload : g, messageIndex : 12
routerChannel route : g
stringFilterChannel stringFilter : g
stringTransformerChannel uppercase : g
resultChannel loggingResult payload : G, messageIndex : 12
headerFilterChannel headerFilter payload : h, messageIndex : 13
discardChannel loggingDiscardMessage payload : h, messageIndex : 13
headerFilterChannel headerFilter payload : 7, messageIndex : 14
routerChannel route : 7
numberFilterChannel numberFilter : 7
numberTransformerChannel square : 7
resultChannel loggingResult payload : 49, messageIndex : 14
headerFilterChannel headerFilter payload : 8, messageIndex : 15
discardChannel loggingDiscardMessage payload : 8, messageIndex : 15
```


---  
## Reference
[Overview of Spring Integration Framework](https://docs.spring.io/spring-integration/docs/current/reference/html/overview.html)  
[Integration Endpoints](https://docs.spring.io/spring-integration/reference/html/endpoint-summary.html)  
[Message Processing with Spring Integration](https://www.javacodegeeks.com/2014/12/message-processing-with-spring-integration.html)  
[harikrishna553/springboot](https://github.com/harikrishna553/springboot/tree/master/spring-integration)  
[Message Channels](https://docs.spring.io/spring-integration/docs/current/reference/html/channel.html#channel)  
