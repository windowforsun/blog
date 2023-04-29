--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Stream Source, Processor, Sink 개발"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Stream 을 사용해서 Source, Processor, Sink 애플리케이션을 개발하는 방법에 대해 알아보자'
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

## Spring Cloud Stream
`Spring Cloud Stream` 을 사용하면 `Spring` 환경에서 메시지기반 `Stream` 애플리케이션을 개발할 수 있다. 
그리고 추후에는 이를 `Spring Cloud Data Flow` 와 같은 툴을 사용해 `Stream` 을 시각화 해서 관리 또한 가능하다.  

예제로 개발할 애플리케이션은 아래와 같다.  







---  
## Reference
[Messaging Channels](https://docs.spring.io/spring-integration/docs/current/reference/html/core.html#channel)  
[Message Store](https://docs.spring.io/spring-integration/docs/current/reference/html/message-store.html#message-store)  
