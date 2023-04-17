--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Integration MessageChannel"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Integration 에서 사용할 수 있는 MessageChannel 의 종류와 특징 그리고 구성 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Integration
    - Java DSL
    - MessageChannel
    - PollableChannel
    - SubscribableChannel
    - PublishSubscribeChannel
    - QueueChannel
    - PriorityChannel
    - RendezvousChannel
    - DirectChannel
    - ExecutorChannel
    - FluxMessageChannel
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
`RendezvousChannel`(랑데뷰 채널)은 해당 채널의 `receive()` 메소드를 호출할 때까지
`sender` 스레드를 블리킹하는 `direct-handoff` 방식의 채널이다. 
해당 채널은 `SynchronousQueue` 를 사용한다는 점만 빼면 기존 `QueueChannel` 과 유시하다. 
`sender` 와 `receiver` 가 서로 다른 스레드에서 동작하면서 큐에 전달된 메시지가 비동기로 삭제되는 
처리가 적절하지 않는, 즉 `QueueChannel` 에 저장된 메시지가 어느 하나의 `receiver` 에는 꼭 전달되는 것이
보장되는 상황에서 사용할 수 있다.  

`RendezvousChannel` 은 `QueueChannel` 의 동기화 버전으로, 
`request-reply` 동작 방식을 구현할 때 유용하다.  

예제는 `sender` 와 `receiver` 를 서로 다른 스레드에서 동작 시키고, 
`sedner` 는 코드상으론 대기없이 메시지를 전송하려 하자만, `receiver` 는 `1000ms` 마다 메시지를 수신하기 때문에 
실제로 메시지는 `RendezvousChannel` 의 `receive()` 를 호출하는 `receiver` 의 주기인 `1000ms` 마다 
전달되는 것을 확인 할 수 있다.  

```java
@Slf4j
@Configuration
public class RendezvousChannelConfig {
    @Bean
    public RendezvousChannel someRendezvousChannel() {
        return new RendezvousChannel();
    }

    @Bean
    public Thread sender() {
        Thread thread = new Thread(() -> {
            for(int i = 0; i < 100; i++) {
                log.info("wait send message {}", i);
                this.someRendezvousChannel().send(MessageBuilder.withPayload(i).build());
                log.info("done send message {}", i);
            }
        });

        thread.start();

        return thread;
    }

    @Bean
    public Thread receiver() {
        Thread thread = new Thread(() -> {
            while(true) {
                try {
                    Message<?> msg = this.someRendezvousChannel().receive();
                    log.info("receive message : {}", msg);
                    Thread.sleep(1000);
                } catch (Exception e) {

                }
            }
        });

        thread.start();

        return thread;
    }
}
```  

```
17:03:23.160  INFO 92836 --- [Thread-1] test: wait send message 0
17:03:23.163  INFO 92836 --- [Thread-1] test: done send message 0
17:03:23.163  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=0, headers={id=ac1f36cb-a493-a7e9-51a4-5c3092cc4cf7, timestamp=1681545803163}]
17:03:23.163  INFO 92836 --- [Thread-1] test: wait send message 1
17:03:24.167  INFO 92836 --- [Thread-1] test: done send message 1
17:03:24.167  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=1, headers={id=e5aa3344-dbd4-5738-0bb7-12877d118a3b, timestamp=1681545803163}]
17:03:24.168  INFO 92836 --- [Thread-1] test: wait send message 2
17:03:25.168  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=2, headers={id=98fe9953-a419-0c27-4961-e2fd0543e398, timestamp=1681545804168}]
17:03:25.168  INFO 92836 --- [Thread-1] test: done send message 2
17:03:25.168  INFO 92836 --- [Thread-1] test: wait send message 3
17:03:26.173  INFO 92836 --- [Thread-1] test: done send message 3
17:03:26.173  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=3, headers={id=dbf0f244-f800-903d-39ce-c235cbc185fd, timestamp=1681545805168}]
17:03:26.174  INFO 92836 --- [Thread-1] test: wait send message 4
17:03:27.179  INFO 92836 --- [Thread-1] test: done send message 4
17:03:27.179  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=4, headers={id=a0947cf6-16f5-a5c3-e3b9-90b52717810b, timestamp=1681545806174}]
17:03:27.179  INFO 92836 --- [Thread-1] test: wait send message 5
17:03:28.184  INFO 92836 --- [Thread-1] test: done send message 5
17:03:28.184  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=5, headers={id=3bc39d7e-b850-e224-4600-bab5833f9d20, timestamp=1681545807179}]
17:03:28.185  INFO 92836 --- [Thread-1] test: wait send message 6
17:03:29.191  INFO 92836 --- [Thread-1] test: done send message 6
17:03:29.191  INFO 92836 --- [Thread-1] test: wait send message 7
17:03:29.190  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=6, headers={id=213a5b66-8623-f775-c07a-f0a6e6c59a33, timestamp=1681545808185}]
17:03:30.196  INFO 92836 --- [Thread-1] test: done send message 7
17:03:30.196  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=7, headers={id=f250da3a-cf49-931c-b2cf-620dcf8420d9, timestamp=1681545809191}]
17:03:30.196  INFO 92836 --- [Thread-1] test: wait send message 8
17:03:31.201  INFO 92836 --- [Thread-2] test: receive message : GenericMessage [payload=8, headers={id=b88b4add-76cb-ee7d-6318-c01044208482, timestamp=1681545810196}]
17:03:31.201  INFO 92836 --- [Thread-1] test: done send message 8
```  


### DirectChannel
`DirectChannel` 은 `Spring Integration` 의 기본 `MessgaeChannel` 이면서 `point-to-point` 방식과 `publish-subscirbe` 채널의 특징을 복합적으로 가지고 있다. 
`DirectChannel` 은 `PollableChannel` 이 아닌 `SubscribableChannel` 인터페이스를 구현하고 있기 때문에 `SubscribableChannel` 에 더 유사하다고 할 수 있다. 
그러므로 자신의 채널을 구독하는 구독자에게 메시지를 직접 발송하지만, `point-to-point` 특징도 가지고 있기 때문에 
단일 구독자인 `MessageHandler` 에게만 `message` 를 전송한다는 점에서는 일반적인 `PublishSubscirbeChannel` 과는 차이점이 있다.  

`DirectChannel` 의 특징중 하나는 입력/출력이 단일 스레드에서 채널의 약쪽 작업을 모두 수행할 수 있다는 것이다. 
특정 구독자가 `DirectChannel` 을 구독하고 있는 상황에서 `sender` 가 메시지를 전송하면 `send()` 메소드가 
값을 반환하기 전 `handleMessage(Message)` 를 직접 호출해 메시지가 구독자에게 전달되기 때문에 
이는 `sender` 의 스레드에서 메시지의 송/수신이 수행된다.  

이러한 특징의 주된 요인은 채널을 사용할 때의 장점인 `loose coupling` 은 그대로 가지면서, 
채널 전체에 커버할 수 있는 트랜잭션을 지원하기 위함이다. 
예를 들어 `send()` 메소드를 통해 전달되는 메시지는 `handleMessage()` 메소드를 거쳐 
구독자에게 전달되고 수신 처리의 완료에 대한 트랜잭션의 최정 결과가 결정될 수 있다.  

앞서 언급한 것처럼 `DirectChannel` 은 `SubscribableChannel` 의 구현체 이기 때문에, 
여러 구독자가 하나의 채널을 구독 할 순 있지만, 실제 메시지는 단일 구독자에게만 전달 된다. 
이때 해당 채널이 어느 구독자에게 메시지를 전달할지는 `MessageDispatcher` 에게 위임한다. 
디스패처는 `load-balancer` 혹은 `load-balancer-ref`(두 필드는 함꼐 사용 될 수 없다.) 에 설정된 속성을 바탕으로 로드 밸런싱 전략이 결정된다. 
이런 로드 밸런싱은 `LoadBalancingStrategy` 의 구현체로 기본 제공되는 것으로는 `round-robin`(기본) 과 `none` 이 있다. 

또한 로드 밸런싱에 대한 동작은 `failover` 전략에 따라서 달라질 수 있다. 
만약 `true`(기본값) 인 경우 먼저 선택된 메시지 핸들러가(구독자) 예외를 던지게 되면, 그 다음 메시지 핸들러에게 다시 메시지를 전송한다. 
만약 로드 밸런싱이 `none` 인 상태라도 `failover` 가 `true` 이면 예외 발생시 메시지는 다른 구독자에게 전달 될 수 있다. 
이러한 메시지 핸들러의 선택은 `order` 속성을 통해 우선순위를 결정 할 수 있다.  

아래 예제는 2개의 채널이 있다.

- `roundrobinDirectChannel` : 기본 로드 밸런싱 동작인 `round-robin` 으로 설정된 채널로 여러 구독자가 있는 경우 번갈아가며 구독자에게 메시지를 전달한다. 
- `noneDirectChannel` : 로드 밸런싱 동작을 비활성화한 채널로 `failover` 가 동작 할 떄만 다른 채널로 메시지를 전달한다.  

```java
@Configuration
public class DirectChannelConfig {
    @Bean
    public MessageChannel roundrobinDirectChannel() {
        DirectChannel directChannel = new DirectCHannel();
        // failover 비활성화
        // directChannel.setFailover(false);
        return directChannel;
    }

    @Bean
    public MessageChannel noneDirectChannel() {
        return new DirectChannel(null);
    }

    @Bean
    public IntegrationFlow mainFlow() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("roundrobinDirectChannel")
                .get();
    }

    @Bean
    public IntegrationFlow subFlow1() {
        return IntegrationFlows.from("roundrobinDirectChannel")
                .log("subFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow subFlow2() {
        return IntegrationFlows.from("roundrobinDirectChannel")
                .log("subFlow2")
                .channel("noneDirectChannel")
                .get();
    }

    @Bean
    public IntegrationFlow noneFlow1() {
        return IntegrationFlows.from("noneDirectChannel")
                .handle(Integer.class, (payload, headers) -> {
                    if(payload % 3 == 0) {
                        throw new RuntimeException("noneFlow1 Exception");
                    }

                    return payload;
                })
                .log("noneFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow noneFlow2() {
        return IntegrationFlows.from("noneDirectChannel")
                .log("noneFlow2")
                .handle(message -> {
                    try {
                        Thread.sleep(3000);
                    } catch (Exception ignore) {

                    }
                })
                .get();
    }
}
```  

각 플로우에 대한 설명과 수신할 메시지는 아래와 같다. 

- `subFlow1` : `roundrobinDirectChannel` 에서 0부터 짝수의 값을 수신한다. 
- `subFlow2` : `roundrobinDirectChannel` 에서 1부터 홀수의 값을 수신하고 이를 다시 `noneDirectChannel` 로 전달 한다. 
- `noneFlow1` : `noneDirectChannel` 로 부터 `subFlow2` 가 수신한 메시지를 전달 받는 플로우이다. 수신 값이 3의 배수인 경우에는 예외를 발생시킨다. 
- `noneFlow2` : `noneDirectChannel` 로 부터 `subFlow2` 가 수신한 메지시 중 `noneFlow1` 으로 전달 됐지만, `failover` 동작으로 홀수 중 3의 배수의 값만 수신된다.  

```
.. 500ms 마다 메시지 전송 ..
17:53:44.566  subFlow1                                 : GenericMessage [payload=0, headers={id=51f447a4-a0c3-df89-dd7b-168ba156629a, timestamp=1681547775961}]
17:53:45.068  subFlow2                                 : GenericMessage [payload=1, headers={id=22382474-1d2c-e23e-0f03-d5145673c482, timestamp=1681547776464}]
17:53:45.068  noneFlow1                                : GenericMessage [payload=1, headers={id=d8dafd7d-3aa3-2f9b-a4a0-011719b532e4, timestamp=1681547776464}]
17:53:45.571  subFlow1                                 : GenericMessage [payload=2, headers={id=f0783a03-9c70-e313-97ea-01c16c55df62, timestamp=1681547776960}]
17:53:46.065  subFlow2                                 : GenericMessage [payload=3, headers={id=ccf427c3-25c9-c727-9b01-73f78074d8fe, timestamp=1681547777462}]
17:53:46.066  o.s.i.dispatcher.UnicastingDispatcher    : An exception was thrown by 'ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@70e889e9] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)' while handling 'GenericMessage [payload=3, headers={id=ccf427c3-25c9-c727-9b01-73f78074d8fe, timestamp=1681547777462}]': error occurred in message handler [ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@70e889e9] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)]; nested exception is java.lang.RuntimeException: noneFlow1 Exception. Failing over to the next subscriber.
17:53:46.066  noneFlow2                                : GenericMessage [payload=3, headers={id=ccf427c3-25c9-c727-9b01-73f78074d8fe, timestamp=1681547777462}]
.. noneFlow2 의 동작으로 3초 후 메시지 전송 ..
.. 다시 500ms 마다 메시지 전송 ..
17:53:49.072  subFlow1                                 : GenericMessage [payload=4, headers={id=a3293f71-e238-7337-2a7f-021cb46fbbe3, timestamp=1681547777964}]
17:53:49.073  subFlow2                                 : GenericMessage [payload=5, headers={id=b959ea7d-2847-b8f8-ddb0-9f0a5b9c68fb, timestamp=1681547778464}]
17:53:49.073  noneFlow1                                : GenericMessage [payload=5, headers={id=cfcb872b-f9b8-2881-feb4-346dc26af193, timestamp=1681547778464}]
17:53:49.073  subFlow1                                 : GenericMessage [payload=6, headers={id=db4bf4e7-ab15-a093-5b4c-fcb1703517a4, timestamp=1681547778962}]
17:53:49.073  subFlow2                                 : GenericMessage [payload=7, headers={id=fc82706f-cb51-df35-5c75-82dcaf4051a2, timestamp=1681547779463}]
17:53:49.074  noneFlow1                                : GenericMessage [payload=7, headers={id=a75a6fc3-3ec4-7dbf-5824-ab934baeb44c, timestamp=1681547779463}]
17:53:49.074  subFlow1                                 : GenericMessage [payload=8, headers={id=26df0a5e-a9c9-1121-23ad-fb4899698104, timestamp=1681547779963}]
17:53:49.074  subFlow2                                 : GenericMessage [payload=9, headers={id=4355c0d4-12ba-b543-ebb1-1804ba4fc39a, timestamp=1681547780464}]
17:53:49.074  o.s.i.dispatcher.UnicastingDispatcher    : An exception was thrown by 'ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@70e889e9] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)' while handling 'GenericMessage [payload=9, headers={id=4355c0d4-12ba-b543-ebb1-1804ba4fc39a, timestamp=1681547780464}]': error occurred in message handler [ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@70e889e9] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)]; nested exception is java.lang.RuntimeException: noneFlow1 Exception. Failing over to the next subscriber.
17:53:49.074  noneFlow2                                : GenericMessage [payload=9, headers={id=4355c0d4-12ba-b543-ebb1-1804ba4fc39a, timestamp=1681547780464}]
.. noneFlow2 의 동작으로 3초 후 메시지 전송 ..
17:53:52.075  subFlow1                                 : GenericMessage [payload=10, headers={id=6e7eb1a0-d2e1-4b95-211c-dfa695d55cd9, timestamp=1681548832075}]
```  

`noneFlow2` 의 `sleep(3000)` 의 동작과 메시지 플로우가 실행되며 남긴 로그를 잘 살펴보자. 
`noneFlow2` 가 실행되지 않는 경우에 메시지는 생산 주기인 `500ms` 마다 정상적으로 전달되는 것을 확인 할 수 있다. 
하지만 `noneFlow2` 가 실행되는 홀수 이면서 3의 배수인 경우에는 `3000ms` 후 다음 메시지가 전달된다. 
이는 `DirectChannel` 이 메시지 `send-receive` 전체에 대해서 트랜잭선을 제공하는 방식을 간접적으로 보여주는 동작이다. 
전체 플로우는 `noneFlow2` 의 `sleep(3000)` 가 완료 될때까지 기다리고 있다. 


### ExecutorChannel
`ExecutorChannel` 은 `DirectChannel` 과 아주 유사한 채널로 동일한 디스패처 설정(로드 밸런싱, failover)을 지원하는 `point-to-point` 채널이다. 
`DirectChannel` 과 차이점은 `TaskExecutor` 에게 디스패처 수행을 위임한다는 점에 있다. 
이런 `TaskExecutor` 의 사용으로 `DirectChannel` 은 `send()` 메소드가 구독자에게 데이터를 전달하는 `handleMessage()` 메소드 호출 과정에서 블로킹 될 수 있지만, 
`ExecutorChannel` 은 `send()` 메소드가 블로킹 되지 않는다. 
이는 다시 말하면, `sender` 와 `receiver` 가 각기 다른 스레드에서 수행되고 이러한 특징으로 `DirectChannel` 처럼 트랜잭션 기능은 지원되지 않는다.  

예제는 앞서 진행한 `DirectChannel` 과 동일한다. 
`DirectChannel` 의 예제코드에서 채널만 `ExecutorChannel` 로 변경했다. 
로그밸런싱, `failover` 모두 `DirectChannel` 과 동일하게 동작한다.  

```java
@Configuration
public class ExecutorChannelConfig {
    @Bean
    public MessageChannel roundrobinExecutorChannel() {
        ExecutorChannel executorChannel = new ExecutorChannel(Executors.newWorkStealingPool());
        // failover 비활성화
        // executorChannel.setFailover(false);
        return executorChannel;
    }

    @Bean
    public MessageChannel noneExecutorChannel() {
        return new ExecutorChannel(Executors.newWorkStealingPool());
    }


    @Bean
    public IntegrationFlow mainFlow() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("roundrobinExecutorChannel")
                .get();
    }

    @Bean
    public IntegrationFlow subFlow1() {
        return IntegrationFlows.from("roundrobinExecutorChannel")
                .log("subFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow subFlow2() {
        return IntegrationFlows.from("roundrobinExecutorChannel")
                .log("subFlow2")
                .channel("noneExecutorChannel")
                .get();
    }

    @Bean
    public IntegrationFlow noneFlow1() {
        return IntegrationFlows.from("noneExecutorChannel")
                .handle(Integer.class, (payload, headers) -> {
                    if(payload % 3 == 0) {
                        throw new RuntimeException("noneFlow1 Exception");
                    }

                    return payload;
                })
                .log("noneFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow noneFlow2() {
        return IntegrationFlows.from("noneExecutorChannel")
                .log("noneFlow2")
                .handle(message -> {
                    try {
                        Thread.sleep(3000);
                    } catch (Exception ignore) {

                    }
                })
                .get();
    }
}
```  

가장 큰 차이점은 `DirectChannel` 은 `noneFlow2` 로 인해 특정 주기마다 `sleep(3000)` 동작으로 다음 메시지가 늦게 전달되는 현상이 있었다. 
하지만 `ExecutorChannel` 은 `sender` 와 `receiver` 가 서로 다른 스레드에서 동작하므로 `noneFlow2` 가 실행되는 메시지더라도, 
정상적으로 모두 `sleep(3000)` 으로 인한 대기 없이 메시지가 `500ms` 마다 전달된다.  


```
.. sender와 receiver 가 서로 다른 스레드에서 동작하므로 500ms 메시지 전달 
18:10:09.490  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=0, headers={id=932d0fa8-ab3b-c1ef-0be4-896b27afb57d, timestamp=1681549809488}]
18:10:09.992  [ool-1-worker-19] subFlow2                                 : GenericMessage [payload=1, headers={id=850ea570-b148-0798-c534-6a480312901a, timestamp=1681549809992}]
18:10:09.992  [ool-2-worker-19] noneFlow1                                : GenericMessage [payload=1, headers={id=02e631a0-99b1-cdb8-bdbf-cc23374eb6e3, timestamp=1681549809992}]
18:10:10.491  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=2, headers={id=688e0745-11d0-e8b6-7158-6f1899795f12, timestamp=1681549810491}]
18:10:10.991  [ool-1-worker-19] subFlow2                                 : GenericMessage [payload=3, headers={id=692b54af-0568-647a-60fc-fe1505acfb77, timestamp=1681549810991}]
18:10:10.991  [ool-2-worker-19] noneFlow2                                : GenericMessage [payload=3, headers={id=692b54af-0568-647a-60fc-fe1505acfb77, timestamp=1681549810991}]
18:10:11.489  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=4, headers={id=b5277a7e-4465-da24-dee9-977da62fb018, timestamp=1681549811489}]
18:10:11.994  [ool-1-worker-19] subFlow2                                 : GenericMessage [payload=5, headers={id=f8d9fd71-634f-c531-28d0-2f8774f12f6b, timestamp=1681549811994}]
18:10:11.995  [Pool-2-worker-5] noneFlow1                                : GenericMessage [payload=5, headers={id=0705743f-e197-2b1c-b76c-eaf4fcf58b9a, timestamp=1681549811995}]
18:10:12.490  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=6, headers={id=55a7db7d-8c26-2f1b-d829-8a7867d60f6e, timestamp=1681549812490}]
18:10:12.991  [ool-1-worker-19] subFlow2                                 : GenericMessage [payload=7, headers={id=abcd802e-22c8-de89-7c92-3348b489fad9, timestamp=1681549812991}]
18:10:12.992  [Pool-2-worker-5] noneFlow2                                : GenericMessage [payload=7, headers={id=abcd802e-22c8-de89-7c92-3348b489fad9, timestamp=1681549812991}]
18:10:13.492  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=8, headers={id=185d2de3-3a63-5dc4-4caf-d3fb47fe2362, timestamp=1681549813492}]
18:10:13.991  [ool-1-worker-19] subFlow2                                 : GenericMessage [payload=9, headers={id=25084884-6417-4d77-00a1-e688a847a4d9, timestamp=1681549813990}]
18:10:13.994  [ool-2-worker-23] o.s.i.dispatcher.UnicastingDispatcher    : An exception was thrown by 'ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@7b4acdc2] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)' while handling 'GenericMessage [payload=9, headers={id=25084884-6417-4d77-00a1-e688a847a4d9, timestamp=1681549813990}]': error occurred in message handler [ServiceActivator for [org.springframework.integration.handler.LambdaMessageProcessor@7b4acdc2] (noneFlow1.org.springframework.integration.config.ConsumerEndpointFactoryBean#0)]; nested exception is java.lang.RuntimeException: noneFlow1 Exception. Failing over to the next subscriber.
18:10:13.995  [ool-2-worker-23] noneFlow2                                : GenericMessage [payload=9, headers={id=25084884-6417-4d77-00a1-e688a847a4d9, timestamp=1681549813990}]
18:10:14.492  [ool-1-worker-19] subFlow1                                 : GenericMessage [payload=10, headers={id=4de3d046-8faa-aabc-f10b-5d4083649920, timestamp=1681549814491}]
```  


### FluxMessageChannel
`FluxMessageChannel` 은 다운스트에 있는 `Reactive Stream` 구독자가 `on-demand` 로 메시지를 컨숨해 
가져 살 수 있는 메시지 채널이다. 
`MessageChannel` 과` Publisher<Message<?>>` 를 모두 구현하고 있어 
또한 `FluxMessageChannel` 은 업스트림의 `Publisher` 를 `òn-demand` 로 컨슘해 갈 수 있도록
`ReactiveStreamSubscribableChannel` 인터페이스도 구현하고 있다.
그러므로 `FluxMessageChannel` 을 사용하면 `Spring Integration` 과 `Reactive Stream` 를 서로 동일한 흐름으로 연결하는 것이 가능하다.  

앞서 언급한 것처럼 `FluxMessageChannel` 은 `SubscribableChannel`, `PollableChannel` 의 구현체가 아니라 
`ReactiveStreamSubscribableChannel` 의 구현체이므로 `org.reactivestreams.Subscriber` 인터페이스만 
`Reactive Stream` 의 `back-preesure` 를 지키며 메시지 컨슘이 가능하다. 
`Spring Integration` 의 `MessageHandler` 의 구현체들은 `Reactive Stream` 의  `CoreSubscriber` 를
구현 하고 있기 때문에 큰 어려움 없이 `Spring Integration` 과 `Reactive Stream` 를 하나의 플로우로 통합 할 수 있다.  

`Spring Integration` 에서 `Reactive Stream` 을 활용하는 방법에 대한 더 자세한 내용은 
추후에 별도 포스트에서 다루도록 한다.  

아래 예제는 하나의 `FluxMessageChannel` 을 두고 `Reactive Stream` 과 `Spring Integration Flow` 를 통합하는 예제이다. 

- `fluxToIntegration` : `FluxMessageChannelsubscribeTo()` 를 사용해서 `Flux` 시퀀스를 `Spring Integration` 과 통합한다. 
- `integrationToFluxFlow1` : `FluxMessageChannel.subscirbeTo()` 과 `IntegrationFlowBuilder.toReactivePublisher()` 를 사용해서 `Spring Integration` 플로우를 `Reactive Stream` 과 통합한다. 
- `integrationToFluxFlow2` : `Spring Integration` 플로우에서 메시지를 `FluxMessageChannel` 로 전달하는 방식으로 `Reactive Stream` 과 통합한다. 
- `integrationSubFlow1` : `FluxMessageChannel` 메시지를 전달 받아 `Spring Integration` 에서 처리한다. 
- `integrationSubFlow2` : `FluxMessageChannel` 메시지를 전달 받아 `Spring Integration` 에서 처리한다. 여러 구독자에게 모든 메시지 전달이 가능하다. 
- `fluxFlow` : `Reactive Stream` 의 시퀀스로 `FluxMessageChannel` 의 내용을 `Reactive Stream` 으로 가져와 처리한다. 

```java
@Configuration
public class FluxMessageChannelConfig {
    @Bean
    public FluxMessageChannel someFluxMessageChannel() {
        FluxMessageChannel fluxMessageChannel = new FluxMessageChannel();
        fluxMessageChannel.subscribeTo(this.integrationToFluxFlow1());
        fluxMessageChannel.subscribeTo(this.fluxToIntegration());

        return fluxMessageChannel;
    }

    @Bean
    public Publisher<Message<String>> fluxToIntegration() {
        AtomicInteger counter = new AtomicInteger();
        return Flux.fromStream(Stream.generate(() -> counter.getAndIncrement() + ":flux"))
                .delayElements(Duration.ofMillis(1000))
                .publishOn(Schedulers.parallel())
                .map(s -> MessageBuilder.withPayload(s).build())
                ;
    }

    @Bean
    public Publisher<Message<Integer>> integrationToFluxFlow1() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        () -> counter.getAndIncrement() + ":integration1",
                        poller -> poller.poller(Pollers.fixedRate(1000))
                )
                .toReactivePublisher()
                ;
    }

    @Bean
    public IntegrationFlow integrationToFluxFlow2() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        () -> counter.getAndIncrement() + ":integration2",
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("someFluxMessageChannel")
                .get();
    }

    @Bean
    public IntegrationFlow integrationSubFlow1() {
        return IntegrationFlows.from("someFluxMessageChannel")
                .log("integrationSubFlow1")
                .get();
    }

    @Bean
    public IntegrationFlow integrationSubFlow2() {
        return IntegrationFlows.from("someFluxMessageChannel")
                .log("integrationSubFlow2")
                .get();
    }

    @Bean
    public Disposable fluxFlow() {
        return Flux.from(this.someFluxMessageChannel()).subscribeOn(Schedulers.boundedElastic())
                .log("fluxFlow")
                .subscribe();
    }

    // FluxMessageChannel 은 ReactiveStream 에 대한 채널로 
    // ReactiveStream 시퀀스가 수행되는 스레드와 동이랗게 daemon thread 에 해당하므로
    // 메인 스레드인 non-daemon thread 가 죽지 않도록 반복 작업을 수행해 준다. 
    @Bean
    public Thread waiter() {
        Thread thread = new Thread(() -> {
            IntStream.range(1, 100).forEach(value -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            });
        });

        thread.start();

        return thread;
    }
}
```  

`FluxMessageChannel` 로 전달되는 모든 메시지는 기본적으로 `Reactvie Stream` 의 `Schedulers.boundedElastic()` 의 스레드에서 수행된다. 
아래 실행결과를 보면 `Reactive Stream` 시퀀스에서 전달되는 메시지를 `Spring Integration` 플로우에서 처리할 수도 있고, 
반대로 `Spring Integration` 플로우에서 전달되는 메시지를 `Reactive Stream` 시퀀스에서 처리하는 등의 통합이 가능하다.  

```
[           main] fluxFlow           : onSubscribe(FluxSubscribeOn.SubscribeOnSubscriber)
[           main] fluxFlow           : request(unbounded)
        
.. integrationToFlux2 메시지 ..
[oundedElastic-5] fluxFlow           : onNext(GenericMessage [payload=0:integration2, headers={id=c63d2f65-4de5-4dad-beaf-dba8a4cfb147, timestamp=1681631095696}])
[oundedElastic-5] integrationSubFlow1: GenericMessage [payload=0:integration2, headers={id=c63d2f65-4de5-4dad-beaf-dba8a4cfb147, timestamp=1681631095696}]
[oundedElastic-5] integrationSubFlow2: GenericMessage [payload=0:integration2, headers={id=c63d2f65-4de5-4dad-beaf-dba8a4cfb147, timestamp=1681631095696}]

.. integrationToFlux2 메시지 ..
[oundedElastic-1] fluxFlow           : onNext(GenericMessage [payload=0:integration1, headers={id=04d32768-9d04-c79e-b998-6940e640a312, timestamp=1681631095696}])
[oundedElastic-1] integrationSubFlow1: GenericMessage [payload=0:integration1, headers={id=04d32768-9d04-c79e-b998-6940e640a312, timestamp=1681631095696}]
[oundedElastic-1] integrationSubFlow2: GenericMessage [payload=0:integration1, headers={id=04d32768-9d04-c79e-b998-6940e640a312, timestamp=1681631095696}]

.. fluxToIntegration 메시지 ..
[oundedElastic-2] fluxFlow           : onNext(GenericMessage [payload=0:flux, headers={id=8c57a0e2-4f92-6ddb-096d-c49f3893cac7, timestamp=1681631096631}])
[oundedElastic-2] integrationSubFlow1: GenericMessage [payload=0:flux, headers={id=8c57a0e2-4f92-6ddb-096d-c49f3893cac7, timestamp=1681631096631}]
[oundedElastic-2] integrationSubFlow2: GenericMessage [payload=0:flux, headers={id=8c57a0e2-4f92-6ddb-096d-c49f3893cac7, timestamp=1681631096631}]

.. 반복 ..
[oundedElastic-1] fluxFlow           : onNext(GenericMessage [payload=1:integration1, headers={id=1751d137-6e70-0f1b-24f0-4318a542fc3a, timestamp=1681631096694}])
[oundedElastic-1] integrationSubFlow1: GenericMessage [payload=1:integration1, headers={id=1751d137-6e70-0f1b-24f0-4318a542fc3a, timestamp=1681631096694}]
[oundedElastic-1] integrationSubFlow2: GenericMessage [payload=1:integration1, headers={id=1751d137-6e70-0f1b-24f0-4318a542fc3a, timestamp=1681631096694}]
[oundedElastic-5] fluxFlow           : onNext(GenericMessage [payload=1:integration2, headers={id=86886c7b-ae72-76cb-a661-2c1c56cf65b3, timestamp=1681631096696}])
[oundedElastic-5] integrationSubFlow1: GenericMessage [payload=1:integration2, headers={id=86886c7b-ae72-76cb-a661-2c1c56cf65b3, timestamp=1681631096696}]
[oundedElastic-5] integrationSubFlow2: GenericMessage [payload=1:integration2, headers={id=86886c7b-ae72-76cb-a661-2c1c56cf65b3, timestamp=1681631096696}]
[oundedElastic-2] fluxFlow           : onNext(GenericMessage [payload=1:flux, headers={id=a1df09ef-f951-293d-ee97-ace65a9df53c, timestamp=1681631097637}])
[oundedElastic-2] integrationSubFlow1: GenericMessage [payload=1:flux, headers={id=a1df09ef-f951-293d-ee97-ace65a9df53c, timestamp=1681631097637}]
[oundedElastic-2] integrationSubFlow2: GenericMessage [payload=1:flux, headers={id=a1df09ef-f951-293d-ee97-ace65a9df53c, timestamp=1681631097637}]
[oundedElastic-5] fluxFlow           : onNext(GenericMessage [payload=2:integration2, headers={id=c1147da7-e103-1fe5-a2ce-538697f6b951, timestamp=1681631097700}])
[oundedElastic-5] integrationSubFlow1: GenericMessage [payload=2:integration2, headers={id=c1147da7-e103-1fe5-a2ce-538697f6b951, timestamp=1681631097700}]
[oundedElastic-5] integrationSubFlow2: GenericMessage [payload=2:integration2, headers={id=c1147da7-e103-1fe5-a2ce-538697f6b951, timestamp=1681631097700}]
[oundedElastic-1] fluxFlow           : onNext(GenericMessage [payload=2:integration1, headers={id=09806cd0-7b41-3780-327d-1aa17e0013bb, timestamp=1681631097699}])
[oundedElastic-1] integrationSubFlow1: GenericMessage [payload=2:integration1, headers={id=09806cd0-7b41-3780-327d-1aa17e0013bb, timestamp=1681631097699}]
[oundedElastic-1] integrationSubFlow2: GenericMessage [payload=2:integration1, headers={id=09806cd0-7b41-3780-327d-1aa17e0013bb, timestamp=1681631097699}]
[oundedElastic-2] fluxFlow           : onNext(GenericMessage [payload=2:flux, headers={id=f097ef8d-c29f-f51d-cc29-49b91fb3e7bc, timestamp=1681631098639}])
[oundedElastic-2] integrationSubFlow1: GenericMessage [payload=2:flux, headers={id=f097ef8d-c29f-f51d-cc29-49b91fb3e7bc, timestamp=1681631098639}]
[oundedElastic-2] integrationSubFlow2: GenericMessage [payload=2:flux, headers={id=f097ef8d-c29f-f51d-cc29-49b91fb3e7bc, timestamp=1681631098639}]
```  

### DataType Channel
메시지 플로우 처리에 있어서 페이로드의 타입이 중요한 경우가 있다. 
이런 경우 매번 타입검사를 수행해 줘야 하는데, 필터를 사용하거나 라우터 기반으로 타입이 맞지 않는다면 
타입 변환을 해주는 곳으로 라우팅하는 등의 방법이 있다. 
`Spring Integration` 에서는 [DataType Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DatatypeChannel.html)
을 적용하면 별도의 추가적인 구현 없이 페이로드의 타입의 안정성을 보장 할 수 있다. 
추가적으로 특정경우 타입변환이 필요하다면 별도의 `IntegrationConveter` 를 빈으로 등록해 
사요할 수 있으니 타입 변환에 대한 처리도 추가적으로 수행 할 수 있다.  

만약 설정한 페이로드의 데이터 타입과 맞지 않은 데이터가 채널로 들어온다면 아래와 같은 예외가 발생한다. 
아래 예외는 `Integer` 만 혀옹하는 채널에서 `String` 타입의 데이터가 들어 왔을 떄 발생한 예외이다.  

```
ERROR 72434 --- [   scheduling-1] o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessageDeliveryException: failed to send Message to channel 'someChannel'; nested exception is org.springframework.core.convert.ConversionFailedException: Failed to convert from type [java.lang.String] to type [java.lang.Integer] for value '0:str'; nested exception is java.lang.NumberFormatException: For input string: "0:str", failedMessage=GenericMessage [payload=0:str, headers={id=7be79160-fd39-d9f7-f2f5-540ac9d7128a, timestamp=1681638449766}]
	at org.springframework.integration.support.utils.IntegrationUtils.wrapInDeliveryExceptionIfNecessary(IntegrationUtils.java:166)
	at org.springframework.integration.channel.AbstractMessageChannel.send(AbstractMessageChannel.java:339)
	at org.springframework.integration.channel.AbstractMessageChannel.send(AbstractMessageChannel.java:272)
	at org.springframework.messaging.core.GenericMessagingTemplate.doSend(GenericMessagingTemplate.java:187)
	at org.springframework.messaging.core.GenericMessagingTemplate.doSend(GenericMessagingTemplate.java:166)
	at org.springframework.messaging.core.GenericMessagingTemplate.doSend(GenericMessagingTemplate.java:47)
	at org.springframework.messaging.core.AbstractMessageSendingTemplate.send(AbstractMessageSendingTemplate.java:109)
	at org.springframework.integration.endpoint.SourcePollingChannelAdapter.handleMessage(SourcePollingChannelAdapter.java:196)
	at org.springframework.integration.endpoint.AbstractPollingEndpoint.messageReceived(AbstractPollingEndpoint.java:475)
	at org.springframework.integration.endpoint.AbstractPollingEndpoint.doPoll(AbstractPollingEndpoint.java:461)
	at org.springframework.integration.endpoint.AbstractPollingEndpoint.pollForMessage(AbstractPollingEndpoint.java:413)
	at org.springframework.integration.endpoint.AbstractPollingEndpoint.lambda$createPoller$4(AbstractPollingEndpoint.java:348)
	at org.springframework.integration.util.ErrorHandlingTaskExecutor.lambda$execute$0(ErrorHandlingTaskExecutor.java:57)
	at org.springframework.core.task.SyncTaskExecutor.execute(SyncTaskExecutor.java:50)
	at org.springframework.integration.util.ErrorHandlingTaskExecutor.execute(ErrorHandlingTaskExecutor.java:55)
	at org.springframework.integration.endpoint.AbstractPollingEndpoint.lambda$createPoller$5(AbstractPollingEndpoint.java:341)
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54)
	at org.springframework.scheduling.concurrent.ReschedulingRunnable.run(ReschedulingRunnable.java:95)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: org.springframework.core.convert.ConversionFailedException: Failed to convert from type [java.lang.String] to type [java.lang.Integer] for value '0:str'; nested exception is java.lang.NumberFormatException: For input string: "0:str"
	at org.springframework.core.convert.support.ConversionUtils.invokeConverter(ConversionUtils.java:47)
	at org.springframework.core.convert.support.GenericConversionService.convert(GenericConversionService.java:192)
	at org.springframework.core.convert.support.GenericConversionService.convert(GenericConversionService.java:175)
	at org.springframework.integration.support.converter.DefaultDatatypeChannelMessageConverter.fromMessage(DefaultDatatypeChannelMessageConverter.java:76)
	at org.springframework.integration.channel.AbstractMessageChannel.convertPayloadIfNecessary(AbstractMessageChannel.java:382)
	at org.springframework.integration.channel.AbstractMessageChannel.send(AbstractMessageChannel.java:302)
	... 22 more
Caused by: java.lang.NumberFormatException: For input string: "0:str"
	at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.base/java.lang.Integer.parseInt(Integer.java:652)
	at java.base/java.lang.Integer.valueOf(Integer.java:983)
	at org.springframework.util.NumberUtils.parseNumber(NumberUtils.java:211)
	at org.springframework.core.convert.support.StringToNumberConverterFactory$StringToNumber.convert(StringToNumberConverterFactory.java:64)
	at org.springframework.core.convert.support.StringToNumberConverterFactory$StringToNumber.convert(StringToNumberConverterFactory.java:50)
	at org.springframework.core.convert.support.GenericConversionService$ConverterFactoryAdapter.convert(GenericConversionService.java:437)
	at org.springframework.core.convert.support.ConversionUtils.invokeConverter(ConversionUtils.java:41)
	... 27 more
```  

예제는 `DirectChannel` 을 만들어 `Integer` 와 `Boolean` 타입만 허용하도록 한다. 
그리고 `String` 타입의 경우에는 `Converter` 를 만들어 채널에서 사용할 수 있도록 등록한다.  

```java
@Configuration
public class DataTypeChannelConfig {
    @Bean
    public MessageChannel someChannel() {
        DirectChannel messageChannel = new DirectChannel();
        // Integer 와 Boolean 만 채널에서 허용하는 타입이다.
        messageChannel.setDatatypes(Integer.class, Boolean.class);

        return messageChannel;
    }

    // 문자열 타입으 경우 stringToIntgerConveter 를 통해 채널에서 허용가능한 타입으로 변환해 준다.
    @Bean
    @IntegrationConverter
    public Converter<String, Integer> stringToIntegerConverter() {
        return new Converter<String, Integer>() {
            @Override
            public Integer convert(String source) {
                return Integer.parseInt(source.split(":")[0]) * 100;
            }
        };
    }

    // string 데이터를 채널에 추가한다.
    // 만약 등록된 converter 가 없다면 예외가 발생한다.
    @Bean
    public IntegrationFlow stringFlow() {
        AtomicInteger counter = new AtomicInteger();
        return IntegrationFlows.fromSupplier(
                () -> counter.getAndIncrement() + ":str",
                poller -> poller.poller(Pollers.fixedRate(500))
        )
                .channel(this.someChannel())
                .get();
    }

    @Bean
    public IntegrationFlow integerFlow() {
        AtomicInteger counter = new AtomicInteger();
        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel(this.someChannel())
                .get();
    }

    @Bean
    public IntegrationFlow booleanFlow() {
        AtomicInteger counter = new AtomicInteger();
        return IntegrationFlows.fromSupplier(
                        () -> counter.getAndIncrement() % 2 == 0,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel(this.someChannel())
                .get();
    }

    @Bean
    public IntegrationFlow someChannelLogger() {
        return IntegrationFlows.from("someChannel")
                .log("someChannelLogger")
                .get();
    }
}
```  

```
someChannelLogger                        : GenericMessage [payload=0, headers={id=317eadb0-68c4-0687-30f9-6a60c9bc4954, timestamp=1681639006756}]
someChannelLogger                        : GenericMessage [payload=0, headers={id=ebfd884d-cd37-5eb1-1c97-b5b0f3ec37a9, timestamp=1681639006757}]
someChannelLogger                        : GenericMessage [payload=true, headers={id=c0f6ea4d-55da-83d6-7f4f-55932f564e09, timestamp=1681639006757}]
someChannelLogger                        : GenericMessage [payload=100, headers={id=1e78e8da-ede4-4577-7bc3-b25090cb9d8e, timestamp=1681639007258}]
someChannelLogger                        : GenericMessage [payload=false, headers={id=2c559423-6248-6c42-8c51-93782eff27b3, timestamp=1681639007258}]
someChannelLogger                        : GenericMessage [payload=1, headers={id=e59ec898-b3bd-b725-ca80-5f59fa8c26df, timestamp=1681639007258}]
someChannelLogger                        : GenericMessage [payload=200, headers={id=90b6208a-438d-69f7-9556-9626af41bbb2, timestamp=1681639007755}]
someChannelLogger                        : GenericMessage [payload=true, headers={id=18a2e6c5-93ab-aaad-fbfb-8997cc815ba2, timestamp=1681639007755}]
someChannelLogger                        : GenericMessage [payload=2, headers={id=ff31e2be-c919-bc85-0824-685ba6fa661f, timestamp=1681639007755}]
someChannelLogger                        : GenericMessage [payload=300, headers={id=f7b3ca09-8865-37ac-d754-04c00a35257d, timestamp=1681639008255}]
someChannelLogger                        : GenericMessage [payload=false, headers={id=30f51703-4b4e-989d-5265-fd9e02e402d7, timestamp=1681639008256}]
```  

### ChannelInterceptor
`ChannelInterceptor` 를 사용하면 메시지 채널에서 공통적으로 필요한 동작들을 처리할 수 있다. 
특정 메시지를 제외처리 한다던가, 채널을 모니터링, `send`, `receive` 동작을 가로체는 등의 동작을 추가로 수행할 수 있다.  

```java
public interface ChannelInterceptor {

    // send() 메소드 호출 전
    @Nullable
	default Message<?> preSend(Message<?> message, MessageChannel channel) {
		return message;
	}

    // send() 메소드 호출 후
	default void postSend(Message<?> message, MessageChannel channel, boolean sent) {
	}

    // send() 메소드 수행 완료 후
    default void afterSendCompletion(
			Message<?> message, MessageChannel channel, boolean sent, @Nullable Exception ex) {
	}

    // receive() 메소드 호출 전 (Queue 계열 채널에서만 사용된다.)
	default boolean preReceive(MessageChannel channel) {
		return true;
	}

    // receive() 메소드 호출 후 (Queue 계열 채널에서만 사용된다.)
	@Nullable
	default Message<?> postReceive(Message<?> message, MessageChannel channel) {
		return message;
	}

    // receive() 메소드 수행 완료 후 (Queue 계열 채널에서만 사용된다.)
	default void afterReceiveCompletion(@Nullable Message<?> message, MessageChannel channel,
			@Nullable Exception ex) {
	}
}
```  

```java
@Slf4j
@Configuration
public class ChannelInterceptorConfig {
    @Bean
    public MessageChannel someChannel() {
        DirectChannel queueChannel = new DirectChannel();
        queueChannel.addInterceptor(this.myChannelInterceptor());

        return queueChannel;
    }

    @Bean
    public ChannelInterceptor myChannelInterceptor() {
        return new ChannelInterceptor() {
            // send() 메소드 호출 전
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                log.info("preSend {}, {}", message);
                if(Integer.parseInt(message.getPayload().toString()) % 2 == 0) {
                    return message;
                } else {
                    return null;
                }
            }

            // send() 메소드 호출 후
            @Override
            public void postSend(Message<?> message, MessageChannel channel, boolean sent) {
                log.info("postSend {}, {}, {}", message.getPayload(), channel, sent);
                ChannelInterceptor.super.postSend(message, channel, sent);
            }

            // send() 메소드 수행 완료 후
            @Override
            public void afterSendCompletion(Message<?> message, MessageChannel channel, boolean sent, Exception ex) {
                log.info("afterSendCompletion {}", message.getPayload());
                ChannelInterceptor.super.afterSendCompletion(message, channel, sent, ex);
            }

            // receive() 메소드 호출 전 (Queue 계열 채널에서만 사용된다.)
            @Override
            public boolean preReceive(MessageChannel channel) {
                log.info("preReceive {}", channel);
                return ChannelInterceptor.super.preReceive(channel);
            }

            // receive() 메소드 호출 후 (Queue 계열 채널에서만 사용된다.)
            @Override
            public Message<?> postReceive(Message<?> message, MessageChannel channel) {
                log.info("postReceive {}", message.getPayload());
                if(Integer.parseInt(message.getPayload().toString()) % 3 == 0) {
                    return ChannelInterceptor.super.postReceive(message, channel);
                } else {
                    return null;
                }
            }

            // receive() 메소드 완료 후 (Queue 계열 채널에서만 사용된다.)
            @Override
            public void afterReceiveCompletion(Message<?> message, MessageChannel channel, Exception ex) {
                log.info("afterReceiveCompletion {}", message.getPayload());
                ChannelInterceptor.super.afterReceiveCompletion(message, channel, ex);
            }
        };
    }


    @Bean
    public IntegrationFlow produceFlow() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .log("produceFlow")
                .channel(this.someChannel())
                .get();
    }

    @Bean
    public IntegrationFlow logFlow() {
        return IntegrationFlows.from("someChannel")
                .log("logFlow")
                .get();
    }
}
```  

채널은 모든 `ChannelInterceptor` 동작이 수행되는 `QueueChannel` 을 사용했다. 
`preSend` 와 `postReceive` 에서 `null` 리턴을 통해 제외되는 메시지는 `QueueChannel` 내에서 예외가 발생한다. 
그리고 최종 결과인 `logFlow` 에 찍히는 페이로드는 2와 3의 공배수인 6의 배수를 가진 값들이다.  

```
.. 0 ..
produceFlow                              : GenericMessage [payload=0, headers={id=c63790ed-9cf0-38e6-3c58-e612f28b306c, timestamp=1681643706827}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=0, headers={id=c63790ed-9cf0-38e6-3c58-e612f28b306c, timestamp=1681643706827}], {}
c.w.s.i.m.ChannelInterceptorConfig       : postSend 0, bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()', true
c.w.s.i.m.ChannelInterceptorConfig       : afterSendCompletion 0
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
c.w.s.i.m.ChannelInterceptorConfig       : postReceive 0
c.w.s.i.m.ChannelInterceptorConfig       : afterReceiveCompletion 0
logFlow                                  : GenericMessage [payload=0, headers={id=c63790ed-9cf0-38e6-3c58-e612f28b306c, timestamp=1681643706827}]

.. 1 ..
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
o.s.integration.channel.QueueChannel     : Exception from afterReceiveCompletion in com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig$1@4724bc16
produceFlow                              : GenericMessage [payload=1, headers={id=d3d958ad-3a23-209a-840f-007716fc4e58, timestamp=1681643707842}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=1, headers={id=d3d958ad-3a23-209a-840f-007716fc4e58, timestamp=1681643707842}], {}
o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessageDeliveryException: Failed to send message to channel 'bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelIn

.. 2 ..
produceFlow                              : GenericMessage [payload=2, headers={id=8a642af3-4857-ce81-6980-d74a91bf72ba, timestamp=1681643707845}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=2, headers={id=8a642af3-4857-ce81-6980-d74a91bf72ba, timestamp=1681643707845}], {}
c.w.s.i.m.ChannelInterceptorConfig       : postSend 2, bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()', true
c.w.s.i.m.ChannelInterceptorConfig       : afterSendCompletion 2
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
c.w.s.i.m.ChannelInterceptorConfig       : postReceive 2
o.s.integration.channel.QueueChannel     : Exception from afterReceiveCompletion in com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig$1@4724bc16

.. 3 ..
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
o.s.integration.channel.QueueChannel     : Exception from afterReceiveCompletion in com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig$1@4724bc16
produceFlow                              : GenericMessage [payload=3, headers={id=5ce0e15f-27bb-1992-7524-0668bebb67e8, timestamp=1681643708869}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=3, headers={id=5ce0e15f-27bb-1992-7524-0668bebb67e8, timestamp=1681643708869}], {}
o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessageDeliveryException: Failed to send message to channel 'bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelIn

.. 4 ..
produceFlow                              : GenericMessage [payload=4, headers={id=e284d212-4cc0-73fe-b392-4477da389844, timestamp=1681643708871}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=4, headers={id=e284d212-4cc0-73fe-b392-4477da389844, timestamp=1681643708871}], {}
c.w.s.i.m.ChannelInterceptorConfig       : postSend 4, bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()', true
c.w.s.i.m.ChannelInterceptorConfig       : afterSendCompletion 4
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
c.w.s.i.m.ChannelInterceptorConfig       : postReceive 4
o.s.integration.channel.QueueChannel     : Exception from afterReceiveCompletion in com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig$1@4724bc16

.. 5 ..
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
o.s.integration.channel.QueueChannel     : Exception from afterReceiveCompletion in com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig$1@4724bc16
produceFlow                              : GenericMessage [payload=5, headers={id=ba0481a3-162b-968b-7a7c-431b7281ab39, timestamp=1681643709906}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=5, headers={id=ba0481a3-162b-968b-7a7c-431b7281ab39, timestamp=1681643709906}], {}
o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessageDeliveryException: Failed to send message to channel 'bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelIn

.. 6 ..
produceFlow                              : GenericMessage [payload=6, headers={id=bdaa7195-a071-383c-44c1-d3f34ecf28e4, timestamp=1681643709911}]
c.w.s.i.m.ChannelInterceptorConfig       : preSend GenericMessage [payload=6, headers={id=bdaa7195-a071-383c-44c1-d3f34ecf28e4, timestamp=1681643709911}], {}
c.w.s.i.m.ChannelInterceptorConfig       : postSend 6, bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()', true
c.w.s.i.m.ChannelInterceptorConfig       : afterSendCompletion 6
c.w.s.i.m.ChannelInterceptorConfig       : preReceive bean 'someChannel'; defined in: 'class path resource [com/windowforsun/spring/intergration/messagechannel/ChannelInterceptorConfig.class]'; from source: 'com.windowforsun.spring.intergration.messagechannel.ChannelInterceptorConfig.someChannel()'
c.w.s.i.m.ChannelInterceptorConfig       : postReceive 6
c.w.s.i.m.ChannelInterceptorConfig       : afterReceiveCompletion 6
logFlow                                  : GenericMessage [payload=6, headers={id=bdaa7195-a071-383c-44c1-d3f34ecf28e4, timestamp=1681643709911}]

```  

주의해야 할점은 `ChannelInterceptor` 메소드들이 호출되는 순서는 채널 유형에 따라 달라질 수 았다는 점이다. 
`receive()` 관련 메소드들은 `QueueChannel` 관련 채널들만 수행에 포함되는 것도 있고, 
`send()`, `receive()` 메소드의 호출 순서나 실제 인터셉터 되는 시점도 `sender` 와 `receiver` 스레드 타이밍에 따라 달라질 수 있다. 


### WireTap
`Spring Integration` 에서는 `wire tap` 이라는 간편한 인터셉터를 제공한다. 
[WireTap](https://www.enterpriseintegrationpatterns.com/patterns/messaging/WireTap.html) 
은 인터셉텨를 설정하는 것과 같이 채널과 플로우상에 설정할 수 있어 로깅등의 용도로 간편하게 사용 할 수 있다. 
주의 할점은 `WireTap` 동작은 비동기로 동작하지 않는 다는 점으로, 
메시지 플로우를 구성한 스레드내에서 다른 `EIP` 메소드들과 동일하게 수행된다. 
`WireTap` 을 통해 아래와 같은 동작을 구현할 수 있다. 

- 채널에서 메시지 플로우 가로체기
- 모든 메시지 가져오기 
- 메시지를 다른 채널로 전송하기 

`WireTap` 의 동작은 기존 메시지 플로우에서 추가적인 갈래를 `fork` 시키는 것과 동일하다. 
그리고 `fork` 된 흐름은 그 흐름을 구성하는 채널의 종류에 따라 결정 될 수 있다. 
만약 채널이 `Pollable` 이거나 `Executor` 채널이라면 비동기로 동작 할 수 있짐, 
다른 채널이라면 아닐 수도 있다는 점이다. 
이러한 특징의 장점은 `WireTap` 의 동기/비동기의 요구조건이 변경 됐을 떄 구현로직에 변경이 필요하지 않고, 
적절한 채널로 다시 연결해주면 된다는 것이다.  

또한 `WireTap` 은 `selector`, `selectpr-expression` 속성을 통해 조건부로 동작 할 수 있도록 구성할 수 있다. 
`selector-expression` 은 `boolean SpEL` 표현식을 사용한다. 

아래는 `WireTap` 을 사용한 간단한 예시이다.  

```java
@Configuration
@RequiredArgsConstructor
public class WireTapConfig {
    @Bean
    public MessageChannel someChannel() {
        return MessageChannels
                .direct("someDirectChannel")
                .get();
    }

    @Bean
    public MessageChannel someExecutorChannel() {
        return MessageChannels
                .executor("evenNumberLogChannel", Executors.newSingleThreadExecutor())
                .get();
    }

    @Bean
    public IntegrationFlow produceFlow() {
        AtomicInteger counter = new AtomicInteger();

        return IntegrationFlows.fromSupplier(
                        counter::getAndIncrement,
                        poller -> poller.poller(Pollers.fixedRate(500))
                )
                .channel("someDirectChannel")
                .channel("someExecutorChannel")
                .get();
    }

    @Bean
    public IntegrationFlow directConsumeFlow() {
        return IntegrationFlows.from("someDirectChannel")
                .log("directConsumeFlow")
                // logFlow WirTap 적용
                .wireTap("logFlow.input")
                .get();
    }

    @Bean
    public IntegrationFlow executorConsumeFlow() {
        return IntegrationFlows.from("someExecutorChannel")
                .log("executorConsumeFlow")
                // evenNumberLogFlow WireTap 적용
                .wireTap("evenNumberLogFlow.input", wireTapSpec -> wireTapSpec.selector("payload % 2 == 0"))
                .get();
    }

    // 전체 메시지 로깅 WireTap
    @Bean
    public IntegrationFlow logFlow() {
        return flow -> flow.log("logFlow");
    }

    // 짝수 메시지만 로깅 WireTap
    @Bean
    public IntegrationFlow evenNumberLogFlow() {
        return flow -> flow.log("evenNumberLogFlow");
    }
}
```  

`logFlow` 는 전체 메시지에 대해서 수행되고, `evenNumberLogFlow` 는 짝수인 메시지에 대해서만 수행되는 것을 확인 할 수 있다. 
그리고 `logFlow` 는 `DirectChannel` 과 연결됐기 때문에 `scheduling` 스레드에서 동작하고, `eventNumberLogFlow` 는 `ExecutorChannel` 과 연결 됐기 때문에 
별도의 스레드풀에서 동작하는 것을 확인 할 수 있다.  


```
[pool-1-thread-1] executorConsumeFlow : GenericMessage [payload=0, headers={id=223adc4d-1d51-9afe-8787-ac127df960ae, timestamp=1681743763513}]
[pool-1-thread-1] evenNumberLogFlow   : GenericMessage [payload=0, headers={id=223adc4d-1d51-9afe-8787-ac127df960ae, timestamp=1681743763513}]
[   scheduling-1] directConsumeFlow   : GenericMessage [payload=1, headers={id=22f10825-8f82-c2aa-1e2e-df4fe2596dac, timestamp=1681743764014}]
[   scheduling-1] logFlow             : GenericMessage [payload=1, headers={id=22f10825-8f82-c2aa-1e2e-df4fe2596dac, timestamp=1681743764014}]
[pool-1-thread-1] executorConsumeFlow : GenericMessage [payload=2, headers={id=687ca247-6ab0-72a3-5a72-d31cc3585f9f, timestamp=1681743764512}]
[pool-1-thread-1] evenNumberLogFlow   : GenericMessage [payload=2, headers={id=687ca247-6ab0-72a3-5a72-d31cc3585f9f, timestamp=1681743764512}]
[   scheduling-1] directConsumeFlow   : GenericMessage [payload=3, headers={id=cdbc2185-ba70-a968-eb9a-4b05736fb686, timestamp=1681743765011}]
[   scheduling-1] logFlow             : GenericMessage [payload=3, headers={id=cdbc2185-ba70-a968-eb9a-4b05736fb686, timestamp=1681743765011}]
[pool-1-thread-1] executorConsumeFlow : GenericMessage [payload=4, headers={id=790b9a31-a746-3040-752b-f051c67c68eb, timestamp=1681743765512}]
[pool-1-thread-1] evenNumberLogFlow   : GenericMessage [payload=4, headers={id=790b9a31-a746-3040-752b-f051c67c68eb, timestamp=1681743765512}]
[   scheduling-1] directConsumeFlow   : GenericMessage [payload=5, headers={id=3c4c87b2-0a4f-0ffa-9357-0d0bbf5591bf, timestamp=1681743766012}]
[   scheduling-1] logFlow             : GenericMessage [payload=5, headers={id=3c4c87b2-0a4f-0ffa-9357-0d0bbf5591bf, timestamp=1681743766012}]
[pool-1-thread-1] executorConsumeFlow : GenericMessage [payload=6, headers={id=3f4280f6-43a2-b65e-1e22-86158e2dbeea, timestamp=1681743766514}]
```  


---  
## Reference
[Messaging Channels](https://docs.spring.io/spring-integration/docs/current/reference/html/core.html#channel)  
[Message Store](https://docs.spring.io/spring-integration/docs/current/reference/html/message-store.html#message-store)  
