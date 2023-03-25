--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Integration Java DSL"
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
    - Spring Integration
    - pipe-and-filters
    - Message Endpoint
    - Filter
    - Message Gateway
    - Transformer
    - Splitter
    - Service Activator
    - Router
    - Java DSL
toc: true
use_math: true
---  

## Spring Integration Java DSL
[이전 포스트 1](),
[이전 포스트 2]()
에서는 `Spring Integration` 에 대한 기본적인 개념과 `Java Config` 과 `Annotation` 을 통해 
`Spring Integration` 의 기본적인 사용 방법에 대해 알아보았다.  

`Java DSL` 을 사용하면 `@Configuration` 클래스에서 `Builder` 와 `Fluent API` 를 사용해서 
간편하게 `Spring Integration` 메시지 플로우 작성이 가능하다. 
이번 방법은 각 메소드 혹은 클래스를 생성해서 `Spring Integration` 의 엔드포인트인 `Filter`, `Transformer` 등을 구현 해서 플로우를 구성했다. 
`Java DSL` 을 사용하면 보다 간편하면서도 기존 `Spring Framework` 코드 설정에 쉽고 간결하게 추가할 수 있다.  

`Java DSL` 은 `IntegrationFlowBuilder` 의 팩토리 클래스인 `IntegrationFlows` 로 사용해서 작성 가능하다. 
최종적으로 생성된 `IntegrationFlow` 를 `@Bean` 을 통해 스프링 빈으로 등록해주면 된다. 
빌더 방식으로 `IntegrationFlow` 인 메시지 플로우를 생성하기 때문에, 
복잡성을 줄이고 람다를 통해 계층 구조로 표현이 가능하다.  

아래는 `Java DSL` 을 사용해서 간단한 메시지 플로우를 구성한 예이다. 

```java
@Configuration
public class FirstJavaDslConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow firstFlow() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .filter((Integer p) -> p % 2 == 0)
                .channel("inputChannel")
                .transform(Object::toString)
                .transform("Hello "::concat)
                .channel(MessageChannels.queue("myQueue"))
                .log("firstFlow")
                .get();
    }
}
```  

정의한 플로우는 아래와 같다. 

1. `fromSupplier` 에서 `Supplier` 는 `Integer` 값을 0 부터 증가해 반환한다. 
2. `fromSupplier` 에서 `Consumer` 의 `poller` 를 사용해서 `Supplier` 의 값을 1초마다 소비한다. 
3. `filter` 에서 짝수인 값만 필터를 통과 시킨다. 
4. 짝수인 숫자들만 `inputChannel` 로 전달한다. 
5. `transform` 은 `inputChannel` 로 전달된 짝수 값에 대해서 문자열로 형변환 한다. 
6. 앞선 `tranform` 을 통해 문자열 값에 `Hello` 문자열을 더한다. 
7. `Hello` 값까지 더하진 문자열을 `myQueue` 라는 `Queue` 채널에 넣는다. 
8. `log` 가 선언된 시점까지의 메시지를 로깅한다. 

최종적으로 로깅된 메시지는 아래와 같다. 

```
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 0, headers={id=df5e91f4-9c1c-e13a-d759-198be47376d5, timestamp=1679740468138}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 2, headers={id=74ff43cf-eab7-e454-2cc1-f45cc9ac504d, timestamp=1679740470139}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 4, headers={id=04a9121d-1522-0e33-0323-83ad0ce55c7b, timestamp=1679740472136}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 6, headers={id=c85f3280-625d-47da-be74-0dc0f8bc8ce5, timestamp=1679740474136}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 8, headers={id=38a8f5db-5787-4345-fdc5-f8cb1a825ef4, timestamp=1679740476137}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 10, headers={id=0f172df4-b928-fcc1-a250-012141ff6230, timestamp=1679740478134}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 12, headers={id=44e1cbe5-8c85-22e9-cd88-5910774cd72c, timestamp=1679740480139}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 14, headers={id=116cd051-15ed-8494-df2c-ad9b0c9f528c, timestamp=1679740482137}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 16, headers={id=eedc7053-da09-6879-7a55-082b8a1988cc, timestamp=1679740484136}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 18, headers={id=95a98b9a-9f31-d44b-7e85-a2cac19b1cc6, timestamp=1679740486137}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 20, headers={id=00285c43-6ab5-d68f-0dba-8db1bc92900d, timestamp=1679740488139}]
INFO 32880 --- [   scheduling-1] firstFlow                                : GenericMessage [payload=Hello 22, headers={id=651e7344-7f31-b77d-867a-4912c8c83da2, timestamp=1679740490137}]
```  

### DSL Method
`Java DSL` 은 `Spring Integration` 에서 제공하는 모든 엔드포인트를 메서드 단위로 제공한다. 
제공하는 종류와 `Spring Integration` 의 엔드포인트 이름과 매칭하면 아래와 같다.  

DSL Method|Endpoint
---|---
transform|Transformer
filter|Filter
handle|ServiceActivator
split|Splitter
aggregator|Aggregator
route|Router
bridge|Bridge

사용자는 `Spring Integration` 의 엔드포인트에 해당하는 동작을 `DSL` 메서드를 사용해서, 
아래와 같이 간결하고 쉽게 메시지 플로우 구현이 가능하다.  

```java
@Bean
public IntegrationFlow myFlow() {
    return IntegrationFlows.from("myChannel")
    .filter((Integer i) -> i % 2 == 0)
    .transform(Object::toString)
    .transform("Hello"::concat)
    .handle(System.out::println)
    .get();
}
```  

위 메시지 플로우는 아래와 같은 흐름으로 수행된다. 

`Filter -> Transformer -> Transformer -> Service Activator`

### Message Channel
`Java DSL` 은 엔드포인트에 대한 메서드 뿐만아니라, 
`MessageChannel` 인스턴스를 설정 할 수 있는 `Fluent API` 도 제공한다. 
`Java Config` 를 통해 엔드포인트에 `input/output` 채널을 설정했던 것과 동일하게, 
`Java DSL` 에서는 `IntegrationFlowBuilder` 의 `channel()` 메서드를 사용해서 엔드포인트와 채널을 연결 할 수 있다.  

엔드포인트는 기본적으로 `DirectChannel` 로 연결되고, 이 떄 채널의 빈 이름은 아래와 같은 규칙을 갖는다. 

```
[IntegrationFlow.beanName].channel#[channelNameIndex]

e.g.) firstFlow.channel#0, firstFlow.channel#1
```

실제 생성된 로그를 확인하면 아래와 같다. 

```
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {filter} as a subscriber to the 'firstFlow.channel#0' channel
INFO 32880 --- [           main] o.s.integration.channel.DirectChannel    : Channel 'application.firstFlow.channel#0' has 1 subscriber(s).
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean 'firstFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#0'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/FirstJavaDslConfig.class]'; from source: 'bean method firstFlow'
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {transformer} as a subscriber to the 'inputChannel' channel
INFO 32880 --- [           main] o.s.integration.channel.DirectChannel    : Channel 'application.inputChannel' has 1 subscriber(s).
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean 'firstFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#1'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/FirstJavaDslConfig.class]'; from source: 'bean method firstFlow'
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {transformer} as a subscriber to the 'firstFlow.channel#1' channel
INFO 32880 --- [           main] o.s.integration.channel.DirectChannel    : Channel 'application.firstFlow.channel#1' has 1 subscriber(s).
INFO 32880 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean 'firstFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#2'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/FirstJavaDslConfig.class]'; from source: 'bean method firstFlow'
INFO 32880 --- [           main] o.s.i.e.SourcePollingChannelAdapter      : started bean 'firstFlow.org.springframework.integration.config.SourcePollingChannelAdapterFactoryBean#0'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/FirstJavaDslConfig.class]'; from source: 'bean method firstFlow'
```  

`channel()` 메서드는 빈이름을 문자열로 작성 할 수도 있고, 
`MessageChannel` 인스턴스 자체를 인자로 전달 할 수도 있다. 
문자열로 채널 이름을 작성할 경우 존재하지 않는 경우 `DirectChannel` 로 새로 생성한다.  

`channel()` 메서드 예제는 아래와 같다.  

```java

```  






---  
## Reference
[Spring Integration Java DSL](https://docs.spring.io/spring-integration/reference/html/dsl.html#java-dsl)  
[Spring Integration Java DSL sample](https://dzone.com/articles/spring-integration-java-dsl)  
