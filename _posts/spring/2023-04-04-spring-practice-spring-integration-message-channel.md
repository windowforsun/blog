--- 
layout: single
classes: wide
title: "[Spring 실습] "
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

## Spring Integration MessageChannel
`Spring Integration` 에서 `MessageChannel` 은 메시지 `Producer` 와 `Consumer` 를 구조적으로 분리하지만, 
메시지 흐름으로 볼때는 연결을 해주는 역할을 한다. 
그리고 `MessageChannel` 에서는 데이터를 캡슐화한 `Message` 가 전달 된다.  

아래는 `Spring Integration` 에서 최상위 `MessageChannel` 인터페이스의 내용이다. 
메시지를 전송하는 `send()` 메서드는 전송이 성공하면 `true` 를 반환한다. 
만약 전송 중 전달된 `timeout` 시간을 넘어가면 `interrupt` 되면서 `false` 를 반환한다.  

```java
@FunctionalInterface
public interface MessageChannel {

	long INDEFINITE_TIMEOUT = -1;
    
	default boolean send(Message<?> message) {
		return send(message, INDEFINITE_TIMEOUT);
	}

	boolean send(Message<?> message, long timeout);
}
```  

위 최상위 `MessageChannel` 은 다시 각 특성을 지닌 2개의 채널로 분류된다. 
이후 나열할 실제 메시지 채널 구현체들은 위2개 중 하나의 메시지 채널 인터페이스를 구현한다. 

- `PollableChannel`
- `SubscribableChannel`

### PollableChannel

```java
public interface PollableChannel extends MessageChannel {

	@Nullable
	Message<?> receive();

	@Nullable
	Message<?> receive(long timeout);
}
```  

`PollableChannel` 은 `pollable` 즉 버퍼링을 사용하는 채널이다. 
`PollableChannel` 과 같은 버퍼링을 수행해주는 채널기반으로 메시지를 수신 받으면, 
인바운드 메시지들을 스로틀링해서 컨슈머를 과부하로부터 지킬 수 있다는 장점이 있다. 
하지만 별도로 `poller` 를 설정해 주어야 메시지 수신이 가능하다.  

만약 `timeout` 이 발생하면 `interrupt` 를 통해 `null` 이 반환 된다.  


### SubscribableChannel

```java
public interface SubscribableChannel extends MessageChannel {

	boolean subscribe(MessageHandler handler);

	boolean unsubscribe(MessageHandler handler);
}
```  

`SubscribableChannel` 은 자신을 구독하는 `MessageHandler` 인스턴스에 직접 메시지를 보내는 채널이다. 
구독 기반의 채널이기 때문에 구독관리에 대한 메소드를 정의하고 있다.  


## MessageChannel 구현체
`Spring Integration` 에서는 필요에 따라 선택해 사용할 수 있도록, 
다양한 종류의 메시지 채널을 제공한다. 

- `PubishSubscribeChannel`
- `QueueChannel`
- `PriorityChannel`
- `RendezvousChannel`
- `DirectChannel`
- `ExecutorChannel`
- `FluxMessageChannel`
- `ScopedChannel`

### PublishSubscribeChannel
`PublishSubscribeChannel` 은 전달 받은 메시지를 자신을 구독하는 모든 구독자(`MessageHandler`)에게 
브로드캐스트한다. 즉 전체 구독자에게 통지하거나 이벤트를 전체로 전파하는 데 사용 할 수 있다. 
채널을 구독하는 구독자는 `MessageHandler` 이여야 하고, 
채널은 자신을 구독하는 `MessageHandler` 의 `handleMessage(Message)` 를 `send(Message)` 를 통해 호출해서 메시지가 전달 된다. 
그러므로 `PublishSubscribeChannel` 을 사용하면 `N` 명의 구독자에게 메시지를 전달할 수 있다는 장점이 있다.  

`PublishSubscribeChannel` 은 아래와 같은 설정이 가능하다. 
- 메시지 전송을 수행 할 `Executor`
- 에러 메시지를 처리 할 `errorHandler`
- 최소 구독자 수인 `minSubscribers`
- 다운스트림에서 `resequence`, `aggregator` 사용을 위해 `sequenceSize` 와 `sequenceNumber`, `correlationId` 를 설정하는 `applySequence`
- 구독자가 없는 경우를 방지할 수 있는 `requireSubscribers`


```java
@Configuration
public class PublishSubscribeChannelConfig {
    @Bean
    public MessageChannel somePubSubChannel() {
        return new PublishSubscribeChannel();
    }

    @Bean
    public MessageChannel executorPubSubChannel() {
        return new PublishSubscribeChannel(Executors.newWorkStealingPool());
    }

    @Bean
    public MessageChannel sequencePubSubChannel() {
        PublishSubscribeChannel channel = new PublishSubscribeChannel();
        channel.setApplySequence(true);

        return channel;
    }

    @Bean
    public IntegrationFlow mainFlow() {
        AtomicInteger counter = new AtomicInteger();
        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .channel("somePubSubChannel")
                .channel("executorPubSubChannel")
                .channel("sequencePubSubChannel")
                .get();
    }

    @Bean
    public IntegrationFlow someSubFlow1() {
        return IntegrationFlows.from("somePubSubChannel")
                .log("someSubFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow someSubFlow2() {
        return IntegrationFlows.from("somePubSubChannel")
                .log("someSubFlow2")
                .get();
    }

    @Bean
    public IntegrationFlow executorSubFlow1() {
        return IntegrationFlows.from("executorPubSubChannel")
                .log("executorSubFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow executorSubFlow2() {
        return IntegrationFlows.from("executorPubSubChannel")
                .log("executorSubFlow2")
                .get();
    }

    @Bean
    public IntegrationFlow sequenceSubFlow1() {
        return IntegrationFlows.from("sequencePubSubChannel")
                .log("sequenceSubFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow sequenceSubFlow2() {
        return IntegrationFlows.from("sequencePubSubChannel")
                .log("sequenceSubFlow2")
                .get();
    }
}
```  

```
[Pool-1-worker-5] executorSubFlow1                         : GenericMessage [payload=0, headers={id=561e08a2-e41b-3a21-94d0-aeb106c75ff9, timestamp=1680717969935}]
[ool-1-worker-23] executorSubFlow2                         : GenericMessage [payload=0, headers={id=561e08a2-e41b-3a21-94d0-aeb106c75ff9, timestamp=1680717969935}]
[ool-1-worker-19] sequenceSubFlow1                         : GenericMessage [payload=0, headers={sequenceNumber=1, correlationId=561e08a2-e41b-3a21-94d0-aeb106c75ff9, id=93b0fa93-fafc-088b-56d1-65b3e71df040, sequenceSize=2, timestamp=1680717969937}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=0, headers={id=561e08a2-e41b-3a21-94d0-aeb106c75ff9, timestamp=1680717969935}]
[   scheduling-1] someSubFlow2                             : GenericMessage [payload=0, headers={id=561e08a2-e41b-3a21-94d0-aeb106c75ff9, timestamp=1680717969935}]
[ool-1-worker-19] sequenceSubFlow2                         : GenericMessage [payload=0, headers={sequenceNumber=2, correlationId=561e08a2-e41b-3a21-94d0-aeb106c75ff9, id=ae144811-383f-c914-0fae-a040331b241e, sequenceSize=2, timestamp=1680717969938}]
[ool-1-worker-23] executorSubFlow2                         : GenericMessage [payload=1, headers={id=443935cf-e3e9-9e78-1eef-f04c85e646ad, timestamp=1680717970937}]
[Pool-1-worker-5] executorSubFlow1                         : GenericMessage [payload=1, headers={id=443935cf-e3e9-9e78-1eef-f04c85e646ad, timestamp=1680717970937}]
[ool-1-worker-19] sequenceSubFlow1                         : GenericMessage [payload=1, headers={sequenceNumber=1, correlationId=443935cf-e3e9-9e78-1eef-f04c85e646ad, id=8e321cb1-93cf-50ff-d27a-24ac1f4d92af, sequenceSize=2, timestamp=1680717970938}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=1, headers={id=443935cf-e3e9-9e78-1eef-f04c85e646ad, timestamp=1680717970937}]
[   scheduling-1] someSubFlow2                             : GenericMessage [payload=1, headers={id=443935cf-e3e9-9e78-1eef-f04c85e646ad, timestamp=1680717970937}]
[ool-1-worker-19] sequenceSubFlow2                         : GenericMessage [payload=1, headers={sequenceNumber=2, correlationId=443935cf-e3e9-9e78-1eef-f04c85e646ad, id=9b139d7e-2728-f838-515a-4bfefb6eb872, sequenceSize=2, timestamp=1680717970938}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=2, headers={id=e0972c11-f855-bb4c-ba68-5a531fa9b253, timestamp=1680717971938}]
[   scheduling-1] someSubFlow2                             : GenericMessage [payload=2, headers={id=e0972c11-f855-bb4c-ba68-5a531fa9b253, timestamp=1680717971938}]
[Pool-1-worker-5] executorSubFlow1                         : GenericMessage [payload=2, headers={id=e0972c11-f855-bb4c-ba68-5a531fa9b253, timestamp=1680717971938}]
[ool-1-worker-23] executorSubFlow2                         : GenericMessage [payload=2, headers={id=e0972c11-f855-bb4c-ba68-5a531fa9b253, timestamp=1680717971938}]
[ool-1-worker-19] sequenceSubFlow1                         : GenericMessage [payload=2, headers={sequenceNumber=1, correlationId=e0972c11-f855-bb4c-ba68-5a531fa9b253, id=f26025ca-504a-53a4-8e50-273e3551e489, sequenceSize=2, timestamp=1680717971938}]
[ool-1-worker-19] sequenceSubFlow2                         : GenericMessage [payload=2, headers={sequenceNumber=2, correlationId=e0972c11-f855-bb4c-ba68-5a531fa9b253, id=91cc8889-1d3d-1e00-4b7b-5ca35a167591, sequenceSize=2, timestamp=1680717971940}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=3, headers={id=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, timestamp=1680717972934}]
[Pool-1-worker-5] executorSubFlow2                         : GenericMessage [payload=3, headers={id=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, timestamp=1680717972934}]
[ool-1-worker-23] executorSubFlow1                         : GenericMessage [payload=3, headers={id=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, timestamp=1680717972934}]
[ool-1-worker-19] sequenceSubFlow1                         : GenericMessage [payload=3, headers={sequenceNumber=1, correlationId=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, id=123dc305-ecd3-48a6-2a98-61284c1d72d3, sequenceSize=2, timestamp=1680717972934}]
[   scheduling-1] someSubFlow2                             : GenericMessage [payload=3, headers={id=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, timestamp=1680717972934}]
[ool-1-worker-19] sequenceSubFlow2                         : GenericMessage [payload=3, headers={sequenceNumber=2, correlationId=e8a6e844-e1ab-79fd-59d7-c969351bb0f5, id=e0932f83-cd1b-2c0d-9c54-7e41923d8231, sequenceSize=2, timestamp=1680717972935}]
```  

`PublishSubscribeChannel` 을 사용 했기 때문에 `mainFlow` 를 통해 전달되는 메시지가 모든 하위 플로우인 컨슈머에게 함께 전달되는 것을 확인 할 수 있다.  

- `somePubSubChannel` : 기본 생성자로 생성한 채널이다. `scheduling` 스레드를 통해 메시지가 하위 구독자 들에게 전달 된다. 
- `executorPubSubChannel` : `Executor` 를 생성자에 전달해 생성한 채널이다. 전달된 `Executor` 의 스레드 풀을 통해 메시지가 전달된다. 
- `sequencePubSubChannel` : `applySequence` 를 `true` 로 설정한 채널이다. 로그에서 `headers` 부분을 보면 다른 채널들과 달리 `sequenceId`, `sequenceNumber`, `correlationId` 가 존재하는 것을 알 수 있다. `applySequence` 가 `false` 일 때는 매번 완전히 동일한 메시지가 전달 되지만 `true` 인 경우에는 `id` 값이 매번 달라지는 것으 확인 할 수 있다. 


### QueueChannel
`QueueChannel` 은 `point-to-point` 방식의 채널이기 때문에, 여러 컨슈머가 있더라도 하나의 컨슈머만 메시지 수신이 가능하다.
만약 다수라면 번갈아가며 메시지를 받아가게 된다. 
채널을 생성할 때 `capacity` 를 지정할 수 있는데 초기 값은 `Integer.MAX_VALUE` 로 사실상 무제한에 가깝다. 
채널로 전달된 메시지는 모두 `Queue` 에 저장되는 방식이다.  

```java
public QueueChannel() {
    this(new LinkedBlockingQueue<>());
}
```  


- `send(Message)` 
  - 용량 여유가 있는 경우 : 컨슈머가 없더라도 `send` 를 통해 메시지를 즉시 반환한다. 
  - 용량이 다 찬 경우 : 여유가 생길 때까지 `sender` 를 블로킹한다. 타임아웃이 전달 된 경우는 타임아웃 시간까지만 블로킹한다. 
- `receive()` 
  - 메시지가 있는 경우 : 메시지가 있다면 즉시 반환한다. 
  - 메시지가 없는 경우 : 비어 있다면 `receive` 가 블로킹된다. 만약 타임아웃이 전달된다면 타임아웃 시간까지만 블로킹된다.

`send`, `receive` 모두 타임아웃을 설정할 수 있는데, `0` 을 전달하면 블로킹 없이 바로 반환될 것이다. 
하지만 타임아웃 없이 호출하는 경우에는 무한정 블로킹상태에 빠질 수 있으므로 주의해야 한다.  

```java
@Configuration
public class QueueChannelConfig {
    @Bean
    public MessageChannel someQueueChannel() {
        return new QueueChannel(2);
    }

    @Bean
    public IntegrationFlow mainFlow() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                counter::getAndIncrement,
                poller -> poller.poller(Pollers.fixedRate(500))
        )
                .channel("someQueueChannel")
                .get();
    }

    @Bean
    public IntegrationFlow someSubFlow1() {
        return IntegrationFlows.from("someQueueChannel")
                .log("someSubFlow1")
                .get();
    }
}
```  

```
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=0, headers={id=63f99408-7480-3e89-8f49-25dad3f1fb71, timestamp=1680723346643}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=1, headers={id=0fdfb408-9c05-7638-fd1c-34cf8f8c2184, timestamp=1680723347651}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=2, headers={id=69ac8a0a-e5eb-58f1-9be2-a01b8a830848, timestamp=1680723347651}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=3, headers={id=b68095b2-4315-8f5e-db92-010ebd7bdbff, timestamp=1680723348667}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=4, headers={id=3245a2e5-d96b-af27-84af-cfa8a877591e, timestamp=1680723348667}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=5, headers={id=546c0688-f769-ca6f-915e-79e662227722, timestamp=1680723349685}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=6, headers={id=56e13079-43ad-006f-59cf-ae32f137613b, timestamp=1680723349685}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=7, headers={id=f4ebdb97-3509-13ff-ef4d-3e4fdb9ecc0b, timestamp=1680723350702}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=8, headers={id=730c93ce-c8cc-bc7f-cba8-1303fd06368d, timestamp=1680723350702}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=9, headers={id=e413ec41-6e2f-8027-a9cc-15349a217c9b, timestamp=1680723351718}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=10, headers={id=a9fc025a-4319-9863-b1ba-e7a2b6c65c64, timestamp=1680723351718}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=11, headers={id=3847bceb-85ba-4185-d73c-4008a59b0eb6, timestamp=1680723352734}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=12, headers={id=1fdf49a4-44a6-c386-84fc-dd10de0eb6a9, timestamp=1680723352734}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=13, headers={id=47ccaab6-5564-bf06-76df-53cd14bd0d33, timestamp=1680723353750}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=14, headers={id=7c349ed7-8cae-b5bf-e3ce-43e8879387d3, timestamp=1680723353750}]
[   scheduling-1] someSubFlow1                             : GenericMessage [payload=15, headers={id=2c7106c5-29cd-743b-e44c-5c069643adbd, timestamp=1680723354768}]
```  

추가적으로 `QueueChannel` 은 큐를 통해 전달된 메시지를 저장해놓고 순서대로 컨슈머에게 전달하는 방식이기 때문에 메시지 유실 위험이 있다. 
기본적으로 인메모리를 통해 큐를 관리하기 때문이다. 
이러한 위험성을 극복하기 위해 `MessageGroupStore` 를 기반으로하는 `Persistent QueueChannel` 을 생성 할 수 있다. (이때는 `capacity` 설정이 불가하다.)
`MessageGroupStore` 와 `MessageStore` 에 대한 자세한 설명은 [여기](https://docs.spring.io/spring-integration/docs/current/reference/html/message-store.html#message-store)
에서 확인 가능하다.  

말그대로 `QueueChannel` 이 메시지를 받으면 등록된 외부 저장소에 메시지를 추가하고, 
그리고 `QueueChannel` 에서 메시지를 폴링하면 외부 저장소에서는 삭제된다. 

```java
@Bean
public MessageChannel inputChannel(BasicMessageGroupStore mongoDbChannelMessageStore) {
    return new QueueChannel(new MessageGroupQueue(mongoDbChannelMessageStore, "inputChannel"));
}

@Bean
public BasicMessageGroupStore mongoDbChannelMessageStore(MongoDbFactory mongoDbFactory) {
        MongoDbChannelMessageStore store = new MongoDbChannelMessageStore(mongoDbFactory);
        store.setPriorityEnabled(true);
        return store;
}
```  

### PriorityChannel
`PriorityChannel` 은 `QueueChannel` 의 방식인 `FIFO` 와는 달리 채널 내에 우선순위에 따른 정렬이 가능하다. 
우선순위에 대한 별다른 설정을 하지 않으면 기본으로 헤더의 `priority` 값을 기준으로 정해진다. 
별도로 우선순위를 정하고 싶다면 `PriorityChannel` 생성때 `Comparator<Message<?>>` 를 함께 전달해 주면 된다. 

아래는 예시는 2개의 `PriorityQueue` 가 있는데 설명은 아래와 같다.

- `somePriorityQueueChannel` : 기본인 헤더값의 `priority` 값을 우선순위 기준값으로 정한다. 
- `reversePriorityChannel` : 메시지 `payload` 의 값을 작은 수가 우선순위를 갖는 오름차순으로 정렬한다. 

```java
@Configuration
public class PriorityChannelConfig {
    @Bean
    public MessageChannel somePriorityChannel() {
        return new PriorityChannel(100);
    }

    @Bean
    public MessageChannel reversePriorityChannel() {
        return new PriorityChannel(100, (o1, o2) -> {
            Integer p1 = (Integer) o1.getPayload();
            Integer p2 = (Integer) o2.getPayload();

            return p1.compareTo(p2);
        });
    }

    @Bean
    public IntegrationFlow mainFlow() {
        AtomicInteger counter = new AtomicInteger();
        return IntegrationFlows.fromSupplier(
                        () -> {
                            int result = counter.getAndIncrement();
                            return MessageBuilder
                                    .withPayload(result)
                                    .setHeader("priority", result % 2)
                                    .build();
                        },
                        poller -> poller.poller(Pollers.fixedRate(50))
                )
                .channel("somePriorityChannel")
                .get();
    }

    @Bean
    public IntegrationFlow someSubFlow() {
        return IntegrationFlows.from("somePriorityChannel")
                .log("someSubFlow")
                .channel("reversePriorityChannel")
                .get();
    }

    @Bean
    public IntegrationFlow reverseSubFlow() {
        return IntegrationFlows.from("reversePriorityChannel")
                .log("reverseSubFlow")
                .get();
    }
}
```  

`mainFlow` 에서는 `0, 1, 2, 3, ..` 과 같이 전달되는 메시지에서 짝수이면 `priority` 를 `0` 으로
홀수면 `priority` 를 `1` 로 헤더에 설정한다. 
그리고 해당 메시지는 `somePriorityChannel` 을 타고 `someSubFlow` 로 전달되는데 이떄 로그는 홀수가 더 높은 우선순위를 가지므로 먼저 출력된다.  
이후에는 `reversePriorityChannel` 을 타고 `reverseSubFlow` 로 전달되면서 로그는 오름차순으로 낮은 수가 먼저 로그에 찍히는 것을 확인 할 수 있다.  


```
.. priority 우선순위로 홀수가 먼저 출력 ..
someSubFlow                              : GenericMessage [payload=3, headers={priority=1, id=4d68b33b-fb11-7762-e3e5-ddb8209546d7, timestamp=1680940649074}]
someSubFlow                              : GenericMessage [payload=2, headers={priority=0, id=2a8b557e-5166-b8f9-bfd5-2ea0fbff3488, timestamp=1680940648067}]
.. 낮은 값이 우선순위를 갖으므로 낮은 수가 먼저 출력 ..
reverseSubFlow                           : GenericMessage [payload=2, headers={priority=0, id=2a8b557e-5166-b8f9-bfd5-2ea0fbff3488, timestamp=1680940648067}]
reverseSubFlow                           : GenericMessage [payload=3, headers={priority=1, id=4d68b33b-fb11-7762-e3e5-ddb8209546d7, timestamp=1680940649074}]
someSubFlow                              : GenericMessage [payload=5, headers={priority=1, id=8f11be6b-6f38-d9aa-26d7-4462ee51e466, timestamp=1680940651085}]
someSubFlow                              : GenericMessage [payload=4, headers={priority=0, id=a1262185-1844-6b4c-4e04-b6ad7ebe5d73, timestamp=1680940650079}]
reverseSubFlow                           : GenericMessage [payload=4, headers={priority=0, id=a1262185-1844-6b4c-4e04-b6ad7ebe5d73, timestamp=1680940650079}]
reverseSubFlow                           : GenericMessage [payload=5, headers={priority=1, id=8f11be6b-6f38-d9aa-26d7-4462ee51e466, timestamp=1680940651085}]
someSubFlow                              : GenericMessage [payload=7, headers={priority=1, id=15d0623b-223c-9a62-5e88-fe5f25306716, timestamp=1680940653090}]
someSubFlow                              : GenericMessage [payload=6, headers={priority=0, id=df642219-b596-279b-1a20-ee1318ae91a6, timestamp=1680940652085}]
reverseSubFlow                           : GenericMessage [payload=6, headers={priority=0, id=df642219-b596-279b-1a20-ee1318ae91a6, timestamp=1680940652085}]
reverseSubFlow                           : GenericMessage [payload=7, headers={priority=1, id=15d0623b-223c-9a62-5e88-fe5f25306716, timestamp=1680940653090}]
someSubFlow                              : GenericMessage [payload=9, headers={priority=1, id=1327a119-074f-3324-2bd3-030e1e1c949c, timestamp=1680940655104}]
someSubFlow                              : GenericMessage [payload=8, headers={priority=0, id=174c3e90-418a-f709-f3a8-5091b8d01175, timestamp=1680940654098}]
reverseSubFlow                           : GenericMessage [payload=8, headers={priority=0, id=174c3e90-418a-f709-f3a8-5091b8d01175, timestamp=1680940654098}]
reverseSubFlow                           : GenericMessage [payload=9, headers={priority=1, id=1327a119-074f-3324-2bd3-030e1e1c949c, timestamp=1680940655104}]
someSubFlow                              : GenericMessage [payload=11, headers={priority=1, id=21004f5e-a413-9569-3478-bd5de7f6124a, timestamp=1680940657111}]
someSubFlow                              : GenericMessage [payload=10, headers={priority=0, id=9f6af242-279a-579b-6c5d-b847c3f99ab5, timestamp=1680940656109}]
reverseSubFlow                           : GenericMessage [payload=10, headers={priority=0, id=9f6af242-279a-579b-6c5d-b847c3f99ab5, timestamp=1680940656109}]
reverseSubFlow                           : GenericMessage [payload=11, headers={priority=1, id=21004f5e-a413-9569-3478-bd5de7f6124a, timestamp=1680940657111}]
```  

### RendezvousChannel













---  
## Reference
[Messaging Channels](https://docs.spring.io/spring-integration/docs/current/reference/html/core.html#channel)  
[Message Store](https://docs.spring.io/spring-integration/docs/current/reference/html/message-store.html#message-store)  
