--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Integration Java DSL"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Fluent API 를 사용해서 Spring Integration 을 좀 더 효율적이고 간결하게 작성할 수 있는 Java DSL 에 대해 알아보자'
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
[이전 포스트 1]({{site.baseurl}}{% link _posts/spring/2023-03-11-spring-practice-spring-integration.md %}),
[이전 포스트 2]({{site.baseurl}}{% link _posts/spring/2023-03-18-spring-practice-spring-integration-basic-application.md %})
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
`Java DSL` 은 `Spring Integration` 에서 제공하는 것들을
`Fluent API` 방식으로 메시지 기반 애플리케이션에 필요한 `EIP`(Enterprise Integration Patterns) 를 제공한다. 


제공하는 종류와 `Spring Integration` 엔드포인트 이름과 `EIP` 를 매칭하면 아래와 같다.  

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
@Configuration
public class MessageChannelConfig {
    @Bean
    public MessageChannel queueChannel() {
        return MessageChannels.queue().get();
    }

    @Bean
    public MessageChannel pubSubChannel() {
        return MessageChannels.publishSubscribe().get();
    }

    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow channelFlow() {
        return IntegrationFlows.from("input")
                .fixedSubscriberChannel()
                .channel("queueChannel")
                .channel(pubSubChannel())
                .channel(MessageChannels.executor("executorChannel", Executors.newWorkStealingPool()))
                .channel("output")
                .get();
    }
}
```  

아래는 위와 같은 채널 설정을 실행 했을 떄 실제로 어떠한 채널들이 생성되는지 확인 하기 위한 실행 로그 일부이다. 
총 7개의 채널이 생성되고 그 종류와 설명은 아래와 같다. 

```
// errorChannel 
o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
o.s.i.endpoint.EventDrivenConsumer       : started bean '_org.springframework.integration.errorLogger'

// inputChannel, channel("input"), from("input") 
o.s.i.endpoint.EventDrivenConsumer       : Adding {bridge} as a subscriber to the 'input' channel
o.s.integration.channel.DirectChannel    : Channel 'application.input' has 1 subscriber(s).
o.s.i.endpoint.EventDrivenConsumer       : started bean 'channelFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#0'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class]'; from source: 'bean method channelFlow'

// channelFlow.channel#0, fixedSubscirberChannel() 
o.s.i.endpoint.EventDrivenConsumer       : Adding {bridge} as a subscriber to the 'channelFlow.channel#0' channel
o.s.i.endpoint.EventDrivenConsumer       : started bean 'channelFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#1'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class]'; from source: 'bean method channelFlow'

// pubSubChannel, channel(publishSubscirbe())  
o.s.i.endpoint.EventDrivenConsumer       : Adding {bridge} as a subscriber to the 'pubSubChannel' channel
o.s.i.channel.PublishSubscribeChannel    : Channel 'application.pubSubChannel' has 1 subscriber(s).
o.s.i.endpoint.EventDrivenConsumer       : started bean 'channelFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#3'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class]'; from source: 'bean method channelFlow'


// executorChannel, channel(MessageChannels.executor("executorChannel", Executors.newWorkStealingPool()) 
o.s.i.endpoint.EventDrivenConsumer       : Adding {bridge} as a subscriber to the 'executorChannel' channel
o.s.integration.channel.ExecutorChannel  : Channel 'application.executorChannel' has 1 subscriber(s).
o.s.i.endpoint.EventDrivenConsumer       : started bean 'channelFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#4'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class]'; from source: 'bean method channelFlow'

// outputChannel, channel("out")
o.s.i.endpoint.EventDrivenConsumer       : Adding {bridge} as a subscriber to the 'output' channel
o.s.integration.channel.DirectChannel    : Channel 'application.output' has 1 subscriber(s).
o.s.i.endpoint.EventDrivenConsumer       : started bean 'outputFlow.org.springframework.integration.config.ConsumerEndpointFactoryBean#0'; defined in: 'class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class]'; from source: 'bean method outputFlow'


```  

- `eorrChannel` : `DirectChannel` 로 메시지 플로우에서 발생한 에러 메시지가 전달된다.
- `channel("input")` : `input` id를 가진 메시지 채널을 찾아 사용하거나 `DirectChannel` 로 새로 생성한다.
- `fixedSubscriberChannel()` : `FixedSubscriberChannel` 인스턴스를 생성해 `channelFlow.channel#0` 이름으로 등록한다.
- `channel("queueChannel")` : 미리 빈으로 생성해 둔 `queueChannel` 를 사용한다. 
- `channel(pubSubChannel())` : 미리 빈으로 생성해 둔 `pubSubChannel` 를 사용한다.
- `channel(MessageChannels.executor("executorChannel", Executors.newWorkStealingPool())` : `ExecutorChannel` 인스턴스를 통해 `IntegrationComponentSpec` 을 정의하고 새로 생성해서 채널을 등록한다. 
- `channel("out")` : `out` id를 가진 메시지 채널을 찾아 사용하거나 `DirectChannel` 를 새로 생성한다.


`MessageChannel` 를 생성하거나 메시지 플로우에서 사용시 주의점이 있는데, 
`MessageChannels` 팩토리를 사용해서 동일한 인라인 채널을 정의하지 않도록 주의해야 한다. 
이름에 해당하는 채널이 존재하지 않다면 이름에 해당하는 채널을 생성해서 빈으로 등록해 주지만, 
동일한 이름의 `MessageChannel` 객체가 존재하는 지는 판별 불가하다. 
그 예시는 아래와 같다.  

```java
@Bean
public IntegrationFlow startFlow() {
    return IntegrationFlows.from("input")
            .channel(MessageChannels.queue("examChannel"))
            .get();
}

@Bean
public IntegrationFlow endFlow() {
    return IntegrationFlows.from(MessageChannels.queue("examChannel"))
            .log()
            .get();
}
```  

위와 같은 설정으로 애플리케이션을 실행하면 아래와 같은 에러 메시지가 출력된다. 

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'examChannel', defined in the 'endFlow' bean definition, could not be registered. A bean with that name has already been defined in class path resource [com/windowforsun/spring/integration/javadsl/MessageChannelConfig.class] and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true


Process finished with exit code 1
```  

아래와 같은 구조로 채널을 빈으로 미리 생성해 재사용하는 것이 좋다.  

```java
@Bean
public MessageChannel examChannel() {
    return MessageChannels.queue().get();
}

@Bean
public IntegrationFlow startFlow() {
    return IntegrationFlows.from("input")
            .channel("examChannel")
            .get();
}

@Bean
public IntegrationFlow endFlow() {
    return IntegrationFlows.from("examChannel")
            .log()
            .get();
}
```  

### Pollers
`Poller` 는 메시지를 주기적으로 가져오기 위한 컴포넌트이다. 
`Poller` 는 인바운드, 아웃바운드 메시지 흐름에서 사용될 수 있다. 

- `FTP`, `DB`, 웹서비스 같은 외부 시스템에 주기적으로 접속해 데이터를 가져와야 할 때
- 내부 메시지 채널로부터 주기적으로 데이터를 가저와야 할 때
- 내부 서비스를 주기적으로 호출해야 할 때

간단한 `Poller` 예시는 아래와 같다.  

```java
@Configuration
public class PollerConfig {
    private final AtomicInteger atomicInteger = new AtomicInteger();

    @Bean(name = PollerMetadata.DEFAULT_POLLER)
    public PollerSpec pollerSpec() {
        return Pollers.fixedRate(1000)
                .errorChannel("pollerErrorChannel");
    }

    @Bean
    public IntegrationFlow pollerChannel() {
        return IntegrationFlows.fromSupplier(
                        // 필요한 주기적인 동작
                        () -> {
                            int i = atomicInteger.getAndIncrement();

                            if (i % 2 == 1) {
                                throw new RuntimeException("test exception");
                            }

                            return i;
                        },
                        // poller 설정
                        spec -> spec.poller(pollerSpec())
                )
                .channel("outputChannel")
                .get();
    }

    @Bean
    public IntegrationFlow loggingOutput() {
        return IntegrationFlows.from("outputChannel")
                .log("loggingOutput")
                .get();
    }

    @Bean
    public IntegrationFlow loggingError() {
        return IntegrationFlows.from("pollerErrorChannel")
                .log("loggingError")
                .get();
    }
}
```  

1초마다 메시지가 발송되고 번갈아 가며 `loggingOutput` 과 `loggingError` 가 찍히는 것을 확인 할 수 있다.  

```
INFO 41781 --- [   scheduling-1] loggingOutput                            : GenericMessage [payload=0, headers={id=03561c39-5d84-abf9-319e-882857b1e3b5, timestamp=1679827765217}]
INFO 41781 --- [   scheduling-1] loggingError                             : ErrorMessage [payload=org.springframework.messaging.MessagingException: nested exception is java.lang.RuntimeException: test exception, headers={id=f331a604-4a70-31d9-1a33-11241de836b8, timestamp=1679827766223}]
INFO 41781 --- [   scheduling-1] loggingOutput                            : GenericMessage [payload=2, headers={id=0271a060-2eee-57bf-35ae-ba3eb13b5b71, timestamp=1679827767225}]
INFO 41781 --- [   scheduling-1] loggingError                             : ErrorMessage [payload=org.springframework.messaging.MessagingException: nested exception is java.lang.RuntimeException: test exception, headers={id=9fc394b9-06a6-3a58-9d24-7a04ceefb998, timestamp=1679827768226}]
INFO 41781 --- [   scheduling-1] loggingOutput                            : GenericMessage [payload=4, headers={id=616cff48-9224-c9c1-7f64-702c2cc72ca3, timestamp=1679827769227}]
INFO 41781 --- [   scheduling-1] loggingError                             : ErrorMessage [payload=org.springframework.messaging.MessagingException: nested exception is java.lang.RuntimeException: test exception, headers={id=a42d3ed1-5bfb-6496-c92e-b12a254d991c, timestamp=1679827770232}]
INFO 41781 --- [   scheduling-1] loggingOutput                            : GenericMessage [payload=6, headers={id=58fc0764-f75c-f3ab-21d8-d7639c95c41f, timestamp=1679827771234}]
INFO 41781 --- [   scheduling-1] loggingError                             : ErrorMessage [payload=org.springframework.messaging.MessagingException: nested exception is java.lang.RuntimeException: test exception, headers={id=158c2c44-8bb9-b5c4-1885-90d71a97c8d6, timestamp=1679827772237}]
```  
ㄷ
### reactive()
`Spring Integration 5.5` 버전 부터는 `reactor-core` 의존성을 가지고 있기 때문에 `ConsumerEndpointSpec` 에서 `reactive()` 라는 메서드를 제공한다. 
이는 채널 종류와 상관없이 타겟 엔드포인트를 `ReactiveStreamConsumer` 인스턴스로 설정해서, 
입력 채널은 `IntegrationReactiveUtils.messageChannelToFlux()` 를 통해 `Flux` 로 변환하게 된다. 
`Flux` 의 연산자를 이용해서 입력 채널 스트림 소스를 커스텀하는 것도 가능하다.  

아래는 `DirectChannel` 로 시작하는 채널을 `inputChannel` 에서 `publishingChannel` 을 변경하는 예시이다.  

```java
@Configuration
public class ReactiveConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow reactiveFlow() {
        return IntegrationFlows.from("input")
                .transform(Object::toString)
                .log("beforeReactiveFlow")
                .transform(
                        "Hello "::concat,
                        consumerEndpointSpec -> consumerEndpointSpec.reactive(flux -> flux.publishOn(Schedulers.parallel()))
                )
                .log("afterReactiveFlow")
                .get();
    }
}
```  

`flux` 로 전환 전까지는 `scheduling` 채널 이였던 반면에 `flux` 전환 후 정의한 `parallel` 스레드로 변경된 것을 확인 할 수 있다.  

```
INFO 56257 --- [   scheduling-1] beforeReactiveFlow                       : GenericMessage [payload=0, headers={id=f28c6965-ba75-b9ff-0056-149b795c071d, timestamp=1679828676888}]
INFO 56257 --- [     parallel-1] afterReactiveFlow                        : GenericMessage [payload=Hello 0, headers={id=8b485c88-1358-5e1f-1f3f-9711cf2fc60a, timestamp=1679828676889}]
INFO 56257 --- [   scheduling-1] beforeReactiveFlow                       : GenericMessage [payload=1, headers={id=25e6b924-11f4-0e79-c297-5e140a9317d2, timestamp=1679828677890}]
INFO 56257 --- [     parallel-1] afterReactiveFlow                        : GenericMessage [payload=Hello 1, headers={id=2b07a69d-a091-bb05-be0e-fb60d8d181ec, timestamp=1679828677890}]
INFO 56257 --- [   scheduling-1] beforeReactiveFlow                       : GenericMessage [payload=2, headers={id=cd179fbd-f49b-2bd2-27ae-0322f666df16, timestamp=1679828678886}]
INFO 56257 --- [     parallel-1] afterReactiveFlow                        : GenericMessage [payload=Hello 2, headers={id=cc9f617d-4dc7-2fdd-3941-8e62b5ba554d, timestamp=1679828678887}]
```  

`ReactiveStreams` 에 대한 자세한 내용은 [여기](https://godekdls.github.io/Spring%20Integration/reactive-streams/)
에서 확인 가능하다.  

### Transformers
`Transformers` 팩토리 클래스를 이용하면, `transform()` 내에서 타겟 객체를 간편하게 인라인으로 변환 할 수 있다.  

```
@Bean
public IntegrationFlow transformFlow() {
    return IntegrationFlows.from("input")
            .transform(Transformers.fromJson(MyPojo.class))
            .transform(Transformers.serializer())
            .get();
}
```  

`Transformers` 에 대한 자세한 내용은 [여기](https://docs.spring.io/spring-integration/api/org/springframework/integration/dsl/Transformers.html)
에서 확인 가능하다.  

### Inbound Channel Adapters
`Spring Intgration` 에서 메시지 플로우는 `Inbound Channel Adapter` 에서 시작한다. 
`Java DSL` 의 `IntegrationFlow` 를 사용하면, 
`from()` 메서드의 인자 중 `from(MessageSource, Consumer<SoucePollerChannelAdapterSpec>)` 을 사용해서 간단하게 주기적인 `Consumer` 동작을 설정 할 수 있다.  

```java
@Bean
public MessageSource<Object> jdbcMessageSource() {
    return new JdbcPollingChannelAdapter(this.dataSource, "SELECT * FROM something");
}

@Bean
public IntegrationFlow pollingFlow() {
    return IntegrationFlows.from(jdbcMessageSource(),
                c -> c.poller(Pollers.fixedRate(100).maxMessagesPerPoll(1)))
            .transform(Transformers.toJson())
            .channel("furtherProcessChannel")
            .get();
}
```  

만약 `Message` 타입이 아니라면 `fromSupplier()` 를 사용하여 `Message` 타입이 아니라도 자동으로 `Message` 로 감싸준다.  


### Routers
`Spring Integration` 은 아래와 같은 종류의 리우터를 기본으로 제공하는데, 

- `HeaderValueRouter`
- `PayloadTypeRouter`
- `ExceptionTypeRouter`
- `RecipientListRouter`
- `XPathRouter`

여타 다른 메시지 엔드포이트들과 비슷하게 커스텀한 `routing` 동작도 정의가 가능하며, 
`SpEL` 포현식, `ref-method` 쌍 등 다양한 방식으로 원하는 라우팅 동작 정의가 가능하다.  

```java
@Configuration
public class RouterConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow routeByLambda() {
        return IntegrationFlows.from("input")
                .<Integer, Boolean>route(
                        i -> i % 2 == 0,
                        m -> m.suffix("Channel")
                                .channelMapping(true, "even")
                                .channelMapping(false, "odd")
                )
                .get();
    }

    @Bean
    public IntegrationFlow eventFlow() {
        return IntegrationFlows.from("evenChannel")
                .log("even")
                .routeToRecipients(r -> r
                        .<Integer>recipient("10", source -> source / 10d  == 1)
                        .<Integer>recipient("20", source -> source / 20d == 1)
                        .defaultOutputChannel("else")
                )
                .get();
    }

    @Bean
    public IntegrationFlow oddFlow() {
        return IntegrationFlows.from("oddChannel")
                .log("odd")
                .get();
    }

    @Bean
    public IntegrationFlow flow10() {
        return IntegrationFlows.from("10")
                .log("10")
                .get();
    }

    @Bean
    public IntegrationFlow flow20() {
        return IntegrationFlows.from("20")
                .log("20")
                .get();
    }

    @Bean
    public IntegrationFlow elseFlow() {
        return IntegrationFlows.from("else")
                .log("else")
                .get();
    }
}
```  

1. `0 ~ ` 전달되는 숫자 중 짝수, 홀수를 구분해 `even`, `odd` 채널로 `routing` 한다. 
2. 짝수 중 `10 ~ 19` 범위는 `10` 채널로 보낸다. 
3. 짝수 중 `20 ~ 29` 범위는 `20` 채널로 보낸다. 
4. 짝수 중 위 2가지 경우에 포함되지 않는다면 `else` 채널로 보낸다. 

수행 결과의 로그는 아래와 같다.  

```
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=0, headers={id=74b09bfc-8b67-1338-f69c-d87e403e15bb, timestamp=1679833682191}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=1, headers={id=7961c158-d98a-4dad-3783-7fe8e2a07d57, timestamp=1679833682689}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=2, headers={id=ced85cd4-405a-c331-39fb-1b6c18c47a48, timestamp=1679833683193}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=2, headers={id=ced85cd4-405a-c331-39fb-1b6c18c47a48, timestamp=1679833683193}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=3, headers={id=48718455-37e7-3af5-b310-cf17060ce4fb, timestamp=1679833683693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=4, headers={id=3ba88023-542d-8dda-77f8-bae97ef2af02, timestamp=1679833684191}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=4, headers={id=3ba88023-542d-8dda-77f8-bae97ef2af02, timestamp=1679833684191}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=5, headers={id=3fdd86e0-bad0-2355-d55f-994291ac77ce, timestamp=1679833684690}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=6, headers={id=37caf97d-cfcf-ef6f-41ce-5c6a69151763, timestamp=1679833685192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=6, headers={id=37caf97d-cfcf-ef6f-41ce-5c6a69151763, timestamp=1679833685192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=7, headers={id=8ff7eadd-321c-ad06-33fd-534656f2df62, timestamp=1679833685689}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=8, headers={id=c4b0882e-ec4e-125f-4e4a-50a626e4b4e3, timestamp=1679833686193}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=8, headers={id=c4b0882e-ec4e-125f-4e4a-50a626e4b4e3, timestamp=1679833686193}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=9, headers={id=283dd82c-e4eb-ca2d-c4a7-be02b899d11b, timestamp=1679833686691}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=10, headers={id=e760d48b-98e8-0d4e-f8ae-b42ad3174ba3, timestamp=1679833687189}]
INFO 30809 --- [   scheduling-1] 10                                       : GenericMessage [payload=10, headers={id=e760d48b-98e8-0d4e-f8ae-b42ad3174ba3, timestamp=1679833687189}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=11, headers={id=446a0428-0a6a-89a1-64b9-e91c332aac4c, timestamp=1679833687692}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=12, headers={id=284f2066-f7ed-777c-bc23-ae23e551bf50, timestamp=1679833688192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=12, headers={id=284f2066-f7ed-777c-bc23-ae23e551bf50, timestamp=1679833688192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=13, headers={id=dd9e9ad1-4e53-42ee-531f-71b8d4c4d828, timestamp=1679833688693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=14, headers={id=164c317e-df75-5515-beb5-87c1a252083d, timestamp=1679833689193}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=14, headers={id=164c317e-df75-5515-beb5-87c1a252083d, timestamp=1679833689193}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=15, headers={id=329332d2-5351-0964-b6e2-ed62bc42c4d9, timestamp=1679833689693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=16, headers={id=c18033b6-5cdf-e267-42e8-e4f2a87f1b60, timestamp=1679833690192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=16, headers={id=c18033b6-5cdf-e267-42e8-e4f2a87f1b60, timestamp=1679833690192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=17, headers={id=46dcc9f6-2c25-59d3-1541-e02ba0a38e95, timestamp=1679833690692}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=18, headers={id=3e7bb335-2a24-d97f-34d5-649064ba9472, timestamp=1679833691193}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=18, headers={id=3e7bb335-2a24-d97f-34d5-649064ba9472, timestamp=1679833691193}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=19, headers={id=6140f1a5-e3a1-4b11-db54-94294cccc9f9, timestamp=1679833691693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=20, headers={id=274bfc09-3efb-1a41-a1dd-58809c8612fb, timestamp=1679833692192}]
INFO 30809 --- [   scheduling-1] 20                                       : GenericMessage [payload=20, headers={id=274bfc09-3efb-1a41-a1dd-58809c8612fb, timestamp=1679833692192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=21, headers={id=8d1ba0d8-f476-e67b-da4d-c3791b95e13e, timestamp=1679833692690}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=22, headers={id=18ae6ce4-3ecb-f3fd-3cea-70cbe6808053, timestamp=1679833693192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=22, headers={id=18ae6ce4-3ecb-f3fd-3cea-70cbe6808053, timestamp=1679833693192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=23, headers={id=1ad45f7f-cfca-467e-c439-f32281280712, timestamp=1679833693693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=24, headers={id=334c9578-046d-527a-da3b-7f152e25ca37, timestamp=1679833694192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=24, headers={id=334c9578-046d-527a-da3b-7f152e25ca37, timestamp=1679833694192}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=25, headers={id=1a5fc1e0-59a5-b583-b35c-d71982b1b904, timestamp=1679833694693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=26, headers={id=9d0432a4-1fbe-a971-f6cb-b56f34e5af64, timestamp=1679833695194}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=26, headers={id=9d0432a4-1fbe-a971-f6cb-b56f34e5af64, timestamp=1679833695194}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=27, headers={id=e6d44e6a-6b2a-efdf-a219-3314385a468a, timestamp=1679833695689}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=28, headers={id=ff4950fb-5103-6100-2e3d-ff030b8fb82a, timestamp=1679833696189}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=28, headers={id=ff4950fb-5103-6100-2e3d-ff030b8fb82a, timestamp=1679833696189}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=29, headers={id=97156ba1-52a4-26f2-f16a-01c1509ad96c, timestamp=1679833696692}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=30, headers={id=058f348b-121b-fec4-c45b-55ab9c03799e, timestamp=1679833697190}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=30, headers={id=058f348b-121b-fec4-c45b-55ab9c03799e, timestamp=1679833697190}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=31, headers={id=8d7dd15d-4d5e-912f-e5ea-e2738dd5fb7c, timestamp=1679833697689}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=32, headers={id=e2c2d0a6-0b80-7328-1d2e-371b213dac3c, timestamp=1679833698193}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=32, headers={id=e2c2d0a6-0b80-7328-1d2e-371b213dac3c, timestamp=1679833698193}]
INFO 30809 --- [   scheduling-1] odd                                      : GenericMessage [payload=33, headers={id=eca9a73b-134f-82d1-256c-1450444a93ff, timestamp=1679833698693}]
INFO 30809 --- [   scheduling-1] even                                     : GenericMessage [payload=34, headers={id=cbdddd57-9d1b-3c00-4595-a0f8aaa5d56d, timestamp=1679833699192}]
INFO 30809 --- [   scheduling-1] else                                     : GenericMessage [payload=34, headers={id=cbdddd57-9d1b-3c00-4595-a0f8aaa5d56d, timestamp=1679833699192}]
```  

### Splitters
`split()` 메서드를 사용하면 페이로드가 집합 구조일때 개별 메시지로 분리가 가능하다. 

- `Interable`
- `Iterator`
- `Array`
- `Stream`
- `Publisher`

혹은 `SpEL` 이나 람다를 사용해서 커스텀한 동작도 구현할 수 있다. 
아래는 `Stream` 과 `,` 로 합쳐진 `String` 형태를 `split()` 을 사용해 개별 메시로 분리하는 예제이다.  

```java
@Configuration
public class SplitterConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow array() {
        return IntegrationFlows.fromSupplier(
                        () -> IntStream
                                .range(integerSource().get(), integerSource().addAndGet(3))
                                .boxed(),
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("inputArray")
                .get();
    }

    @Bean
    public IntegrationFlow string() {
        return IntegrationFlows.fromSupplier(
                        () -> IntStream
                                .range(integerSource().get(), integerSource().addAndGet(3))
                                .boxed()
                                .map(Object::toString)
                                .map("Hello "::concat)
                                .collect(Collectors.joining(",")),
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("inputString")
                .get();
    }

    @Bean
    public IntegrationFlow splitArrayFlow() {
        return IntegrationFlows.from("inputArray")
                .split()
                .channel("result")
                .get();
    }

    @Bean
    public IntegrationFlow splitStringFlow() {
        return IntegrationFlows.from("inputString")
                .split(s -> s.applySequence(false).delimiters(","))
                .channel("result")
                .get();
    }

    @Bean
    public IntegrationFlow resultFlow() {
        return IntegrationFlows.from("result")
                .log("result")
                .get();
    }
}
```  

`Stream` 타입은 은 기본 `split()` 을 통해 메시지를 분리하고, `String` 은 `delimiters()` 에 구분자를 정의해서 분리한다. 
그 결과 로그는 아래와 같다.  

```
beforeSplitArray                         : GenericMessage [payload=java.util.stream.IntPipeline$1@167cb340, headers={id=5f4953c7-9cf8-7525-38d5-371202d20d4b, timestamp=1679905569642}]
result                                   : GenericMessage [payload=0, headers={sequenceNumber=1, correlationId=5f4953c7-9cf8-7525-38d5-371202d20d4b, id=35440b02-4540-5158-1991-67a152f443f6, sequenceSize=0, timestamp=1679905569644}]
result                                   : GenericMessage [payload=1, headers={sequenceNumber=2, correlationId=5f4953c7-9cf8-7525-38d5-371202d20d4b, id=1a3a6fc3-f602-ef59-1c7c-d6a88995cb7d, sequenceSize=0, timestamp=1679905569645}]
result                                   : GenericMessage [payload=2, headers={sequenceNumber=3, correlationId=5f4953c7-9cf8-7525-38d5-371202d20d4b, id=df30196a-d94d-58a6-bb98-bceaca3c12de, sequenceSize=0, timestamp=1679905569645}]
beforeSplitString                        : GenericMessage [payload=Hello 3,Hello 4,Hello 5, headers={id=d16c08c4-7f6a-e0b7-e564-e19910400c86, timestamp=1679905569646}]
result                                   : GenericMessage [payload=Hello 3, headers={id=3b7440d6-13f6-b198-2573-779185507bd0, timestamp=1679905569646}]
result                                   : GenericMessage [payload=Hello 4, headers={id=2e475617-d2b7-b911-02f2-b59613f8206a, timestamp=1679905569646}]
result                                   : GenericMessage [payload=Hello 5, headers={id=83cf326c-baea-c3b5-a9c5-786e3bdc2239, timestamp=1679905569646}]
beforeSplitArray                         : GenericMessage [payload=java.util.stream.IntPipeline$1@72c699f0, headers={id=20cd3d35-81ee-b7d3-c917-965a26daeff7, timestamp=1679905570645}]
result                                   : GenericMessage [payload=6, headers={sequenceNumber=1, correlationId=20cd3d35-81ee-b7d3-c917-965a26daeff7, id=7bc29453-c893-2989-be81-22e4f94f52a3, sequenceSize=0, timestamp=1679905570645}]
result                                   : GenericMessage [payload=7, headers={sequenceNumber=2, correlationId=20cd3d35-81ee-b7d3-c917-965a26daeff7, id=5ca525e6-9077-362e-f134-cb8f6d274d39, sequenceSize=0, timestamp=1679905570645}]
result                                   : GenericMessage [payload=8, headers={sequenceNumber=3, correlationId=20cd3d35-81ee-b7d3-c917-965a26daeff7, id=e657bd3d-ec21-1948-7dc5-ffb99d75260a, sequenceSize=0, timestamp=1679905570645}]
beforeSplitString                        : GenericMessage [payload=Hello 9,Hello 10,Hello 11, headers={id=d72d9619-f8db-4230-167b-1bc822c4db0f, timestamp=1679905570645}]
result                                   : GenericMessage [payload=Hello 9, headers={id=0e02f313-0971-68f5-264c-58a45c5769bf, timestamp=1679905570645}]
result                                   : GenericMessage [payload=Hello 10, headers={id=4da01774-254f-fd69-38e7-530a12060f84, timestamp=1679905570645}]
result                                   : GenericMessage [payload=Hello 11, headers={id=4087b798-b5ba-5b4b-996e-91133d765254, timestamp=1679905570645}]
```



### Aggregators

### Service Activators
`ServiceActivator` 에 해당하는 `Java DSL` 메서드는 `handle()` 이다. 
`handle()` 에서는 기존 `ServiceActivator` 에서 수행했던 것과 동일하게, 
`POJO` 메소드를 호출해서 변환을 수행한다거나 관련 동작을 이어 수행하는 등이 가능하다.  

`handle()` 는 `payload` 와 `header` 를 파라미터로 제공하는 람다를 사용할 수 있다. 
그리고 `payload` 의 타입 캐스팅의 경우 `<TypeClass>handle()` 과 같이 사용하거나, 
`handle(TypeClass.class, (payload, header) -> {})` 와 같이 사용할 수 있다.  

```java
@Configuration
public class ServiceActivatorConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow handleFlow() {
        return IntegrationFlows.from("input")
                .<Integer>handle((payload, headers) -> payload * 10)
                .log("handleFlow")
                .channel("handleChannel")
                .get();
    }

    @Bean
    public IntegrationFlow handleTypeCastFlow() {
        return IntegrationFlows.from("handleChannel")
                .handle(Integer.class, (payload, headers) -> payload * payload)
                .log("handleTypeCastFlow")
                .get();
    }
}
```  

`handleFlow()` 는 `input` 채널의 숫자를 받아 `<Integer>` 로 제네릭 타입을 명시하는 방식으로 `handle()` 을 사용한다. 
그리고 `handleTypeCastFlow()` 는 따로 제네릭 타입을 명시하지 않고 `handle(Integer.class, () -> {})` 을 사용해서, 
타입 캐스팅을 제공하는 `handle()` 메소드를 사용했다. 

### Gateway
`Gateway` 역할을 하는 `Java DSL` 메소드는 `gateway()` 로 입력 채널을 통해 다른 엔드포인트나 통합 플로우를 호출하고 응답을 기다린다. 
이런 게이트웨이를 활용하면 분배된 작은 플로우를 합쳐 큰 하나의 플로우 동작을 정의할 수 있다. 
3가지 형태로 오버로딩 돼 있는데 그 형태는 아래와 같다.  

- `gateway(String)` : `requestChannel` 문자열을 받아 입력 채널에 메시지를 해당하는 채널의 엔드포인트로 전달한다. 
- `gateway(MessageChannel)` : `MessageChannel` 타입의 `requestChannel` 을 직접 주입 받아 입력 채널에 메시지를 해당하는 채널의 엔드포인트로 전달한다. 
- `gateway(IntergrationFlow)` : `IntegrationFlow` 인 플로우 자체를 주입 받아 입력 채널의 메시지를 해당 플로우로 전달 한다. 

`gateway` 를 통해 메시지를 전달하는 플로우는 입력 메시지를 해당 플로우에 요청 후 응답을 다시 다운 스트림으로 전달하는 형태이다. 
그러므로 다운 스트림의 엔드포인트가 메시지를 시 반환하는 형태이여야 이후 다운 스트림이 계속 진행 가능하다. 
예를 들어 특정 플로우가 `log()` 와 같은 엔드포인트라면 응답 반환이 없으므로 이후 플로우 진행이 되지 않을 수 있으므로 주의해야 한다. 
위와 같은 경우 `replyTimeout` 혹은 `requestTimeout` 등으로 타입아웃을 설정해서 무한정 다운 스트림의 응답을 기다리지 않도록 해야 한다.  

```java
@Configuration
public class GatewayConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow mainFlow() {
        return IntegrationFlows.from("input")
                .log("startMainFlow")
                .gateway(this.subFlow())
                .gateway("subChannel")
//                .gateway(this.subNoReplyFlow(), gatewayEndpointSpec -> gatewayEndpointSpec.replyTimeout(0L))
                .log("endMainFlow")
                .handle(Integer.class, (payload, headers) -> payload * 10)
                .log("result")
                .get();
    }

    @Bean
    public IntegrationFlow subFlow() {
        return flow -> flow
                .log("subFlow")
                .handle(Integer.class, (payload, headers) -> payload * 10);
    }

    @Bean
    public IntegrationFlow subChannelFlow() {
        return IntegrationFlows.from("subChannel")
                .log("subChannelFlow")
                .handle(Integer.class, (payload, headers) -> payload * 10)
                .get();
    }

    @Bean
    public IntegrationFlow subNoReplyFlow() {
        return flow -> flow
                .log("subNoReplyFlow");
    }
}
```  

`input` 채널로 부터 전달되는 메시지는 `mainFlow` 에서 아래와 같이 다운 스트림 플로우로 전달 된다. 
그리고 각 플로우에서는 메시지의 값에 10을 곱한 값을 반환한다. 

```
mainFlow -> subFlow -> subChannelFlow -> mainFlow
```  

위 플로우의 실행 로그는 아래와 같다.  

```
startMainFlow                            : GenericMessage [payload=0, headers={id=d835afc5-03db-ca48-377d-981e70c79f2f, timestamp=1680341381199}]
subFlow                                  : GenericMessage [payload=0, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@11f82636, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@11f82636, id=1c84628a-a13e-489b-993a-ca1151279dc6, timestamp=1680341381211}]
subChannelFlow                           : GenericMessage [payload=0, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@ef24df0, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@ef24df0, id=c8f9e1f0-1092-7cd2-2347-f37107228efd, timestamp=1680341117427}]
endMainFlow                              : GenericMessage [payload=0, headers={id=28e361f2-6636-6fa4-825c-c37c8d97bba8, timestamp=1680341117427}]
result                                   : GenericMessage [payload=0, headers={id=26432ab3-dfbb-6e62-bd8c-5dd786ede780, timestamp=1680341117427}]
startMainFlow                            : GenericMessage [payload=1, headers={id=24b044b9-a990-fa7d-b0c6-3b24f24e6a82, timestamp=1680341117917}]
subFlow                                  : GenericMessage [payload=1, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@50695682, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@50695682, id=25bbcd42-10b7-077d-e829-b376412185bc, timestamp=1680341117917}]
subChannelFlow                           : GenericMessage [payload=10, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@6eadd4a6, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@6eadd4a6, id=4761d7cd-cc3e-f1a7-fa69-419185565f5e, timestamp=1680341117917}]
endMainFlow                              : GenericMessage [payload=100, headers={id=3b583a1a-8ffb-36e0-c45d-f52ad196ebd3, timestamp=1680341117917}]
result                                   : GenericMessage [payload=1000, headers={id=6df858a3-8d86-34c8-a0a2-295c1f213c92, timestamp=1680341117917}]
startMainFlow                            : GenericMessage [payload=2, headers={id=66dd5158-08e1-8f1f-bda8-6a5b3902de0e, timestamp=1680341118418}]
subFlow                                  : GenericMessage [payload=2, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5c077803, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5c077803, id=16085425-eeef-5b32-a19a-bcd901e8efaa, timestamp=1680341118419}]
subChannelFlow                           : GenericMessage [payload=20, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@451572fd, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@451572fd, id=f76b0498-2f02-1450-f663-4fadc76bf0f9, timestamp=1680341118419}]
endMainFlow                              : GenericMessage [payload=200, headers={id=ca37d8cb-e22d-4fdc-85d7-96592b0692e1, timestamp=1680341118419}]
result                                   : GenericMessage [payload=2000, headers={id=0b13f34f-7e1c-1588-a6b7-251a0e57145f, timestamp=1680341118419}]
startMainFlow                            : GenericMessage [payload=3, headers={id=ad1be481-5383-e735-e2b5-e559278d2e24, timestamp=1680341118916}]
subFlow                                  : GenericMessage [payload=3, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5fd2f19b, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5fd2f19b, id=88ff84b5-f122-9700-b90c-5d59c9bf320d, timestamp=1680341118917}]
subChannelFlow                           : GenericMessage [payload=30, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5fda0623, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5fda0623, id=7f124bbc-c4ab-6217-ffc1-82a193d0f68c, timestamp=1680341118917}]
endMainFlow                              : GenericMessage [payload=300, headers={id=7ae714b2-e805-7d0f-ae0b-cbd86165c80f, timestamp=1680341118918}]
result                                   : GenericMessage [payload=3000, headers={id=29ab014f-bab6-83a4-3944-fe249b43ed6b, timestamp=1680341118918}]
```  

로그로 실행된 흐름을 보면 첫 로그인 `startMainFlow` 부터 시작해 `result` 까지 모두 정상적으로 메시지가 전달되면 실행된 것을 확인 할 수 있다.  

위 코드를 보면 `subNoReplyFlow` 는 사용되지 않고 있다. 
아래 코드를 주석 해제 했을 때 메시지가 전달되는 플로우는 아래와 같다.  

```java
IntegrationFlows.from("input")
        .log("startMainFlow")
        .gateway(this.subFlow())
        .gateway("subChannel")
        // 주석 해제
        .gateway(this.subNoReplyFlow(), gatewayEndpointSpec -> gatewayEndpointSpec.replyTimeout(500L))
        .log("endMainFlow")
        .handle(Integer.class, (payload, headers) -> payload * 10)
        .log("result")
        .get();
```  

```
mainFlow -> subFlow -> subChannelFlow -> subNoReplyFlow(500ms 후) -> mainFlow
```  

하지만 주석처리가 있을 때와 다른 점은 `subNoReplyFlow` 는 메시지를 응답하지 않기 때문에 이후 플로우는 진행되지 않고, 
다시 처음부터 플로우가 수행된다. 
실제 동작을 로그로 살펴보면 아래와 같다.  

```
startMainFlow                            : GenericMessage [payload=0, headers={id=d835afc5-03db-ca48-377d-981e70c79f2f, timestamp=1680341381199}]
subFlow                                  : GenericMessage [payload=0, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@11f82636, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@11f82636, id=1c84628a-a13e-489b-993a-ca1151279dc6, timestamp=1680341381211}]
subChannelFlow                           : GenericMessage [payload=0, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@6961a9ea, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@6961a9ea, id=e6861a6f-dca2-acc7-0267-96e39e6d60e6, timestamp=1680341381212}]
subNoReplyFlow                           : GenericMessage [payload=0, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@18daad1, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@18daad1, id=527e19b0-4a73-0b16-3e54-c9a469469408, timestamp=1680341381213}]
startMainFlow                            : GenericMessage [payload=1, headers={id=33c10c4a-8bde-b685-b4f5-1bc82d6e6635, timestamp=1680341386220}]
subFlow                                  : GenericMessage [payload=1, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@2e985c4a, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@2e985c4a, id=d47dba12-ce3a-953e-886b-456995f7e1cd, timestamp=1680341386220}]
subChannelFlow                           : GenericMessage [payload=10, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@60907421, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@60907421, id=60034c04-9d3e-355a-5803-fe7ff3b482b8, timestamp=1680341386221}]
subNoReplyFlow                           : GenericMessage [payload=100, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@100bb0b9, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@100bb0b9, id=9e26b3f0-a98d-fc9b-425f-27926bea2d39, timestamp=1680341386221}]
startMainFlow                            : GenericMessage [payload=2, headers={id=4e56578a-deb0-a466-35fe-0f8a04cd1c5e, timestamp=1680341391227}]
subFlow                                  : GenericMessage [payload=2, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@556a5db1, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@556a5db1, id=d8b1a3e3-1bc6-b377-ab6e-ce79d0b4af27, timestamp=1680341391227}]
subChannelFlow                           : GenericMessage [payload=20, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@397c215a, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@397c215a, id=1fbd754c-5c30-7dc6-d678-f119b6301963, timestamp=1680341391228}]
subNoReplyFlow                           : GenericMessage [payload=200, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5aae6b28, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@5aae6b28, id=3a8e5083-b63e-8d76-f107-feaba3205db8, timestamp=1680341391228}]
startMainFlow                            : GenericMessage [payload=3, headers={id=3345a925-6495-0f05-02ba-5f509e99f354, timestamp=1680341396231}]
subFlow                                  : GenericMessage [payload=3, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@17ed86bc, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@17ed86bc, id=ae68f0ca-6ff2-b6a3-cc53-7ab82bc78b96, timestamp=1680341396233}]
subChannelFlow                           : GenericMessage [payload=30, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@106dd625, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@106dd625, id=2c168625-8265-7706-c44c-0b6b91500d0f, timestamp=1680341396233}]
subNoReplyFlow                           : GenericMessage [payload=300, headers={replyChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@48c35383, errorChannel=org.springframework.messaging.core.GenericMessagingTemplate$TemporaryReplyChannel@48c35383, id=b0f8f8c3-5877-e18c-1251-ebec38c84832, timestamp=1680341396233}]
```  

`subNoReplyFlow` 가 추가 됐을 때 로그로 실행된 흐름을 보면 첫 로그인 `startMainFlow` 부터 시작해 `subNoReplyFlow` 까지만
로그가 찍히고 다시 `startMainFlow` 로 이어지는 것을 확인 할 수 있다. 
앞서 언급한 것처럼 `subNoReplyFlow` 에서 반환되는 값이 없기 때문에 `500ms` 전도 응답을 기다린 후, 
해당 메시지의 플로우는 종료하고 다음 메시지로 플로우가 전환되는 것이다.  

### Intercept
`5.3` 이후 버전 부터 사용할 수 있는 `intercept()` 는 현재 플로우에서 추가적인 `MessageChannel` 을 만들지 않고, 
플로우에 하나 이상의 `ChannelInterceptor` 를 등록해 필터링 등의 동작을 수행 할 수 있다.  

아래는 플로우를 통해 전달되는 정수형 메시지 중 `intercept()` 를 사용해서 짝수만 허용하고 홀수인 메시지는 거부하는 예시이다.  

```java
@Configuration
public class InterceptConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("input")
                .get();
    }

    @Bean
    public IntegrationFlow interceptFlow() {
        return IntegrationFlows.from("input")
                .log("beforeIntercept")
                .intercept(new MessageSelectingInterceptor(message -> {
                    try {
                        return (Integer)message.getPayload() % 2 == 0;
                    } catch (Exception e) {
                        return false;
                    }

                }))
                .handle(Integer.class, (payload, headers) -> payload * 10)
                .log("afterIntercept")
                .get();
    }
}
```  

위 구성을 실행하면 아래와 같이 최종적으로 `afterIntercept` 로그에는 짝수인 정수에만 `10` 을 곱한 결과가 출력되는 것을 확인 할 수 있다.  

```
beforeIntercept                          : GenericMessage [payload=0, headers={id=dfbc77a3-04e7-89da-475a-f963bbd5ad95, timestamp=1680342544373}]
afterIntercept                           : GenericMessage [payload=0, headers={id=65faeb91-1eda-5150-fcb2-c7156d9aedca, timestamp=1680342544374}]
beforeIntercept                          : GenericMessage [payload=1, headers={id=1ef2112f-8fea-3ab5-d5fe-0d38107a1469, timestamp=1680342544876}]
beforeIntercept                          : GenericMessage [payload=2, headers={id=78cb31ca-840c-5aa1-0367-ecb932978eee, timestamp=1680342545372}]
afterIntercept                           : GenericMessage [payload=20, headers={id=2496f989-4f67-fcc3-9484-be9a70aaf752, timestamp=1680342545372}]
beforeIntercept                          : GenericMessage [payload=3, headers={id=613a768f-a709-63cd-481a-e08706487816, timestamp=1680342545875}]
beforeIntercept                          : GenericMessage [payload=4, headers={id=907fd60b-b176-5279-00cb-052387c2f592, timestamp=1680342546371}]
afterIntercept                           : GenericMessage [payload=40, headers={id=cf01b787-c48a-2a9e-524e-fa36da83e4ab, timestamp=1680342546371}]
```  

거부된 메시지는 아래와 같이 에러와 함께 출력된다.  

```
ERROR 83110 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessageDeliveryException: selector 'com.windowforsun.spring.integration.javadsl.InterceptConfig$$Lambda$443/0x0000000800345040@6dc5eea7' did not accept message, failedMessage=GenericMessage [payload=3, headers={id=613a768f-a709-63cd-481a-e08706487816, timestamp=1680342545875}]
```  


### Log, wireTap
`Java DLS` 의 `log()` 를 사용하면 메시지 플로우 중간 흐름을 기록할 수 있다. 
내부적으로는 `LoggingHandler` 를 구독하는 `WireTab` 과 `ChannelInterceptor` 로 구현된다. 
전달되는 메시지를 다음 엔드포인트 혹은 현재 채널에 기록하는 동작을 수행한다. 
아래는 다양한 `log()` 의 사용 예시이다.  

```java
@Configuration
public class LogConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .log(LoggingHandler.Level.ERROR)
                .transform(Integer.class, source -> source * 10)
                .log("log2")
                .transform(Integer.class, source -> source * 10)
                .log(LoggingHandler.Level.INFO, "log3")
                .transform(Integer.class, source -> source * 10)
                .log(LoggingHandler.Level.WARN, "log3", message -> message.getPayload())
                .get();
    }
}
```  

`log()` 메소드는 출력할 로그 레벨을 설정하거나, 출력에 사용할 카테고리 설정 및 출력 메시지 포맷등 다양한 활용이 가능하다. 
위 구성을 실행하면 아래와 같은 로그가 출력된다. 

```
ERROR 84577 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : GenericMessage [payload=0, headers={id=ecb78132-4bc2-93f8-bb7d-8b29cc6bc6ca, timestamp=1680430979704}]
 INFO 84577 --- [   scheduling-1] log2                                     : GenericMessage [payload=0, headers={id=47d5132d-22f4-e92c-40a0-aac60ba45218, timestamp=1680430979705}]
 INFO 84577 --- [   scheduling-1] log3                                     : GenericMessage [payload=0, headers={id=5c6b6190-141f-9751-50be-e28e96f6e65b, timestamp=1680430979705}]
 WARN 84577 --- [   scheduling-1] log3                                     : 0
ERROR 84577 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : GenericMessage [payload=1, headers={id=2a61ad14-388b-0642-58d0-d36b6bbb241a, timestamp=1680430980206}]
 INFO 84577 --- [   scheduling-1] log2                                     : GenericMessage [payload=10, headers={id=8d8512d4-29ef-64a9-0c04-c1d193e9a634, timestamp=1680430980206}]
 INFO 84577 --- [   scheduling-1] log3                                     : GenericMessage [payload=100, headers={id=23ddcdb0-bd6f-ba95-2d6c-44bb0e0b29c6, timestamp=1680430980206}]
 WARN 84577 --- [   scheduling-1] log3                                     : 1000
ERROR 84577 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : GenericMessage [payload=2, headers={id=c495f7ea-4ec4-95f8-ff5d-f00f841f043c, timestamp=1680430980706}]
 INFO 84577 --- [   scheduling-1] log2                                     : GenericMessage [payload=20, headers={id=f15673f9-fb89-cfcb-77b0-750aa123cfa3, timestamp=1680430980706}]
 INFO 84577 --- [   scheduling-1] log3                                     : GenericMessage [payload=200, headers={id=c0d6fbd2-b7bf-2300-52ac-200f9a80bfbd, timestamp=1680430980706}]
 WARN 84577 --- [   scheduling-1] log3                                     : 2000
ERROR 84577 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : GenericMessage [payload=3, headers={id=7d730a53-6c18-0b5c-7e0e-b1135f782883, timestamp=1680430981207}]
 INFO 84577 --- [   scheduling-1] log2                                     : GenericMessage [payload=30, headers={id=8a5e6ae7-899c-a39c-201d-d8940d4e9410, timestamp=1680430981209}]
 INFO 84577 --- [   scheduling-1] log3                                     : GenericMessage [payload=300, headers={id=9789cf9a-03d5-787c-2eb3-dd36a597c457, timestamp=1680430981209}]
 WARN 84577 --- [   scheduling-1] log3                                     : 3000
```  

`wireTab()` 은 `log()` 보다는 조금 더 저수준의 메소드로 메시지 플로우 중간에 메시지를 다른 채널로도 전달 할 수 있다. 
메시지 하나에 별도의 추가 흐름을 만들거나 로그, 통계 등의 작업을 할 수 있다.  

```java
@Configuration
public class WireTabConfig {
    @Bean
    public AtomicInteger integerSource() {
        return new AtomicInteger();
    }

    @Bean
    public IntegrationFlow intput() {
        return IntegrationFlows.fromSupplier(
                        integerSource()::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .wireTap("firstWireTab.input")
                .transform(Integer.class, source -> source * 10)
                .wireTap(this.secondWireTab())
                .log("end")
                .get();
    }

    @Bean
    public IntegrationFlow firstWireTab() {
        return flow -> flow
                .log("firstWireTab");
    }

    @Bean
    public IntegrationFlow secondWireTab() {
        return flow -> flow
                .log("secondWireTab")
                .transform(Integer.class, source -> source * 10)
                .channel("subChannel");
    }

    @Bean
    public IntegrationFlow subFlow() {
        return IntegrationFlows.from("subChannel")
                .log("subFlow")
                .get();
    }
}
```  

위 구성은 `input()` 에서 시작한 메시지가 `firstWireTab()`  에서는 로깅만 수행하고, 
`secondWireTab()` 에서는 로깅과 함께 `transfer()` 동작으 수행 후 별도 플로우인 `subChannel` 로 메시지를 전송한다. 
그러면 `secondWireTab()` 채널에서 전달된 메시지는 `subFlow()` 전달되고 로깅이 수행된다. 
그리고 메인 플로우격인 `input()` 에서는 `secondWireTab()` 과는 별개로 메시지 전달이 수행되고, 
최종 `end` 로그에는 `subFlow()` 의 로그와는 다른 메시지가 출력되는 것을 확인 할 수 있다.  

```
firstWireTab                             : GenericMessage [payload=0, headers={id=6804c5be-7dca-d8f9-cd11-44376e97ceb9, timestamp=1680431527149}]
secondWireTab                            : GenericMessage [payload=0, headers={id=4c23fb8f-391f-56b0-43c8-b1fe14f13a1e, timestamp=1680431527150}]
subFlow                                  : GenericMessage [payload=0, headers={id=5150ee47-9d11-4378-f823-0805003af8be, timestamp=1680431527150}]
end                                      : GenericMessage [payload=0, headers={id=4c23fb8f-391f-56b0-43c8-b1fe14f13a1e, timestamp=1680431527150}]
c.w.s.i.javadsl.JavaDslApplication       : Started JavaDslApplication in 0.699 seconds (JVM running for 1.056)
firstWireTab                             : GenericMessage [payload=1, headers={id=dc1b4057-1c5b-96f6-56e5-a0e11892ca02, timestamp=1680431527650}]
secondWireTab                            : GenericMessage [payload=10, headers={id=4207d28d-458b-3f0a-2cd8-10324bd19a71, timestamp=1680431527650}]
subFlow                                  : GenericMessage [payload=100, headers={id=463f87c3-ed09-e61d-d27b-6624e5766415, timestamp=1680431527650}]
end                                      : GenericMessage [payload=10, headers={id=4207d28d-458b-3f0a-2cd8-10324bd19a71, timestamp=1680431527650}]
firstWireTab                             : GenericMessage [payload=2, headers={id=e0fa085c-ab88-5904-9189-e88723ba4264, timestamp=1680431528148}]
secondWireTab                            : GenericMessage [payload=20, headers={id=4082ab4b-8897-13c2-41fd-cb320a90e549, timestamp=1680431528149}]
subFlow                                  : GenericMessage [payload=200, headers={id=1122ee78-f750-f13f-3284-af0874baf0c1, timestamp=1680431528149}]
end                                      : GenericMessage [payload=20, headers={id=4082ab4b-8897-13c2-41fd-cb320a90e549, timestamp=1680431528149}]
firstWireTab                             : GenericMessage [payload=3, headers={id=028ff01c-0251-470a-4f76-24426b345442, timestamp=1680431528651}]
secondWireTab                            : GenericMessage [payload=30, headers={id=817ddc31-c66b-e112-80bd-6b8fb98fce45, timestamp=1680431528652}]
subFlow                                  : GenericMessage [payload=300, headers={id=c971ea28-21d7-a2bf-504f-0fc2bf17bfc0, timestamp=1680431528652}]
end                                      : GenericMessage [payload=30, headers={id=817ddc31-c66b-e112-80bd-6b8fb98fce45, timestamp=1680431528652}]
firstWireTab                             : GenericMessage [payload=4, headers={id=1f0c9566-a9bb-9229-d01c-672885cd7666, timestamp=1680431529148}]
secondWireTab                            : GenericMessage [payload=40, headers={id=5755fe42-df60-6c4c-ba9c-a22d129194f0, timestamp=1680431529149}]
subFlow                                  : GenericMessage [payload=400, headers={id=9dcfbe2a-a468-8974-d853-defefe0bea35, timestamp=1680431529150}]
end                                      : GenericMessage [payload=40, headers={id=5755fe42-df60-6c4c-ba9c-a22d129194f0, timestamp=1680431529149}]
firstWireTab                             : GenericMessage [payload=5, headers={id=714f8ea1-ebcd-6f9b-a6ff-5e6c45312cc0, timestamp=1680431529647}]
secondWireTab                            : GenericMessage [payload=50, headers={id=f3e0b240-af9a-b4b1-3647-82635d668bc0, timestamp=1680431529649}]
subFlow                                  : GenericMessage [payload=500, headers={id=603d22c7-2789-7f5a-5316-8fbc89a97c20, timestamp=1680431529649}]
end                                      : GenericMessage [payload=50, headers={id=f3e0b240-af9a-b4b1-3647-82635d668bc0, timestamp=1680431529649}]
```  



---  
## Reference
[Spring Integration Java DSL](https://docs.spring.io/spring-integration/reference/html/dsl.html#java-dsl)  
[Spring Integration Java DSL sample](https://dzone.com/articles/spring-integration-java-dsl)  
[Spring Integration - Poller (Polling Consumer)](https://springsource.tistory.com/49)  
[Using Subflows in Spring Integration](https://www.baeldung.com/spring-integration-subflows)  
[Wire Tab](https://www.enterpriseintegrationpatterns.com/patterns/messaging/WireTap.html)  
