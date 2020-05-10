--- 
layout: single
classes: wide
title: "[Spring 개념] WebSocket STOMP"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - WebSocket
  - STOMP
toc: true
---  

## STOMP
- [WebSocket]({{site.baseurl}}{% link _posts/2020-05-09-spring-concept-websocket.md %})
은 `Text`, `Binary` 두 가지 방식의 메시지를 사용 할 수 있지만 이는 정의되지 않은 메시지 포맷이라는 특징이 있다.
- `STOMP` 는 `high-level messaging protocol` 지원을 통해 서버-클라이언트 간 특정 포멧을 통해 메시지를 주고 받을 수 있다.
- 또한 `sub-protocol` 정의를 통해 서버-클라이언트 간의 약속된 동작 수행이 가능하도록 설계할 수 있다.
- `STOMP` 는 `Simple Text Oriented Messaging Protocol` 의 약자로 원래는 스크립트 언어(`Ruby`, `Python`, ..) 에서 메시지 브로커와의 연결을 위해 만들어 졌다.
- 일반적으로 사용되는 메시지 패턴에서 최소한의 동작을 수행 할 수 있도록 설계되었다.
- `STOMP` 는 신뢰할 수 있는(`TCP`, `WebSocket`) 양방항 통신 프로토콜상에서 사용할 수 있는 메시지 프로토콜이다.
- 텍스트 기반 프로토콜이지만 메시지의 `payload` 는 일반적인 텍스트 이거나 바이너리일 수 있다.

### STOMP Message
- `STOMP` 는 `frame-based` 프로토콜로 프레임은 `HTTP` 의 모델을 기반으로 한다.

	```
	COMMAND
    header1:value1
    header2:value2
    
    Body^@
	```  
	
	- 클라이언트는 `SEND`, `SUBSCRIBE` 명령(`COMMAND`)을 사용해서 메시지를 보내거나 구독을 통해 받을 수 있다.
	- 메시지의 목적지는 `header` 의 `DESTINATION` 필드를 통해 나타낸다.
	- 이러한 방식을 통해 클라이언트는 메시지 브로커를 통해 다른 클라이언트들에게 메시지를 보낼 수 있고, 받을 수 있는(서버) `Pub/Sub` 구현이 가능하다.
- `Spring STOMP` 를 사용하면 `Spring WebSocket` 애플리케이션은 `STOMP Broker` 의 역할을 수행하게 된다.
- 기본적으로 메시지는 `@Controller` 가 선언된 메시지 핸들러에 전달되 처리되고, `In-Memory` 를 통해 다른 클라이언트들에게 전달 된다.
- `In-Memory` 브로커 외에도 `RabbitMQ`, `ActiveMQ` 와 같은 전용(외부) 브로커를 설정해서 사용 할 수도 있다.
	- 이 경우 `Spring STOMP` 애플리케이션은 외부 브로커와 `TCP` 연결을 유지하면서, 외부 브로커에게 메시지를 전달하고 전달 받은 식으로 수행 된다.
- 이러한 구성을 통해 통합된 `HTTP` 기반 보안과, 공통 검증, 친숙한 프로그래밍 방식을 유지하며 `STOMP` 개발이 가능하다.
- 아래는 클라이언트가 특정 경로에 대한 구독을 수행하는 메시지 이다.

	```
	SUBSCRIBE
    id:sub-1
    destination:/topic/price.stock.*
    
    ^@
	```  
	
	- 구독에 대한 메시지는 `SimpMessagingTemplate` 를 통해 브로커에게 전송 된다.
- 아래는 클라이언트가 서버에게 요청을 보내는 메시지 이다.

	```
	SEND
    destination:/queue/trade
    content-type:application/json
    content-length:44
    
    {"action":"BUY","ticker":"MMM","shares",44}^@
	```  
	
	- 서버에서는 `@MessageMapping` 을 통해 클라이언트가 보내느 요청 경로에 대한 처리 메소드를 매핑 시킬 수 있다.
	- 이후 서버는 클라이언트의 요청을 처리한 후에, 처리에 대한 결과를 브로커를 통해 클라이언트들에게 응답할 수 있다.
- `STOMP` 메시지 헤더에서 `destination` 경로에 대한 포멧은 정확하게 정해진 바는 없고, 서버가 어떤 형식으로 정하냐에 따라 달라질 수 있다. 아래는 보편적으로 사용하는 목적지 경로 형식이다.
	- `/topic/..` : `publish-subscribe`(one-to-many)
	- `/queue/..` : `point-to-point`(one-to-one)
- `STOMP` 서버는 `MESSAGE` 명령을 통해 목적지 경로를 구독하고 있는 모든 구독자들에게 브로드케스트 할 수 있는데, 아래와 같다.

	```
	MESSAGE
    message-id:nxahklf6-1
    subscription:sub-1
    destination:/topic/price.stock.MMM
    
    {"ticker":"MMM","price":129.45}^@
	```  
	
	- 서버는 클라이언트가 `SUBSCRIBE` 요청시에 헤더에 보냈던 `id` 필드의 값을 통해 클라이언트에게 구독에 대한 응답을 전송한다.
	- 위 메세지 헤더에서 `subscription-id` 는 클라이언트가 보냈던 헤더의 `id` 필드의 값과 동일하다.
	- 이러한 방식으로 서버는 구독하지 않은 클라이언트에게 메시지를 전송할 수 없다.
	
### STOMP 장점

### Enable STOMP

### WebSocket Server

### Flow Messages

### Annotated Controllers


---
## Reference
[Web on Servlet Stack - WebSockets STOMP](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#websocket-stomp)  
[STOMP](https://stomp.github.io/index.html)  
[Using WebSocket to build an interactive web application](https://spring.io/guides/gs/messaging-stomp-websocket/#websocket)  