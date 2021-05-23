--- 
layout: single
classes: wide
title: "[Spring 개념] Spring WebFlux"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring WebFlux 등장 배경과 특징과 주요구성에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring MVC
    - Netty
    - Reactive Streams
    - Spring WebFlux
    - Non-Blocking
toc: true
use_math: true
---  

## Spring WebFlux 등장
기존 `Spring MVC` 에서 클라이언트 요청을 `Servlet 3.0 API` 를 사용해서 처리하는 과정을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-1.png)  

`Servlet 3.0 API` 이하를 사용하는 경우 유저의 `Connection` 과 그 요청을 처리하는 `ServletThread` 는 1:1 매핑 관계를 가지게 된다. 
하나의 요청을 처리하는데 `Blocking` 동작으로 인해 1초가 걸린다고 가정했을 때 `ServletThread` 수만큼 `RPS` 가 발생하는 경우,  
개발자가 의도한 만큼의 `Throughput` 과 `Laytency` 를 기대할 수 있다. 
하지만 초당 `ServletThread` 수를 넘는 `RPS` 가 발생한다면 큐에 요청은 쌓이게 되고, 
큐에 대기하는 시간만큼 `Laytency` 는 늘어나게 된다. 
그리고 `Blocking` 이 요청처리 시간에 큰 비율을 차지한다면 전체적인 시스템 자원을 최대한 활용하지 못하므로, 
`Throughput` 또한 떨어지게 된다.  

정리하면 요청을 할당 받을 수 있는 `ServletThread` 가 더 이상 존재하지 않아 
요청은 큐에 쌓여 응답시간은 늘어나지만 정작 시스템 자원은 `Laytency` 가 늘어난 것에 비해 활용되지 못하고 있는 상황이 발생한다.  

그렇다면 `ServletThread` 수를 시스템이 허용가능한 수만큼 생성한다면 요청을 최대한으로 동시에 처리할 수 있다고 생각할 수 있다. 
하지만 시스템의 자원은 한정적이면서, `Thread` 수를 계속해서 늘리는 것만이 `Throughput` 을 향상시키는 방법은 아니다. 
시스템의 `core` 수보다 많은 `Thread` 를 사용하는 경우 `Context Switching` 이 발생하는데, 이는 비교적 비싼 비용이 드는 작업이다. 
그리고 `Thrad` 수가 `core` 수보다 아주 많다면 그 만큼 `Context Swiching` 이 발생하는 횟수는 급격하게 늘어나게 된다.  

[LinkedIn 의 사례](https://www.slideshare.net/brikis98/the-play-framework-at-linkedin/8-Thread_pool_usageLatencyThread_pool_hell) 
를 보면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-2.png)  

위 사진을 보면 `Thread Pool` 에서 감당할 수 있는 요청수가 들어올 때는 안정적인 `Laytency` 를 확인 할 수 있다. 
하지만 최대 요청수를 넘는 이후 부터 급격하게 `Laytency` 가 증가하는 것을 확인 할 수 있다.  

다양한 서비스 조합으로 구성되는 요즘의 서비스(`MSA`)에서 `Blocking I/O` 는 불가피한 동작이다. 
하지만 `DB`, `Network` 요청에 따른 `Blocking` 이 발생하는 `Thread` 는 점유가 된 상태이지만, 
응답을 대기만 할뿐 아무런 동작을 수행하지 않기 때문에 효율성 측면에서는 큰 병목지점이다.  

이러한 `Blocking` 에 따른 병목지점을 해소할 수 있는 방법이 바로 `Non-Blcoking I/O` 를 사용하는 것이다. 
그리고 `Spring` 에서는 `Spring WebFlux` 를 통해 `Spring MVC` 에서 발생하는 이러한 병목구간을 해소 할 수 있다.  


## Spring WebFlux
`Spring WebFlux` 는 `Spring 5.0` 버전에 추가됐다. 
왼전한 `Non-Blocking I/O` 이면서 `Reactive Streams back pressure` 를 지원한다. 
`Non-Blocking I/O` 와 `Reactive Streams`, `Event-Driven` 방식으로 `MSA` 와 같은 구조에서 발생하는 수많은 `Network I/O` 를 좀 더 효율적으로 제어할 수 있고, 
성능에도 좋은 영향을 미친다.  

물론 `Spring WebFlux` 만 사용한다고해서 이런 모든 장점을 바로 누릴 수 있는 것은 아니다. 
`WebFlux` 의 성능을 최대치로 끌어올리기 위해서는 요청 처리과정에 있는 모든 `I/O` 가 `Non-Bloking` 방식으로 동작되어야 한다.  
한 부분에서 `Blocking` 이 발생하게 되면 오히려 전체적인 요청처리에 더 안좋은 영향을 끼칠 수 있다. 
이는 요청에 처리에 필요한 `DB`, `REST API` 등 모든 처리가 `Non-Blocking` 이 필요하므로 각종 라이브러리의 지원도 필요하다.  

### Event Loop
`Spring WebFlux` 는 `Reactive Programming` 방식으로 적은 스레드를 사용해서 동시성을 높이는 방법을 사용한다. 
이러한 구성을 위해서는 `Event Loop` 라는 모델이 필요하다.  

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-3.png)  

- `event loop` 는 보통 시스템의 코어 수 만큼 존재한다. 
- `event loop` 는 순차적으로 `event queue` 의 `event` 를 처리한다. 
- `event loop` 에서 처리하는 `event` 동작은 `Intensive Operations`(구현 플랫폼) 에 `callback` 을 등록하고 바로 리턴 한다. 
- `event` 에 해당하는 `callback` 이 완료되면 `trigger` 를 발생 시키고 결과를 반환한다. 

이러한 `Event Loop` 모델은 `Nginx`, `Node.js` `Netty` 등 여러 플랫폼에서 구현되고 사용되고 있다.  

이렇게 `Event Loop` 방식으로 요청을 처리하기 때문에 다수의 요청을 더 적은 `Thread` 를 사용해서 처리할 수 있다. 


### Reactor Netty
`Spring WebFlux` 는 별도의 설정을 하지 않으면 `Reactor Netty Embedded Server` 를 사용한다. 
`core` 수가 4인 경우 `Reactor Netty` 가 실행되면 아래와 같은 스레드를 확인 할 수 있다. 

Thread Name|State|Type
---|---|---
server|WAITING|Normal
reactor-http-nio-1|RUNNABLE|Daemon
reactor-http-nio-2|RUNNABLE|Daemon
reactor-http-nio-3|RUNNABLE|Daemon
reactor-http-nio-4|RUNNABLE|Daemon

`Reactor Netty` 에서 생성하는 기본 서버 스레드를 제외하고, 
요청 처리를 위해 시스템 코어 수에 해당하는 `worker thread` 를 생성한다. 
그리고 하나의 `worker thread` 는 `Java NIO` 를 사용하는 `event loop` 역할을 한다.  

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-dr-3.png)  

`EventLoop` 에 대한 특징은 아래와 같다. 

- `EventLoop Group` 은 시스템 코어수 만큼의 `EventLoop` 를 관리 (커스텀 가능)
- `EventLoop` 의 이벤트/작업 처리 순서는 `FIFO` 방식
- `EventLoop` 는 변경되지 않는 하나의 `Thread` 에서 동작
- 모든 입출력 작업과 이벤트 처리는 `EventLoop` 에 할당된 `Thread` 에 의해 처리됨
- 실제 이벤트는 `Channel` 에서 수행됨
- 각각의 `EventLoop` 마다 이벤트 큐가 있음

`EventLoop` 마다 이벤트 큐를 가짐으로써, 이벤트 큐와 스레드를 1:1 관계로 만들어 하나의 `EventLoop` 에서 실행 순서 불일치 이슈를 해결한 구조이다.  


`EventLoop Group`, `Event Loop`, `Channel` 의 관계는 아래와 같다. 
- 하나의 `EventLoopGroup` 은 하나 이상의 `EventLoop` 를 포함
- `EventLoop Group` 은 `Channel` 에 `EventLoop` 를 할당
- 하나의 `EventLoop` 는 생명주기 동안 하나의 `Thread` 에 바인딩
- 하나의 `Channel` 은 생명주기 동안 하나의 `EventLoop` 에 등록
- 하나의 `EventLoop` 에는 하나 이상의 `Channel` 등록 가능


### Reactive API
`Spring WebFlux` 는 내부적으로 `Reactive Streams` 방식으로 웹 요청/응답 처리를 수행한다. 
사용하는 `Reactive Library` 는 [Reactive](https://github.com/reactor/reactor) 를 내부적으로 사용하고 있다. 
`Reactive` 과 관련된 내용은 
[Reactive Streams]({{site.baseurl}}{% link _posts/java/2021-04-18-java-concept-reactive-before-and-after.md %})
,
[Reactive Streams 활용]({{site.baseurl}}{% link _posts/java/2021-04-24-java-concept-ractivestreams-advanced.md %})  
에서 확인 할 수 있다.  

### Programming Model
`Spring WebFlux` 는 기존 `Spring MVC` 에서 사용할 수 있었던 `Annotated Controller` 외에도 `Functional Endpoints` 방식을 제공하고, 
이 두 방법은 모두 `Non-Blcoking` 으로 동작한다. 

- `Annotated Controllers` : `Spring MVC` 와 동일하고, `spring-web` 모듈에 있는 같은 `Annotation` 을 사용한다. 
`Spring WebFux` 에서 `Controller` 의 리턴값은 `Reactive`(`Reactor`, `RxJava`) 타입을 지원한다. 
- `Functional Endpoints` : `Labmda` 기반의 경량 함수형 프로그래밍 모델이다. 
애플리케이션이 요청을 라우팅하고 처리하는데 사용할 수 있는 작은 라이브러리라고 생각할 수 있다. 
`Annotated Controllers` 와 차이점은 `Annotation` 을 통해 요청을 정의하고 콜백을 받아 수행하는 방식이 아닌, 
애플리케이션이 `Annotation` 으로 정의하는 부분부터 콜백 처리까지 제어한다는 것이다.  

### Server
`Sping WebFlux` 는 `Tomcat`, `Jetty`, `Servlet 3.1+ Container` 와 `Servlet` 기반이 아닌 `Netty`, `Undertow` 에서도 동작 가능하다. 
`Spring Boot` 에서는 기본으로 `Netty` 서버를 사용하지만, `Maven`, `Gradle` 의존성 수정을 통해 나열된 서버중 하나로 교체할 수 있다.  

`Spring MVC`, `Spring WebFlux` 에서는 `Tomcat`, `Jetty` 서버를 모두 사용할 수 있다. 
하지만 실제 동작 방식의 차이점에 대해 주의가 필요하다. 
`Spring MVC` 는 `Servlet` 에서 `Blocking I/O` 를 사용하고, 애플리케이션에서 필요시 `Servlet API` 를 직접사용 할 수 있다. 
그에 비해 `Spring WebFlux` 는 `Servlet 3.1` 의 `Non-Blocking I/O` 를 사용하고, `Servlet API` 는 애플리케이션에서 사용할 수 없다. 
그리고 `Spring WebFlux` 에서 `Undertow` 를 사용하게 되면 `Servlet API` 가 아닌 `Undertow API` 를 사용한다.  

참고로 요청과 응답을 처리하는 `HttpHandler` 는 여러 `HTTP` 서버 `API` 를 추상화하는데 있다. 
지원하는 서버 `API` 는 아래 표와 같다. 

서버 이름|Server API|Reactive Streams 지원
---|---|---
Netty|Netty API|[Reactor Netty](https://github.com/reactor/reactor-netty)
Undertow|Undertow API|spring-web:Undertow to Reactive Streams Bridge
Tomcat|Servlet 3.1 Non-Blocking I/O<br>`ByteBuffer` 로 `byte[]` 를 읽고 쓰는 `Tomcat API`|spring-web: Servlet 3.1 Non-Blocking I/O to Reactive Streams Bridge
Jetty|Servlet 3.1 Non-Blocking I/O<br> `ByteBuffer` 로 `byte[]` 를 읽고 쓰는 `Jetty API`|spring-web: Servlet 3.1 Non-Blocking I/O to Reactive Streams Bridge
Servlet 3.1 Container|Servlet 3.1 Non-Blocking I/O|spring-web: Servlet 3.1 Non-Blocking I/O to Reactive Streams Bridge

 
### Spring MVC, Spring WebFlux
`Spring WebFlux` 는 기존 `Spring MVC` 에서 발생한 병목지점을 개선할 수 있는 스택이지만, 
그렇다고 해서 이후 웹 개발 부터는 모두 `Spring WebFlux` 를 사용해서 개발해야 되는 것은 아니다. 
`Spring MVC` 에 알맞는 애플리케이션이 있을 것이고, `Spring WebFlux` 에 알맞는 애플리케이션이 있을 것이기 때문에 서비스를 구성과 성격을 고려해서 알맞은 선택이 필요하다. 
위 두가지의 공통부분과 차이점을 다이어그램으로 비교하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-5.png)  


그리고 `Netty`, `Undertow`, `Servlet 3.1+` 컨테이너 서버에서 실행 가능하다. 
`Spring WebFlux` 가 추가되면서 `Netty Embedded Server` 도 함께 추가되었고, 기본 서버로 사용된다.  

`Spring WebFlux` 를 사용한다고 해서 기존 `Spring MVC` 방식으로 서비스 제공이 불가한 것은 아니다. 
`Spring WebFlux` 를 사용할지 `Spring MVC` 을 사용할지는 개발자의 선택과 요청 처리의 구성에 달려있다.  
그리고 `Spring WebFlux` 는 `Spring` 의 `reactive-stack` 웹 프레임워크이기 때문에 아래 그림을 통해 어떠한 스택으로 구성돼있는지 살펴본다.  


### Performance
`Spring WebFlux` 에 대해 알아보고 고민한다는 것은 서버 애플리케이션의 성능 향상에 목적이 있을 것이다. 
`Spring WebFlux` 를 사용하게 되면 기존 `Spring MVC` 방식에 비해서 성능이 향상되냐는 질문의 답은 `"No"` 이다. 
`Spring WebFlux` 의 주요 특징인 `Reactive Streams`, `Non-Blocking` 등은 애플리케이션의 성능을 향상 시켜주는 모델이나 메커니즘이 아니다. 
오히려 위와 같은 특징을 사용하기 위해 더 많은 작업이 필요할 수 있고, 이로 인해 처리시간이 늘어날 수도 있다.  

`Reactive Streams`, `Non-Blocking` 의 주요 이점은 고정된 스레드와 적은 메모리를 사용해서, 더 많은 요청을 효율적으로 처리할 수 있다는데 있다.  

#### Test
설명한 부분에 대해서 `Spring MVC`, `Spring WebFlux` 로 간단한 애플리케이션을 구성해서 테스트를 진행해본다. 
테스트는 `JMeter` 를 사용한다.  

##### Spring MVC Application
`Spring MVC` 에서 요청을 처리하는 `Thread` 수는 1개로 제한한다.  

- `build.gradle`

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.4.2'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
}

group 'com.windowforsun.spring'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.11

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
}
```  

- `MvcPerformanceApplication`

```java
@SpringBootApplication
@RestController
public class MvcPerformanceApplication {
    static {
        System.setProperty("server.tomcat.threads.max", "1");
    }

    @GetMapping("/mvc")
    public String mvc() throws Exception{
        return "success";
    }

    @GetMapping("/mvc/blocking")
    public String mvcB() throws Exception{
        Thread.sleep(100);
        return "success";
    }

    public static void main(String[] args) {
        SpringApplication.run(MvcPerformanceApplication.class, args);
    }
}
```  

##### Spring WebFlux Application
`Spring WebFlux` 애플리케이션도 요청을 처리하는 `Thread` 를 1개로 제한한다.  

- `build.gradle`

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.4.2'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
}

group 'com.windowforsun.spring'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.11

repositories {
    mavenCentral()
}

dependencies {
    implementation( 'org.springframework.boot:spring-boot-starter-webflux')

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'\
}
```  

- `WebFluxPerformanceApplication`

```java
@SpringBootApplication
@RestController
public class WebFluxPerformanceApplication {
    static {
        System.setProperty("reactor.netty.ioWorkerCount", "1");
    }

    @GetMapping("/webflux")
    public Mono<String> webflux() {
        return Mono.just("success");
    }

    @GetMapping("/webflux/blocking")
    public Mono<String> webfluxBlocking() throws Exception {
        return Mono.fromCallable(() -> {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return "success";
        });
    }


    public static void main(String[] args) {
        SpringApplication.run(WebFluxPerformanceApplication.class, args);
    }
}
```  

##### JMeter
`JMeter` 테스트 수행에 설정된 주요 설정 값은 아래와 같다.  

설정 이름|설정 값
---|---
`Thread Count`|200
`RPS`|10000
 

##### Non-Blocking
`Spring MVC`, `Spring WebFlux` 애플리케이션에서 요청 처리를 수행할때 문자열말 리턴하는 방식으로 테스트를 수행했다.  

`TPS` 에 대한 테스트 결과는 아래와 같다.  

- `Spring MVC`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-mvc-tps.png)  

- `Spring WebFlux`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-webflux-tps.png)  

`Spring MVC` 는 평균 `5000 TPS` 를 보여주고, `Spring WebFlux` 는 평균 `8000 TPS` 를 보여주고 있다. 
`Spring WebFlux` 가 `Spring MVC` 동일한 자원을 사용했을 때 확연히 더 높은 `TPS` 를 보여주는 것을 확인 할 수 있다.  

다음으로 응답시간에 대한 테스트 결과는 아래와 같다.  

- `Spring MVC`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-mvc-rtot.png)  

- `Spring WebFlux`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-webflux-rtot.png)  

`Spring MVC` 는 평균 `37ms` 를 보여주고, `Spring WebFlux` 는 평균 `20ms` 를 보여주고 있다. 
응답시간 역시 `TPS` 와 비슷하게 동일한 자원을 사용했을 때 더 빠른 응답시간을 보여주는 것을 확인 할 수 있다.  


##### Blocking
`Spring MVC`, `Spring WebFlux` 애플리케이션에서 요청 처리에 `100 millis` 정도 `Blocking` 이 발생하도록 구성해서 테스트를 수행했다.  

`TPS` 에 대한 테스트 결과는 아래와 같다.  

- `Spring MVC`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-mvc-blocking-tps.png)  

- `Spring WebFlux`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-webflux-blocking-tps.png)  

`Spring MVC`, `Spring WebFlux` 모두 비슷하게 평균 `10 TPS` 를 보여준다. 
요청처리에 `Blocking` 이 존재하는 상황에서는 두 애플리케이션에서 `TPS` 에 따른 큰 차이는 보이지 않는다.  

다음으로 응답시간에 대한 테스트 결과는 아래와 같다.  

- `Spring MVC`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-mvc-blocking-rtot.png)  

- `Spring WebFlux`

![그림 1]({{site.baseurl}}/img/spring/concept-webflux-test-webflux-blocking-rtot.png)  

`Spring MVC` 는 초반에는 큰 변동을 보여주다가 차츰 `20000ms` 로 수렴이 되는 모습을 보여준다. 
그에 비해 `Spring WebFlux` 는 절반 이상의 요청은 `Spring MVC` 보다 빠른 `16000ms` 이하 의 결과를 보여주지만, 
특정 주기마다 `60000ms` 까지 튀어 어느정도 유지되는 모습이다.  

`Spring WebFlux` 의 응답시간 결과를 보면 특정 주기마다 급격하게 응답시간이 느려지는 구간이 존재하고 반복되기 때문에, 
안정적이다라고 볼수는 없는 상황이다. 
이렇게 `Spring WebFlux` 를 사용할때 `Blocking` 이 존재한다면 오히려 `Spring MVC` 보다 전체적인 애플리케이션의 품질이 떨어질 수 있음을 확인 할 수 있다.  


---
## Reference
[Concurrency in Spring WebFlux](https://www.baeldung.com/spring-webflux-concurrency)  
[Netty 세미나](https://www.slideshare.net/JangHoon1/netty-92835335)  
[Web on Reactive Stack](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)  
[Guide to Spring 5 WebFlux](https://www.baeldung.com/spring-webflux)  
[Reactive programming with Spring Boot and Webflux](https://medium.com/@rarepopa_68087/reactive-programming-with-spring-boot-and-webflux-734086f8c8a5)  
[Chapter 7. EventLoop and threading model](https://livebook.manning.com/book/netty-in-action/chapter-7/)  
