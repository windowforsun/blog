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
- `STOMP` 프로토콜 사용시 `Spring` 프레임워크의 기능과 보안에 대해서 원시적인 `WebSocket` 을 사용하는 것보다 활용성이 높다.
- 이는 원시적인 `TCP` 프로토콜에 비해 `HTTP` 프로토콜을 사용하면 `Spring MVC` 와 같은 다양한 프레임워크를 활용하고 기능을 사용할 수 있는 것과 같다.
- 사용자가 별도로 메세지 포맷 및 프로토콜을 정의할 필요 없다.
- `Spring` 프레임워크에서는 `STOMP` 클라이언트를 제공하기 때문에 이를 다양하게 활용 가능하다.
- 외부 메시지 브로커(`RabbitMQ`, `ActiveMQ`)를 사용해서 확장 가능한 `Pub/Sub` 구현이 가능하다.
- `WebSocket` 은 하나의 목적지에 대해 `WebSocketHandler` 구현을 통해 라우팅을 수행하지만, `STOMP` 는 하나 이상의 `@Controller` 와 헤더를 통해 메시지 라우팅을 정의할 수 있다.
- `Spring Security` 를 사용해서 `STOMP` 목적지와 메시지 타입을 바탕으로 메시지 보안이 가능하다.

### Enable STOMP
- `WebSocket` 상에서 `STOMP` 지원은 `spring-messaging`, `spring-websocket` 모듈을 통해 사용 가능하다.
- 모듈에 대한 의존성만 있으면 `WebSocket` 에 접속가능한 `SockJS` 상에서 `STOMP` 의 엔드포인트를 설정할 수 있다.

	```java
	@Configuration
	@EnableWebSocketMessageBroker
	public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	
	    @Override
	    public void registerStompEndpoints(StompEndpointRegistry registry) {
	        registry.addEndpoint("/portfolio").withSockJS();  
	    }
	
	    @Override
	    public void configureMessageBroker(MessageBrokerRegistry config) {
	        config.setApplicationDestinationPrefixes("/app"); 
	        config.enableSimpleBroker("/topic", "/queue"); 
	    }
	}
	```  
	
	- `addEndpoint()` 를 통해 `/portfolio` 엔드포인트를 설정했고, 이는 `WebSocket`(`SockJS`) 를 통해 연결(`Handshake`) 할 `HTTP URL` 이다.
	- `setApplicationDestinationPrefixes()` 를 통해 `/app` 접두사 설정을 통해, `destination` 헤더가 `/app` 으로 시작하는 메시지는 `@Controller` 클래스 안에 있는 `@MessageMapping` 메소드로 라우팅 된다.
	- `enableSimpBroker()` 를 통해 인메모리 브로커(`SimpleBroker`)에 `/topic`, `/queue` 경로 설정을 통해, `destination` 헤더가 `/topic`, `/queue` 로 시작하는 메시지는 브로커에게 라우팅 한다.
	- 브로커 설정에서 `/topic`, `/queue` 는 특별한 의미를 가진 경로가 아닌 `pub-sub` 과 `point-to-point` 를 구분하기 위한 차이만있고, 외부 브로커를 사용할 경우 지원하는 경로 확인후 사용이 필요하다.
- 브라우저에서 `sockjs-client` 를 사용해서 `STOMP` 서버에 접속하기 위해서는 `STOMP` 클라이언트가 필요하다.
	- 많은 애플리케이션에서 [jmesnil/stomp-websocket] 
	(https://github.com/jmesnil/stomp-websocket) 를 사용해 왔지만, 요즘 관리가 되지 않고 있다.
	- 최근에는 [JSteunou/webstomp-client]
	(https://github.com/JSteunou/webstomp-client) 도 많이 사용되고, 관리도 지속적으로 되고 있다.
- `SockJS` 를 사용해서 `STOMP` 에 접속하는 코드는 아래와 같다.

	```javascript
	var socket = new SockJS("/spring-websocket-portfolio/portfolio");
	var stompClient = webstomp.over(socket);
	
	stompClient.connect({}, function(frame) {
	}
	```  
	
- `WebSocket` 를 사용한다면 코드는 아래와 같다.

	```javascript
	var socket = new WebSocket("/spring-websocket-portfolio/portfolio");
	var stompClient = Stomp.over(socket);
	
	stompClient.connect({}, function(frame) {
	}
	```  
	
- `STOMP` 예제 관련 링크는 아래와 같다.
	- [Using WebSocket to build an interactive web application](https://spring.io/guides/gs/messaging-stomp-websocket/)
	- [spring-websocket-portfolio](https://github.com/rstoyanchev/spring-websocket-portfolio)

### WebSocket Server
- 원시 `WebSocket` 서버 [설정]({{site.baseurl}}{% link _posts/2020-05-09-spring-concept-websocket.md#server-configuration %})
을 기반으로 `Jetty` 를 사용한 `WebSocket` 서버 설정은 아래와 같이, `StompEndpointRegistry` 에 `WebSocketPolicy` 및 `HandshakeHandler` 설정이 추가로 필요하다.
	
	```java
	@Configuration
	@EnableWebSocketMessageBroker
	public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	
	    @Override
	    public void registerStompEndpoints(StompEndpointRegistry registry) {
	        registry.addEndpoint("/portfolio").setHandshakeHandler(handshakeHandler());
	    }
	
	    @Bean
	    public DefaultHandshakeHandler handshakeHandler() {
	
	        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
	        policy.setInputBufferSize(8192);
	        policy.setIdleTimeout(600000);
	
	        return new DefaultHandshakeHandler(
	                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
	    }
	}
	```  

### Flow of Messages
- `STOMP` 의 엔드포인트를 설정하게 되면 `Spring` 애플리케이션은 연결된 클라이언트를 위한 `STOMP` 브로커가 되는데, 이때 클라이언트 부터 브로커를 통한 메시지 흐름에 대해서 알아본다.
- 앞서 언금한 것과 같이 `STOMP` 는 `spring-messaging` 모듈을 사용하는데, 이는 `Spring Integration` 에서 시작되었고 이후 더 광범위한 사용을 위해 `Spring` 프레임워크의 메시징 애플리케이션에서 사용 가능하도록 지원한다.
- 아래는 `spring-messaging` 모듈에서 사용가능한 추상체이다.
	- [Message](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/messaging/Message.html)
	: 메시지를 간단하게 `header` 와 `payload` 로 표현한다.
	- [MessageHandler](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/messaging/MessageHandler.html)
	: `handleMessage()` 메소드를 통해 인자값의 `Message` 를 핸들링 한다.
	- [MessageChannel](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/messaging/MessageChannel.html)
	: 메시지 생산자와 소비자 사이에 느슨한 결합을 기반으로 메시지를 전송할 수 있도록 한다.
	- [SubscribableChannel](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/messaging/SubscribableChannel.html)
	: `MessageHandler` 를 구독하는 `MessageChannel` 로  `MessageHandler` 가 구독자들에게 메시지를 헨들링 할 수 있도록 한다.
	- [ExecutorSubscribableChannel](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/messaging/support/ExecutorSubscribableChannel.html) 
	: `SubscribeChannel` 를 통해 실제로 구독자들에게 메시지를 보낼 때 사용하는 `Executor` 이다.
- `Spring` 프레임워크에서는 `Java Configuration`(`@EnableWebSocketMessageBroker`), `XML`(`<websocket:message-broker>`) 를 통해 `STOMP` 관련 설정이 가능하다.
- 아래는 내장 브로커를 사용할 때 메시지 흐름에 대한 그림이다.

	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-stomp-1.png)
	
	- 위 그림에서는 3개의 채널이 존재한다.
	- `clientInboundChannel`(request channel) : `WebSocket` 클라이언트로 부터 메시지를 받을 때 사용하는 채널이다.
	- `clientOutboundChannel`(response channel) : 서버가 `WebSocket` 클라이언트에게 메시지를 보낼 때 사용하는 채널이다.
	- `brokerChannel`(broker channel) : 서버에서 브로커에게 메시지를 보낼 때 사용하는 채널이다.
- 아래는 외부 브로커(`RabbitMQ`, `ActiveMQ`) 를 사용할 때 메시지 흐름에 대한 그림이다.

	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-stomp-2.png)

	- 내장 브로커를 사용한 그림과의 가장 큰 차이는 `TCP` 연결을 통해 외부 `STOMP` 브로커에게 메시지를 전달하고, 브로커로 부터 메시지를 받아 클라이언트에게 전달하는 방식인 `broker relay` 를 사용한다는 점이다.
- `WebSocket` 연결을 통해 메시지를 받으면 `STOMP` 메시지로 디코딩처리를 통해 `Spring` 의 메시지 표현으로 변환되고 이후 처리를 위해 `clientInboundChannel` 로 보내진다.
- 목적지 헤더가 `/app` 로 시작하는 `STOMP` 메시지는 매칭되는 `@MessageMapping` 메소드로 전달되고, `/topic`, `/queue` 와 같은 구독관련 메시지는 브로커에게 전달된다.
- `@Controller` 클래스는 클라이언트가 전송한 `STOMP` 메시지를 `brokerChannel` 을 통해 브로커에게 전달하고, 브로커는 메시지의 목적지와 매칭되는 구독자들에게 `clientOutBoundChannel` 을 통해 브로드캐스트 한다.
- 위와 같은 컨트롤러는 `HTTP` 요청에 대한 매핑도 가능하기 때문에, `HTTP POST` 요청을 통해 구독자들에게 메시지를 브로드캐스트 하는 방식도 가능하다.
- 아래와 같은 `STOMP` 관련 설정과 `@Controller` 클래스가 있다고 할때 처리흐름은 다음과 같다.

	```java
	@Configuration
	@EnableWebSocketMessageBroker
	public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	
	    @Override
	    public void registerStompEndpoints(StompEndpointRegistry registry) {
	        registry.addEndpoint("/portfolio");
	    }
	
	    @Override
	    public void configureMessageBroker(MessageBrokerRegistry registry) {
	        registry.setApplicationDestinationPrefixes("/app");
	        registry.enableSimpleBroker("/topic");
	    }
	}
	
	@Controller
	public class GreetingController {
	
	    @MessageMapping("/greeting") {
	    public String handle(String greeting) {
	        return "[" + getTimestamp() + ": " + greeting;
	    }
	}
	```  
	
	1. 클라이언트가 `http://localhost:8080/portfolio` 주소를 통해 연결을 시도하면 연결이 맺어지고, 해당 연결을 통해 `STOMP` 메시지를 주고 받을 수 있다.
	1. 클라이언트가 `SUBSCRIBE` 프레임을 목적지 헤더 `/topic/greeting` 과 함께 전송하게 되면, 해당 메시지는 디코딩되고 `clientInboundChannel` 에 전송된다. 그리고 메시지 브로커에게 전달되서 클라이언트의 구독정보를 저장한다.
	1. 클라이언트가 `SEND` 프레임을 목적지 헤더 `/app/greeting` 과 함께 전송하면 설정된 `/app` 접두사를 통해 해당 메시지는 `@Controller` 클래스(`GreetingController`)에게 전달된다. 이후 접두사 `/app` 을 제외한 `/greeting` 은 매칭되는 `@MessageMapping` 메소드(`handle()`)에게 전달된다.
	1. `GreetingController` 의 `handle()` 메소드에서 리턴한 값은 `Spring` 메시지의 `payload` 로 설정되고, 목적지 헤더의 접두사가 `/app` 에서 `/topic` 으로 변경된 `/topic/greeting` 으로 목적지를 기본 목적지로 설정한다. 만들어진 메시지는 `brokerChannel` 에 전달되고 브로커에 의해 처리 된다.
	1. 메시지 브로커는 전달 받은 메시지의 목적지를 구독하는 구독자들에게 `childOutboundChannel` 을 통해 메시지를 전송한다. 전송하려는 메시지는 `STOMP` 프레임으로 인코딩되고 `WebSocket` 연결에 의해 전송된다.

### Annotated Controllers
- 클라이언트로부터 전송된 메시지는 `@Controller` 클래스로 매핑되 처리되는데, 해당 클래스에서는 `@MessageMapping`, `@SubscribeMapping`, `@ExceptionHandler` 를 통해 메시지 처리가 가능하다.

#### @MessageMapping
- `@MessageMapping` 은 목적지를 매핑하는 용도로 사용된다.
- 메소드 레벨, 타입 레벨에 사용될 수 있는데, 타입 레벨에 사용되면 하나의 컨트롤러에 있는 모든 메소드에 공통으로 매핑되는 경로로 사용 된다.
- 목적지 매핑은 기본적으로 `Ant-style` 을 사용한다. (`/thing*`, `/thing/**`)
- 목적지의 값에는 `template variables` 사용 가능하다. (`/thing/{id}`)
- 목적지의 값에 사용된 `template variables` 는 메소드 인자에서 `@DestinationVariables` 통해 참조 가능하다.
- `/` 외에도 설정을 통해 `.` 구분자(`dot-separated`) 사용이 가능하다.
- `@MessageMapping` 이 선언된 메소드는 아래와 같은 인자를 사용할 수 있다.
	- `Message` : 전달된 메시지에 대해 접근할 수 있다.
	- `MessageHeader` : 전달된 `Message` 의 헤더에 접근 할 수 있다.
	- `MessageHeaderAccessor`, `SimpMessageHeaderAccessor`, `StompHeaderAccessor` : 헤더를 `strongly type` 으로 받을 수 있고 수정 가능하다.
	- `@Payload` : `전달된 `Message` 의 `payload` 를 설정된 `MessageConverter` 에 의해 변환된 값으로 접근 할 수 있다. 


















































---
## Reference
[Web on Servlet Stack - WebSockets STOMP](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#websocket-stomp)  
[STOMP](https://stomp.github.io/index.html)  
[Using WebSocket to build an interactive web application](https://spring.io/guides/gs/messaging-stomp-websocket/#websocket)  