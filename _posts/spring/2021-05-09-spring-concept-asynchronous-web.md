--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Asynchronous Web"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 비동기 프로그래밍을 할때 사용 할 수 있는 API 와 기능에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Async
    - ThreadPoolTaskExecutor
    - AsyncResult
    - ListenableFuture
toc: true
use_math: true
---  

## Spring Asynchronous Web
[Java Asynchronous Programming]({{site.baseurl}}{% link _posts/java/2021-05-02-java-concept-asynchronous-programming.md %})
에서는 `Java` 에서 비동기 프로그래밍을 할때 사용할 수 있는 `API` 에 대해서 알아보았고, 
[Spring Asynchronous Programming]({{site.baseurl}}{% link _posts/spring/2021-05-05-spring-concept-asynchronous-programming.md %})
에서는 `Spring` 에서 지원해주는 비동기 기능에 대해서 알아 보았다. 
`Spring` 을 사용하면 다양한 종류의 애플리케이션을 개발할 수 있지만, 
대부분의 경우 `Web` 애플리케이션을 구현을 위해 사용된다. 
이번 포스트에서는 `Spring Web` 애플리케이션을 개발할때 지원하는 비동기 방식에 대해서 알아본다.  

기능|버전
---|---
DeferredResult|Spring 3.2
ResponseBodyEmitter|Spring 4.2

위 표에 있는 새로운 `API` 에 대한 설명도 있겠지만, 
`Java`, `Spring` 에서 살펴본 비동기 기술을 활용해서 이를 웹 애플리케이션에 직접 적용 해보는 설명이 주를 이룰 것 같다.  


`Spring Boot` 은 기본으로 컴파일된 `Web Server` 를 제공한다. 
작성한 애플리케이션 코드는 자동으로 `Web Server` 에 의해 동작 되기 때문에 별도로 빌드 결과물을 `Web Server` 에 올리는 작업은 하지 않아도 웹 애플리케이션을 구동 할 수 있다. 
그리고 `Spring Boot` 가 기본으로 제공하는 `Web Server` 는 `Tomcat` 으로 `spring-boot-starter-web` 의존성에 `spring-boot-starter-tomcat` 의존성이 포함돼 있는 것을 확인 할 수 있다.  

`Tomcat` 에서 웹 통신은 `Servlet API` 를 사용해서 이뤄지는데, `3.0` 버전을 기점으로 비동기 지원을 시작했다. 
- `Servlet 3.0`
  - 비동기 서블릿 지원
  - `HTTP connection` 은 이미 `NIO` 를 통해 구현됨
  - `Servlet` 요청 읽기, 응답 쓰기는 `Blocking`
  - 비동기 작업 시작 즉시 `Servlet Thread` 반납
  - 비동기 작업 완료시 `Servlet Thread` 재 할당 후 응답
  - 비동기 서블릿 컨텍스트 이용(`AsyncContext`)
- `Servlet 3.1`
  - `Non-Blocking` 서블릿 요청, 응답 처리 가능
  - `Callback` 활용

테스트 프로젝트에서 사용한 `build.gradle` 파일 내요은 아래와 같다. 

```groovy
plugins {
    id 'org.springframework.boot' version '2.4.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group 'com.windowforsun.spring'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.11

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testCompileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testAnnotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'

}
```  


### Synchronous
먼저 동기식 요청을 받아 처리하는 컨트롤러 클래스의 내용은 아래와 같다.  

```java
@RestController
@Slf4j
public static class MyController {
    @Autowired
    private ExternalService externalService;

    @GetMapping("/sync/{i}")
    public String sync(@PathVariable int i) throws Exception {
        String result = "result sync " + i;
        printAndAddLog("start sync " + i);
        Thread.sleep(100);
        printAndAddLog("end sync " + i);

        return result;
    }
}
```  

요청 처리는 `Blocking` 동작이 있다고 가정하고 100 밀리초 정도 대기후 처리된다.  

그리고 테스트에서 부하 테스트를 위한 `loadTest` 메소드의 내용은 아래와 같다.  

```java
static AtomicInteger count = new AtomicInteger(0);

public long loadTest(String url, int threadCount) throws Exception {

    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("loadtest-");
    ExecutorService es = Executors.newFixedThreadPool(threadCount, threadFactory);
    RestTemplate restTemplate = new RestTemplate();

    count.set(0);
    StopWatch mainSw = new StopWatch();
    mainSw.start();

    for (int i = 1; i <= threadCount; i++) {
        es.execute(() -> {
            int idx = count.addAndGet(1);

            StopWatch subSw = new StopWatch();
            subSw.start();

            String result = restTemplate.getForObject(url + "/" + idx, String.class);
            subSw.stop();
            log.info("Thread-{}, Elapsed : {} millis, result : {}", idx, subSw.getTotalTimeMillis(), result);
            printAndAddLog(result);
        });
    }

    es.shutdown();
    es.awaitTermination(2000, TimeUnit.SECONDS);
    mainSw.stop();
    long totalElapsedMillis = mainSw.getTotalTimeMillis();
    log.info("total Elapsed : {} millis", totalElapsedMillis);

    return totalElapsedMillis;
}
```  

`loadTest` 는 `threadCount` 만큼 스레드 풀을 생성하고 `url` 에 동시에 요청을 수행하는 역할을 한다. 
그리고 테스트를 수행하는데 걸린 총 시간(`millis`)를 결과로 리턴한다.  

테스트를 수행하는 코드 내용은 아래와 같다.  

```java
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = {
                "server.tomcat.threads.max=1",
        }

)
@ExtendWith(SpringExtension.class)
@EnableAsync
@Slf4j
public class SpringAsyncWebTest {
    public static Deque<String> resultQueue = new ConcurrentLinkedDeque<>();

    public static void printAndAddLog(String msg) {
        log.info(msg);
        resultQueue.add(msg);
    }

    @TestConfiguration
    public static class Config {
        @Bean
        public MyController myController() {
            return new MyController();
        }
    }

    @LocalServerPort
    private int port;
    private String serverUrl;
    private RestTemplate restTemplate;

    @BeforeEach
    public void setUp() {
        this.restTemplate = new RestTemplate();
        this.serverUrl = "http://localhost:" + this.port;
        resultQueue.clear();
        log.info("serverUrl : {}", this.serverUrl);
    }

    @Test
    public void sync() throws Exception {
        // when
        long actual = this.loadTest(this.serverUrl + "/sync", 10);

        // then
        assertThat(actual, allOf(greaterThan(1000L), lessThan(2000L)));
        assertThat(resultQueue, hasSize(30));
        assertThat(resultQueue, everyItem(containsString("sync")));
        assertThat(resultQueue, hasItems(
                "start sync 1",
                "end sync 1",
                "result sync 1",
                "start sync 10",
                "end sync 10",
                "result sync 10"
        ));
    }
}
```  

테스트에서 먼저 눈여겨 봐야 할 부분은 `Properties` 설정을 통해, 
`server.tomcat.threads.max=1` 으로 `Tomcat` 에서 사용 가능한 `ServletThread` 의 수를 1개로 제한 한 상태로 이후 테스트까지 모두 진행된다.  

테스트 코드를 보면 동시에 10개의 요청을 `/sync` 경로로 요청한다. 
`Tomcat` 은 `Servlet Thread` 를 1개만 사용 가능하고, 요청 1개를 처리하는데 최소 100 밀리초가 걸린다. 
이를 통해 계산해 보면 전체 요청이 처리되기까지 최소 1초에서 최대 2초 정도 소요됨을 알 수 있다.  

실제로 테스트를 수행하며 출력된 로그는 아래와 같다. 
웹통신을 처리하는 `o-auto-1-exec-1` 라는 스레드 하나가 모든 요청을 처리하고, 
`loadtest-*` 라는 10개의 스레드가 요청을 수행하고 응답 결과 로그를 찍는 것을 확인 할 수 있다.   

```
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 6
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 6
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 4
INFO 9024 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 348 millis, result : result sync 6
INFO 9024 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 6
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 4
INFO 9024 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 455 millis, result : result sync 4
INFO 9024 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 4
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 8
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 8
INFO 9024 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 562 millis, result : result sync 8
INFO 9024 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 8
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 2
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 2
INFO 9024 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 671 millis, result : result sync 2
INFO 9024 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 2
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 1
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 1
INFO 9024 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 779 millis, result : result sync 1
INFO 9024 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 1
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 3
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 3
INFO 9024 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 889 millis, result : result sync 3
INFO 9024 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 3
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 7
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 7
INFO 9024 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 996 millis, result : result sync 7
INFO 9024 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 7
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 5
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 5
INFO 9024 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 1105 millis, result : result sync 5
INFO 9024 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 5
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 10
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 10
INFO 9024 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 1213 millis, result : result sync 10
INFO 9024 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 10
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start sync 9
INFO 9024 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end sync 9
INFO 9024 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 1324 millis, result : result sync 9
INFO 9024 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result sync 9
INFO 9024 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 1336 millis
```  

>`Tomcat` 스레드 수를 1개로 고정한 이유는 극한의 상황을 가정하기 위함이다. 
>가용할 수 있는 모든 스레드가 요청 처리를 위해 할당된 상태에서 계속해서 요청이 들어온다면, 
>이후 요청들은 큐에 쌓이게 되면서 메모리 사용량도 증가하게 될 것이다. 
>그리고 큐에 쌓이게 되면서 대기시간도 계속해서 증가하는 상황이 벌어지게 된다. 
>이러한 상태에서 비동기 기법을 최대한 활용해 적은 리소스를 사용하면서 더 많은 요청을 처리하는 방법에 대해 알아본다. 

### Callable
`Tomcat` 스레드가 1개일 때 10개의 요청을 한번에 처리 가능한 방법 중하나는 `Controller` 에서 `Callable` 을 리턴하는 것이다. 
`Callabele` 을 리턴한다는 것은 앞서 설명한 `Servlet 3.0` 의 방식을 사용하는 방법으로 
전체적인 흐름을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/practice_asychronous_web_1.png)

1. 요청 받은 `Servlet Thread` 는 `Callable` 객체를 바로 리턴한다.
1. `Spring MVC` 내부에서 리턴 받은 `Callable` 객체를 스레드 풀에 전달한다. 
1. `Servlet Thread` 는 반납돼었기 때문에 `DispatcherServlet`, `Filter` 등은 `Servlet Thread` 에서 벗어나지만 요청/응답 커넥션은 열려 있는 상태이다. 
  - 요청에서 벗어난 `Servlet Thread` 는 다른 요청을 받아 처리 할 수 있는 상태이다. 
1. `Callable` 은 결과를 생성하고 `Spring MVC` 는 `Servlet Thread` 에게 재요청을 진행한다. 
1. `DispatcherServlet` 은 `Callable` 에게서 반환된 결과를 가지고 다시 이어 작업을 진행한다.    

요청을 받은 `Servlet Thread` 는 요청 작업을 처리하는 `Worker`(`Callable`)를 등록만 하고 바로 다시 반납된다. 
그리고 `Worker` 가 자신의 모든 작업을 완료하면 다시 `Server Thread` 를 할당 받아 응답을 전달하는 방식을 사용한다. 
이러한 방법을 사용하면 `Tomcat` 스레드는 1개이지만 동시에 많은 요청을 좀 더 효율적으로 처리할 수 있다.  

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/callable/{i}")
    public Callable<String> callableAsync(@PathVariable int i) throws Exception {
        printAndAddLog("start callableAsync " + i);
        return () -> {
            String result = "result callableAsync " + i;
            Thread.sleep(100);
            printAndAddLog("end callableAsync " + i);

            return result;
        };
    }
}

@Test
public void callableAsync() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/callable");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, everyItem(containsString("callableAsync")));
    assertThat(resultQueue, hasItems(
            "start callableAsync 1",
            "end callableAsync 1",
            "result callableAsync 1",
            "start callableAsync 10",
            "end callableAsync 10",
            "result callableAsync 10"
    ));
}
```  

```
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 7
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 4
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 9
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 6
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 2
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 5
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 3
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 10
INFO 16864 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start callableAsync 1
INFO 16864 --- [      MvcAsync3] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 4
INFO 16864 --- [      MvcAsync4] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 9
INFO 16864 --- [      MvcAsync1] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 8
INFO 16864 --- [      MvcAsync2] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 7
INFO 16864 --- [      MvcAsync5] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 6
INFO 16864 --- [     MvcAsync10] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 1
INFO 16864 --- [      MvcAsync8] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 3
INFO 16864 --- [      MvcAsync6] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 2
INFO 16864 --- [      MvcAsync7] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 5
INFO 16864 --- [      MvcAsync9] c.w.r.springasyncweb.SpringAsyncWebTest  : end callableAsync 10
INFO 16864 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 338 millis, result : result callableAsync 8
INFO 16864 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 8
INFO 16864 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 340 millis, result : result callableAsync 4
INFO 16864 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 4
INFO 16864 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 342 millis, result : result callableAsync 7
INFO 16864 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 7
INFO 16864 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 344 millis, result : result callableAsync 9
INFO 16864 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 9
INFO 16864 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 346 millis, result : result callableAsync 1
INFO 16864 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 1
INFO 16864 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 348 millis, result : result callableAsync 6
INFO 16864 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 6
INFO 16864 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 350 millis, result : result callableAsync 2
INFO 16864 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 2
INFO 16864 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 352 millis, result : result callableAsync 5
INFO 16864 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 5
INFO 16864 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 354 millis, result : result callableAsync 3
INFO 16864 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 3
INFO 16864 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 355 millis, result : result callableAsync 10
INFO 16864 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result callableAsync 10
INFO 16864 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 357 millis
```  

`Tomcat` 스레드 1개로 동시에 10개의 요청을 처리하는데 300 밀리초 정도 밖에 걸리지 않았고, 
동기 방식을 테스트한 로그와 동일하게 `Tomcat` 스레드는 한개 밖에 사용되지 않았다.  

하지만 주의해야 하는 부분은 `end callableAsync` 로그가 찍힌 `MvcAsync` 스레드이다. 
동시에 100 밀리초가 걸리는 요청 10개를 처리하기 위해 `Tomcat` 스레드는 1개이지만 대신 
`Callable` 이 수행되는 별도의 스레드 풀에서 10개의 스레드가 동작한 것이다.  

앞서 살펴본 동기 방식에서 `Tomcat` 스레드 수를 10개로 늘린 것과 자원의 효율적 측면으로 본다면 
더 안좋다고 볼 수도 있는 방법이다. 
`Tomcat` 스레드 수를 그냥 10개로 늘린다면 총 10개의 스레드로 동시에 요청 처리가 가능하지만, 
현재 `Callable` 을 사용한 방법은 `Tomcat` 스레드 1개를 포함해서 총 11개의 스레드가 동작 된 것으로 볼 수 있기 때문이다.  

### DeferredResult
practice_asychronous_web_2

### Emitter

### WebClient(skip)


### ListenableFuture
controller 에서 그냥 외부 api 비동기 결과 리턴
listenablefuture 에서 callback 으로 결과 처리는 결과처리에 대한 결과를 다시 리턴 받을 수 없음


### DiferredResult
controller 에서 외부 api 비동기 결과 가공 후 리턴 
callback hell 까지

### DeferredResult Callback hell 개선 ? Completion
callback hell 해결 라이브러리 구현

### CompletableFuture ?
java 에서 공식 지원하는 비동기 작업 파이프라이닝 라이브러리 사용

---
## Reference
[DeferredResult (Spring Framework 5.3.6 API)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html)  