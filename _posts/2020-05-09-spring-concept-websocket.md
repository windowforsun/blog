--- 
layout: single
classes: wide
title: "[Spring 개념] WebSocket"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring WebSocket 에 대해 알아보고 간단한 채팅 애플리케이션을 구현해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - WebSocket
toc: true
---  

## WebSocket
- `WebSocket` 은 [RFC 6455](https://tools.ietf.org/html/rfc6455)
규격의 프로토콜을 제공하는 표준화된 통신방식이다.
- 서버-클라이언트 간의 `full-duplex`, 양방향 통신, 채널 등을 `TCP` 연결을 통해 제공한다.
- 기본적인 `TCP` 의 프로토콜과는 달리 `HTTP` 기반을 사용해 연결(`Handshake`) 및 통신 을 수행한다.(80 및 443 포트 사용)
- `WebSocket` 은 `HTTP` 를 통해 연결(`Handshake`) 을 수행하는 과정에서, `HTTP Upgrade` 헤더를 통해 업그러드를 한 후 `WebSocket` 프로토콜로 전환되어 연결(`TCP Connection`)을 유지한다.

### WebSocket Handshake 와 Message
- 앞서 언급한것과 같이 `WebSocket` 은 `HTTP` 프로토콜을 사용해서 `Handshake` 를 수행한다.
- `클라이언트 -> 서버` 로 `Handshake` 를 요청하는 패킷은 아래와 같다

	```
	GET /spring-websocket-portfolio/portfolio HTTP/1.1
	Host: localhost:8080
	Upgrade: websocket 
	Connection: Upgrade 
	Sec-WebSocket-Key: Uc9l9TMkWGbHFD2qnFHltg==
	Sec-WebSocket-Protocol: v10.stomp, v11.stomp
	Sec-WebSocket-Version: 13
	Origin: http://localhost:8080
	```  
	
	- `GET` 메소드를 사용해서 요청을 하는 것을 확인 할 수 있다.
	- 기존 `HTTP` 프로토콜에서 `Upgrade` 헤더의 `websocket` 값과 `Connection` 헤더의 `Upgrade` 값을 바탕으로 `WebSocket` 프로토콜로 변경된다.
	- `Sec-WebSocket-Key` 는 보안을 위한 요청 키이다.
	- `Sec-WebSocket-Protocol` 사용한 가능한 `WebSocket` 프로토콜을 알려준다.
	
- 위 요청을 받은 서버는 클라이언트로 아래 패킷을 응답해 준다.

	```
	HTTP/1.1 101 Switching Protocols 
    Upgrade: websocket
    Connection: Upgrade
    Sec-WebSocket-Accept: 1qVdfYHU9hPOl4JYYNXF623Gzn0=
    Sec-WebSocket-Protocol: v10.stomp
	```  
	
	- `101 Switching Protocols` 를 통해 이후 부터 `WebSocket` 프로토콜로 통신이 가능하다.
	- `Sec-WebSocket-Accept` 는 보안을 위한 응답 키이다.
	- `Sec-WebSocket-Protocol` 로 사용할 프로토콜을 알려준다.

## Spring WebSocket API
- `Spring` 은 `WebSocket` 구현을 위해 아래와 같은 다양한 `API` 를 제공한다.

### WebSocketHandler
- `WebSocket` 서버의 생성은 `WebSocketHandler` 의 하위클래스 구현을 통해 가능하다.
- `WebSocketHandler` 의 하위 클래스로 `TextWebSocketHandler`, `BinaryWebSocketHandler` 가 있다.
	- `TextWebSocketHandler` 는 문자열 기반 메시지를 위한 핸들러이다.
	- `BinaryWebSocketHandler` 는 바이너리 기반 메시지를 위한 핸들러이다.
- `WebSocketHandler` 의 구현은 아래와 같이 할 수 있다.

	```java
	public class MyHandler extends TextWebSocketHandler {
    
        @Override
        public void handleTextMessage(WebSocketSession session, TextMessage message) {
            // ...
        }
    
    }
	```  
	
	```java
	@Configuration
    @EnableWebSocket
    public class WebSocketConfig implements WebSocketConfigurer {
    
        @Override
        public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
            registry.addHandler(myHandler(), "/myHandler");
        }
    
        @Bean
        public WebSocketHandler myHandler() {
            return new MyHandler();
        }
    
    }
	```  
	
	- `MyHandler` 에서 수행하는 동작은 `/myHandler` 로 연결된 세션에 적용된다.
	- `Spring WebSocket` 은 `Spring MVC` 에 의존하기 않기 때문에 `WebSocket` 이 `HTTP` 통신을 기반한다 하더라도 `DispatchServlet` 과 같은 관련 의존성은 필요없다.
	- `WebSocket` 이 `HTTP` 통신을 사용하기 위해 `WebSocketHttpRequestHandler` 를 사용한다.
	- `WebSocketHandler` 는 직접 구현해서 사용할 수 있고, 이미 구현된 [STOMP]() 와 같은 구현체를 사용 할 수 있다.
	- `WebSocketSession` 은 연결된 `Connection` 을 관리하는 클래스로, 해당 세션으로 메세지를 전송하거나 하는 동작이 가능하다. 하지만 기본적으로 동시에 많은 전송이 불가능하고, 하나씩 전송을 수행한다.
	- `ConcurrentWebSocketSessionDecorator` 는 `WebSocketSession` 에서 동시성을 부여한 랩퍼 클래스이다.

### WebSocketHandshake
- `WebSocket` 의 `Handshake` 동작에 대한 처리는 `HandshakeInterceptor` 를 통해 가능하다.
- `HandshakeInterceptor` 인터페이스는 `Handshake` 의 전과 후에 호출되는 콜백 메소드를 제공한다.
- 이러한 메소드 구현을 통해 `Handshake` 를 막거나 하는 처리가 가능하다.

	```java
	@Slf4j
	@Component
	public class MyHandshakeInterceptor implements HandshakeInterceptor {
	    @Override
	    public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
	        // ...
	        return true;
	    }
	
	    @Override
	    public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Exception e) {
	        // ...
	    }
	}
	```  
	
	```java
	@Configuration
	@EnableWebSocket
	public class WebSocketConfig implements WebSocketConfigurer {
	
	    @Override
	    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
	        registry.addHandler(new MyHandler(), "/myHandler")
	            .addInterceptors(new HttpSessionHandshakeInterceptor(), new MyHandshakeInterceptor());
	    }
	
	}
	```  

	- `HttpSessionHandshakeInterceptor` 는 기본적으로 제공하는 인터셉터로, `HTTP Session` 의 속성을  `WebSocketSession` 에서 사용할 수 있도록 해준다.
	- `DefaultHandshakeHandler` 를 통해 `Handshake`  동작을 확장 가능하다.
	- `RequestUpgradeStrategy` 의 구현을 통해 `WebSocket` 이 지원되지 않는 애플리케이션을 적용 할 수있다.

#### WebSocketHandlerDecorator
- `WebSocketHandlerDecorator` 를 사용해서 `WebSocketHandler` 에 대한 예외처리, 로깅 등 동작을 확장 할 수 있다.
- `ExceptionWebSocketHandlerDecorator` 는 `WebSocketHandler` 에서 발생하는 모든 예외를 캐치해서 소켓 세션 종료 및 `1011` 에러를 응답한다.


### Server Configuration
- `WebSocketConfigurer` 를 상속받고 `ServletServerContainerFactoryBean` 을 사용해서 메시지 버퍼 크기, 유휴 시간 등을 설정 할 수 있다.

	```java
	@Configuration
    @EnableWebSocket
    public class WebSocketConfig implements WebSocketConfigurer {
    
        @Bean
        public ServletServerContainerFactoryBean createWebSocketContainer() {
            ServletServerContainerFactoryBean container = new ServletServerContainerFactoryBean();
            container.setMaxTextMessageBufferSize(8192);
            container.setMaxBinaryMessageBufferSize(8192);
            return container;
        }
    }
	```  
	
- `Jetty` 를 사용할 경우 아래와 같이 `DefaultHandshakeHandler` 를 통해 설정 할 수 있다.

	```java
	@Configuration
    @EnableWebSocket
    public class WebSocketConfig implements WebSocketConfigurer {
    
        @Override
        public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
            registry.addHandler(echoWebSocketHandler(),
                "/echo").setHandshakeHandler(handshakeHandler());
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

### Allow Origins
- `WebSocketHandler` 가 수행 가능한 `Origin` 을 설정 할 수 있다.
- 기본 값은 동일한 `Origin` 만 허용 하는 것으로, IE6, IE7 에서는 지원되지 않는다.
- 특정 `Origin` 을 지정해서 허용 가능하고 `http://` 또는 `https://` 로 시작해야 한다. IE6 ~ IE9 까지 원되지 않는다.
- 모든 `Origin` 을 허용하기 위해서는 `*` 을 사용해서 설정 가능하다.

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").setAllowedOrigins("https://mydomain.com");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }
}
```  

## SockJS
- [SockJS](https://github.com/sockjs/sockjs-protocol)
 의 목표는 애플리케이션이 `WebSocket API` 를 사용 가능하지만, 코드의 변경없이 필요시에 `WebSocket` 으로 대체하는 것이다.
- `SockJS` 는 기본적으로 브라우저에서 사용하도록 설계되었지만, 다양한 플랫폼을 지원하고 있다.
- 플랫폼의 지원에 따라 전송 `WebSocket`, `HTTP Streaming`, `HTTP Long Polling` 을 사용한다.
	- [전송 방식 관련 링크](https://spring.io/blog/2012/05/08/spring-mvc-3-2-preview-techniques-for-real-time-updates/)
- 연결을 수행할 때 먼저 `GET /info` 를 요청해 서버의 기본 정보를 받는다.
- 이후 지원 여부에 따라 전송 방식을 선택해 통신을 수행한다.
- 전송 요청의 `URL` 은 아래와 같다.
	
	```
	http://host:port/app/endpoint/{server-id}/{session-id}/{transport}
	```  
	
	- `{server-id}` : 클러스터에서 요청을 라우팅하는데 사용하고, 그외에는 특별한 역할은 없다.
	- `{session-id}` : `HTTP` 요청에 포함된 `SockJS` 세션과 연관시킨다.
	- `{transport}` : 전송 방식을 나타낸다. (`websocket`, `xhr-streaming`, ...)
- `WebSocket` 은 처음 `Handshake` 과정에서만 `HTTP` 요청을 사용하고, 이후 부터는 `Socket` 통신을 사용한다.
- `HTTP` 전송은 보다 많은 요청이 요구된다.
	- `Ajax` 를 예로 들면, `Ajax` 는 서버-클라이언트에 사용되는 장시간 실행되는 하나의 요청과 추가적으로 클라이언트-서버에 사용되는 `HTTP POST` 요청에 의존한다.
- `Long polling` 방식 또한 서버-클라이언트 전송 후 현재 요청을 종료한다는 점을 제외한다면 위와 유사하다.
- `SockJS` 는 최소화된 메시지를 추가적가해 이를 해결한다.
	- `o` : `open` 을 의미하는 프레임으로, 소켓 연결 초기에 사용된다. 
	- `h` : `heartbeat` 을 의미하는 프레임으로, 25초(기본설정) 간격으로 전송된다.
	- `c` : `close` 를 의미하는 프레임으로, 세션을 종료할 때 사용된다.

### Enable SockJS
- `Spring WebSocket` 에서 `SockJS` 을 통한 연결을 허용하기 위한 설정은 아래와 같다.

	```java
	@Configuration
    @EnableWebSocket
    public class WebSocketConfig implements WebSocketConfigurer {
    
        @Override
        public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
            registry.addHandler(myHandler(), "/myHandler").withSockJS();
        }
    
        @Bean
        public WebSocketHandler myHandler() {
            return new MyHandler();
        }
    
    }
	```  
	
	- `SocketJS` 을 사용한 연결또한 `Spring MVC` 와 의존성이 없기 때문에 `DispatcherServlet` 에 대한 설정은 필요하지 않다.
	- `SockJsHttpRequestHandler` 을 통해 비교적 간단하게 다른 `HTTP` 서비스 환경과 통합 할 수 있다.
- 브라우저에서는 [sockjs-client](https://github.com/sockjs/sockjs-client/)
 를 사용해서 접속할 수 있는데, 이는 각 브라우저마다의 지원 여부 확인이 필요하다. 

### Heartbeats
- `SockJS` 는 서버에서 해당 연결이 끊겼다고 판별하지 않도록 하기 위해 `Heartbeat` 를 보낸다.
- `Spring SockJS` 에서는 `heartbeatTime` 프로퍼티를 통해 사용자가 `Heartbeat` 주기를 설정할 수 있다.
	- `heartbeatTime` 의 기본 값은 25초이다.
- `WebSocket`, `SockJS` 에서 `STOMP` 를 사용할때 `STOMP` 의 `Heartbeat` 로 인해 `SockJS` 의 `Heartbeat` 는 비활성화 된다.
- `Spring SockJS` 는 `TaskScheduler` 사용해서 `Heartbeat` Task 를 사용자 정의에 맞춰 구성할 수 있다.

### SockJsClient
- `Spring` 은 `SockJS` 자바 클라이언트를 제공해 원격 `SockJS` 엔드포인트에 연결할 수 있다.
- 서버간 양방향 통신이 필요할때 유용하게 사용될 수 있다.
	- `WebSocket` 의 사용이 불가할 때
- 다수의 사용자에 대한 시뮬레이션 테스트 등에 `SockJS` 클라이언트트 유용하게 사용 될 수 있다.
- `SockJS` 자바 클라이언트는 `WebSocket`, `xhr-streaming`, `xhr-polling` 전송을 지원하고 나머지 전송은 브라우저에서만 사용 가능하다.
- `WebSocketTransport` 는 `StandardWebSocketClient`(`JSR-356`) 와 `JettyWebSocketClient`(`Jetty9` 이상의 `WebSocket API`), `Spring` 의 `WebSocketClient` 구현체로 설정 할 수 있다.
- `XhrTransport` 는 `xhr-streaming`, `xhr-polling` 을 모두 지원하며 연결시 사용되는 `URL` 외에 차이점은 없다. 아래 2가지 구현체를 사용할 수 있다.
	- `RestTemplateXhrTransport` 는 `Spring` 의 `RestTemplate` 을 사용한 구현체이다.
	- `JettyXhrTransport` 는 `Jetty` 의 `HttpClient` 를 사용한 구현체이다.

```java
List<Transport> transports = new ArrayList<>(2);
transports.add(new WebSocketTransport(new StandardWebSocketClient()));
transports.add(new RestTemplateXhrTransport());

SockJsClient sockJsClient = new SockJsClient(transports);
sockJsClient.doHandshake(new MyWebSocketHandler(), "ws://example.com:8080/sockjs");
```  

- `SocketJS` 의 메시지는 `JSON` 형식을 사용하고 `Jackson2` 를 통해 파싱을 한다. `SockJsClient` 의 `SocketJsMessageCodec` 을 통해 파싱 부분을 커스텀하게 설정할 수 있다.
- 아래는 `SockJsClient` 를 사용해 다수의 사용자 시뮬레이션을 구현한 코드로 `Jetty` 클라이언트를 사용한다.

	```java
	HttpClient jettyHttpClient = new HttpClient();
    jettyHttpClient.setMaxConnectionsPerDestination(1000);
    jettyHttpClient.setExecutor(new QueuedThreadPool(1000));
	```  

- 아래는 `SockJS` 연결을 사용하는 `STOMP` 서버에서 `SockJS` 와 관련있는 커스텀 필드에 대한 설정의 예이다.

	```java
	@Configuration
    public class WebSocketConfig extends WebSocketMessageBrokerConfigurationSupport {
    
        @Override
        public void registerStompEndpoints(StompEndpointRegistry registry) {
            registry.addEndpoint("/sockjs").withSockJS()
                .setStreamBytesLimit(512 * 1024) 
                .setHttpMessageCacheSize(1000) 
                .setDisconnectDelay(30 * 1000); 
        }
    
        // ...
    }
	```  
	
	- `streamBytesLimit` 의 기본 값은 128KB(128 * 1024) 이다.
	- `httpMessageCacheSize` 의 기본 값은 100 이다.
	- `disconnectDelay` 의 기본 값은 5초(5 * 1000) 이다.
	
## 예제
- `WebSocket` 을 사용해서 간단한 채팅 애플리케이션을 구현한다.
- 하나의 엔드포인트에 사용자들이 접속해 채팅을 하는 애플리케이션이다.

### 디렉토리 구조

```  
src
└main
  ├─java
  │  └─com
  │      └─windowforsun
  │          └─websocketexam
  │              │  WebsocketExamApplication.java
  │              │
  │              ├─controller
  │              │      ViewController.java
  │              │
  │              ├─domain
  │              │      Message.java
  │              │
  │              └─websocket
  │                  ├─config
  │                  │      WebSocketConfig.java
  │                  │
  │                  └─handler
  │                          WebSocketHandler.java
  │
  └─resources
      │  application.yml
      │
      ├─static
      └─templates
              websocket.html
```  

### 의존성

```xml

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
```  

### application.yml

```yaml
spring:
  thymeleaf:
    prefix: classpath:templates/
    check-template: true
    suffix: .html
    mode: HTML
    cache: false
    check-template-location: false
  devtools:
    livereload:
      enabled: true
server:
  port: 8080
```  

### Application

```java
@SpringBootApplication
public class WebsocketExamApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebsocketExamApplication.class, args);
    }

}
```  

### WebSocketConfig

```java
// WebSocket 활성화
@EnableWebSocket
@Configuration
@Controller
public class WebSocketConfig implements WebSocketConfigurer {
	// 사용할 WebSocket 핸들러
    @Autowired
    private WebSocketHandler webSocketHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        webSocketHandlerRegistry
        		// WebSocketHandler 구현체를 /websocket 경로에 매핑
                .addHandler(this.webSocketHandler, "/websocket")
                // 모든 오리진 허용
                .setAllowedOrigins("*")
                // SockJS 를 통한 연결 설정
                .withSockJS();
    }
}
```  

### Message

```java
@Getter
@Setter
@ToString
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Message {
    private String roomId;
    private String name;
    private String message;
}
```  

### WebSocketHandler

```java
@Component
public class WebSocketHandler extends TextWebSocketHandler {
    private ObjectMapper objectMapper = new ObjectMapper();
    // 세선 저장소
    private Set<WebSocketSession> sessions = new HashSet<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
    	// 연결이 완료되면 세션 저장소에 추가
        this.sessions.add(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
    	// 클라이언트가 보낸 메시지 문자열
        String payload = message.getPayload();

        try {
        	// 정의된 형식으로 파싱
            this.objectMapper.readValue(payload, Message.class);
            // 세션 저장소에 있는 모든 세션에 클라이언트 메시지 전송
            this.sessions.parallelStream().forEach(sendSession -> {
                try {
                    sendSession.sendMessage(message);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        } catch(JsonProcessingException e) {
        	// 파싱 실패하면 예외처리
            e.printStackTrace();
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
    	// 연결이 끊기면 세션 저장소에서 삭제
        this.sessions.remove(session);
    }
}
```  

### ViewController

```java
@Controller
public class ViewController {
    @GetMapping("/websocket")
    public String websocket() {
        return "websocket";
    }
}
```  

### websocket.html

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="UTF-8">
    <script
            src="https://code.jquery.com/jquery-3.5.1.js"
            integrity="sha256-QWo7LDvxbWT2tbbQ97B53yJnYU3WhH/C8ycbRAkjPDc="
            crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/sockjs-client/1.4.0/sockjs.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/stomp.js/2.3.3/stomp.min.js"></script>
</head>
<body>
<h1 class="head"></h1>
<div style="overflow:scroll;width:500px;height:600px;" id="scrollBox" class="content">
    <ul class="chat_box" id="chatBox">
    </ul>
</div>
<input type="text" name="message" id="messageInput">
<button class="send">보내기</button>
</body>
<script type="text/javascript">
    $(function () {
        var chatBox = $('.chat_box');
        var messageInput = $('input[name="message"]');
        var sendBtn = $('.send');
        var head = $('.head');

        var name = prompt('사용자 이름', new Date().getTime());
        head.append('<span>사용자 이름 : ' + name + '</span>');

        var sock = new SockJS("/websocket");
        sock.onopen = function() {
            sock.send(JSON.stringify({message: name + ' Enter !', name: name}));
            sock.onmessage = function(e) {
                var content = JSON.parse(e.data);
                chatBox.append('<span>' + content.name + ' : ' + content.message + '</span><br>');
                $('#scrollBox').scrollTop($('#scrollBox')[0].scrollHeight);
            }
        }

        sendBtn.click(function () {
            sendMessage();
        });

        document.getElementById('messageInput').addEventListener('keydown', function (ev) {
            if(ev.key == 'Enter') {
                sendMessage();
            }
        });

        function sendMessage() {
            var message = messageInput.val().trim();
            if(message === '') {

            } else {
                sock.send(JSON.stringify({message: message, name: name}));
            }
            messageInput.val('');
        }
    });
</script>
</html>
```  

### 실행 결과
- 브라우저에서 `http://localhost:8080/websocket` 요청 
- 2개의 클라이언트 접속 및 메시지 전송
- `client-1` 접속
	
	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-1.png)
	
	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-2.png)

- `client-2` 접속

	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-3.png)
	
	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-4.png)
	
	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-5.png)
	
- 메시지 전송

	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-6.png)
	
	![그림 1]({{site.baseurl}}/img/spring/concept-websocket-7.png)
	


---
## Reference
[Web on Servlet Stack - WebSockets](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#websocket)  
[Using WebSocket to build an interactive web application](https://spring.io/guides/gs/messaging-stomp-websocket/#websocket)  
[WebSocket](https://en.wikipedia.org/wiki/WebSocket#websocket)  