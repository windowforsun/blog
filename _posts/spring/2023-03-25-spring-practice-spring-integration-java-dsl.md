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
beforeSplitArray  ㄷ                       : GenericMessage [payload=java.util.stream.IntPipeline$1@72c699f0, headers={id=20cd3d35-81ee-b7d3-c917-965a26daeff7, timestamp=1679905570645}]
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

### Gateway

### Intercept

### Log, wireTap




---  
## Reference
[Spring Integration Java DSL](https://docs.spring.io/spring-integration/reference/html/dsl.html#java-dsl)  
[Spring Integration Java DSL sample](https://dzone.com/articles/spring-integration-java-dsl)  
[Spring Integration - Poller (Polling Consumer)](https://springsource.tistory.com/49)  
[Using Subflows in Spring Integration](https://www.baeldung.com/spring-integration-subflows)  
