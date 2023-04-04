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










---  
## Reference
[Messaging Channels](https://docs.spring.io/spring-integration/docs/current/reference/html/core.html#channel)  
