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
WebAsyncTask|Spring 3.2
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
웹통신을 처리하는 `ServletThread` 의 이름인 `o-auto-1-exec-1` 라는 스레드 하나가 모든 요청을 처리하고, 
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

앞서 살펴본 동기 방식에서 `Tomcat` 스레드 수를 10개로 늘린 것과 자원의 효율 측면에서 본다면 
더 안좋다고 볼 수도 있는 방법이다. 
`Tomcat` 스레드 수를 그냥 10개로 늘린다면 총 10개의 스레드로 동시에 요청 처리가 가능하지만, 
현재 `Callable` 을 사용한 방법은 `Tomcat` 스레드 1개를 포함해서 총 11개의 스레드가 동작 된 것으로 볼 수 있기 때문이다.  


### WebAsyncTask
`Spring 3.2` 부터 사용할 수 있는 `WebAsyncTask` 는 `Callable` 를 사용한 비동기 처리보다 
좀 더 다양한 설정으로 비동기 웹 통신을 구성할 수 있는 클래스이다. 
아래와 같이 타임아웃, 에러 처리, `Executor` 등 생성자로 설정을 할 수 있다. 
그리고 `WebAsyncTask` 는 내부적으로 `Callable` 을 사용해서 비동기 처리를 수행한다.  

```java
public class WebAsyncTask<V> implements BeanFactoryAware {

	private final Callable<V> callable;

	private Long timeout;

	private AsyncTaskExecutor executor;

	private String executorName;

	private BeanFactory beanFactory;

	private Callable<V> timeoutCallback;

	private Callable<V> errorCallback;

	private Runnable completionCallback;

	public WebAsyncTask(Callable<V> callable) {
		Assert.notNull(callable, "Callable must not be null");
		this.callable = callable;
	}

	public WebAsyncTask(long timeout, Callable<V> callable) {
		this(callable);
		this.timeout = timeout;
	}

	public WebAsyncTask(@Nullable Long timeout, String executorName, Callable<V> callable) {
		this(callable);
		Assert.notNull(executorName, "Executor name must not be null");
		this.executorName = executorName;
		this.timeout = timeout;
	}

	public WebAsyncTask(@Nullable Long timeout, AsyncTaskExecutor executor, Callable<V> callable) {
		this(callable);
		Assert.notNull(executor, "Executor must not be null");
		this.executor = executor;
		this.timeout = timeout;
	}
}
```  

위와 같은 특징으로 `Callable` 을 직접 사용하기보다는 `WebAsyncTask` 를 사용하는 것을 권정한다.  

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/webAsyncTask/{i}")
    public WebAsyncTask<String> webAsyncTaskAsync(@PathVariable int i) throws Exception {
        printAndAddLog("start webAsyncTaskAsync " + i);
    
        return new WebAsyncTask<>(1000L, () -> {
            String result = "result webAsyncTaskAsync " + i;
            Thread.sleep(100);
            printAndAddLog("end webAsyncTaskAsync " + i);
    
            return result;
        });
    }
}


@Test
public void webAsyncTaskAsync() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/webAsyncTask");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, everyItem(containsString("webAsyncTaskAsync")));
    assertThat(resultQueue, hasItems(
            "start webAsyncTaskAsync 1",
            "end webAsyncTaskAsync 1",
            "result webAsyncTaskAsync 1",
            "start webAsyncTaskAsync 10",
            "end webAsyncTaskAsync 10",
            "result webAsyncTaskAsync 10"
    ));
}
```  

```
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 1
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 4
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 7
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 10
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 3
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 5
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 2
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 6
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 8
INFO 7520 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start webAsyncTaskAsync 9
INFO 7520 --- [      MvcAsync2] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 4
INFO 7520 --- [      MvcAsync1] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 1
INFO 7520 --- [      MvcAsync7] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 2
INFO 7520 --- [      MvcAsync8] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 6
INFO 7520 --- [      MvcAsync9] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 8
INFO 7520 --- [      MvcAsync4] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 10
INFO 7520 --- [      MvcAsync3] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 7
INFO 7520 --- [      MvcAsync6] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 5
INFO 7520 --- [      MvcAsync5] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 3
INFO 7520 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 330 millis, result : result webAsyncTaskAsync 4
INFO 7520 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 331 millis, result : result webAsyncTaskAsync 1
INFO 7520 --- [     MvcAsync10] c.w.r.springasyncweb.SpringAsyncWebTest  : end webAsyncTaskAsync 9
INFO 7520 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 1
INFO 7520 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 4
INFO 7520 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 334 millis, result : result webAsyncTaskAsync 2
INFO 7520 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 2
INFO 7520 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 335 millis, result : result webAsyncTaskAsync 6
INFO 7520 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 6
INFO 7520 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 337 millis, result : result webAsyncTaskAsync 8
INFO 7520 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 8
INFO 7520 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 339 millis, result : result webAsyncTaskAsync 10
INFO 7520 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 10
INFO 7520 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 341 millis, result : result webAsyncTaskAsync 7
INFO 7520 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 7
INFO 7520 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 343 millis, result : result webAsyncTaskAsync 5
INFO 7520 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 5
INFO 7520 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 345 millis, result : result webAsyncTaskAsync 3
INFO 7520 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 3
INFO 7520 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 346 millis, result : result webAsyncTaskAsync 9
INFO 7520 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result webAsyncTaskAsync 9
INFO 7520 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 348 millis
```  

로그와 실제 동작은 `Callable` 을 사용했을 떄와 크게 다르지 않다. 
`Servlet Thread` 는 `o-auto-1-exec-1` 이름으로 하나만 사용됐지만, 
`Servlet Thread` 에서 리턴하는 비동기 작업 처리를 위해 여러개 `MvcAsync` 이름의 스레드가 사용되었다.  


### DeferredResult
`Callable`, `WebAsyncTask` 이외에 비동기 웹 처리를 사용할 수 있는 방법중 하나가 바로 `DeferredResult` 이다. 
`Spring 3.2` 부터 사용가능한 `DeferredResult` 는 `Callable` 처럼 다른 스레드로 부터 생산된 값을 반환하는 방식은 동일하다. 
하지만 해당 스레드가 `Spring MVC` 에서 관리되는 것은 아니다.  

```java
public class DeferredResult<T> {
	public DeferredResult() {
		this(null, () -> RESULT_NONE);
	}

	public DeferredResult(Long timeoutValue) {
		this(timeoutValue, () -> RESULT_NONE);
	}

	public DeferredResult(@Nullable Long timeoutValue, Object timeoutResult) {
		this.timeoutValue = timeoutValue;
		this.timeoutResult = () -> timeoutResult;
	}

	public DeferredResult(@Nullable Long timeoutValue, Supplier<?> timeoutResult) {
		this.timeoutValue = timeoutValue;
		this.timeoutResult = timeoutResult;
	}
}
```

`DeferredResult` 는 `Long polling` 방식이라고도 불리는데, 
외부 이벤트로 부터 생산된 응답을 반환해야 하는 경우에 사용할 수 있다. 
특정 작업이 진행되거나 이벤트가 완료된 후 요청 했었던 클라이언트에게 값을 전달 할 수 있다. 
이를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/practice_asychronous_web_2.png)  

1. 요청 받은 `Servlet Thread`(`Controller`)는 새로 생성한 `DeferedResult` 객체를 큐에 넣고 바로 리턴한다. 
1. 이후 아무 시점에 이벤트에 해당하는 작업을 수행한다. (비동기)
1. `Servlet Thread` 는 요청에서 벗어산 상태이지만 `Response` 는 열린 상태로 대기한다. 
1. 이벤트에 해당하는 작업이 완료되면, 큐에 넣었던 `DeferredResult` 에 작업 결과를 설정한다. 
1. `Spring MVC` 는 해당 요청에 대한 결과를 `Servlet Thread` 에게 다시 전달 한다. 
1. 이후 `Servlet Thread` 는 전달 받은 비동기 결과를 응답 작업을 진행 한다.  


```java
@RestController
@Slf4j
public static class MyController {
    
    // 클라이언트 요청에 대한 DeferedResult 생성 및 큐에 등록
    @GetMapping("/async/deferred/{i}")
    public DeferredResult<String> deferredAsync(@PathVariable int i) throws Exception {
        printAndAddLog("start deferredAsync " + i);
        DeferredResult<String> result = new DeferredResult<>(6000000L);
        deferredMap.put(i, result);
        printAndAddLog("end deferredAsync " + i);

        return result;
    }

    // 현재 생성된 DeferredResult 총 개수
    @GetMapping("/async/deferred/count")
    public int deferredCount() {
        return deferredMap.size();
    }
    
    // 비동기 이벤트 처리 완료(가정)후, DeferredResult 에 결과 등록
    @GetMapping("/async/deferred/event/{msg}")
    public void deferredEvent(@PathVariable String msg) {
        for (int key : deferredMap.keySet()) {
            DeferredResult<String> dr = deferredMap.get(key);
            dr.setResult("result deferredAsync " + msg + " " + key);
            deferredMap.remove(dr);
        }
    }
}

@Test
public void deferredResult() throws Exception {
    // given
    ExecutorService es = Executors.newSingleThreadExecutor();
    Future<Long> loadTester = es.submit(() ->  this.loadTest(this.serverUrl + "/async/deferred"));
    Thread.sleep(100);

    // when
    int deferredCount = this.restTemplate.getForObject(this.serverUrl + "/async/deferred/count", Integer.class);
    this.restTemplate.getForObject(this.serverUrl + "/async/deferred/event/end!", String.class);
    long actual = loadTester.get();

    // then
    assertThat(actual, allOf(greaterThan(100L), lessThan(400L)));
    assertThat(deferredCount, is(10));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, everyItem(containsString("deferredAsync")));
}
```  

```
.. DeferredResult 등록 요청 ..
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 6
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 6
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 8
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 8
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 1
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 1
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 4
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 4
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 5
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 5
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 3
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 3
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 10
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 10
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 7
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 7
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 2
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 2
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start deferredAsync 9
INFO 4664 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end deferredAsync 9
.. DeferredResult 이벤트 완료 요청 ..
INFO 4664 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 300 millis, result : result deferredAsync end! 1
INFO 4664 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 1
INFO 4664 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 302 millis, result : result deferredAsync end! 2
INFO 4664 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 2
INFO 4664 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 305 millis, result : result deferredAsync end! 3
INFO 4664 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 3
INFO 4664 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 307 millis, result : result deferredAsync end! 4
INFO 4664 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 4
INFO 4664 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 310 millis, result : result deferredAsync end! 5
INFO 4664 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 5
INFO 4664 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 313 millis, result : result deferredAsync end! 6
INFO 4664 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 6
INFO 4664 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 316 millis, result : result deferredAsync end! 7
INFO 4664 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 7
INFO 4664 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 318 millis, result : result deferredAsync end! 8
INFO 4664 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 8
INFO 4664 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 320 millis, result : result deferredAsync end! 9
INFO 4664 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 9
INFO 4664 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 323 millis, result : result deferredAsync end! 10
INFO 4664 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result deferredAsync end! 10
INFO 4664 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 325 millis
```  

`Servlet Thread` 인 `o-auto-1-exec-1` 는 `/async/deferred` 10번의 요청을 받아 10개의 `DeferredResult` 를 큐에 넣고 바로 리턴한다. 
`/async/deferred` 로 요청한 클라이언트는 자신의 요청에 리턴된 `DeferredResult` 에 결과값이 설정 될때 까지 대기 상태에 빠지고, 
최대 대기시간은 `DeferredResult` 에 설정된 타임아웃 시간이다. 
그리고 `/async/deferred/count` 를 호출해서 현재 `DeferredResult` 의 개수를 가져오면 `10` 개임을 알 수 있다. 
이후 `/async/deferred/event` 로 `end!` 라는 메시지와 함께 요청을 보내면 대기 중이였던 모든 클라이언트는 응답을 받게 된다.  

이런 방식은 앞서 언급했던것 처럼 `Long polling` 으로 클라이언트 서버로 요청을 보낸 후, 서버가 응답을 줄떄까지 대기를 하는 방식이다. 
그리고 서버는 대기 중인 클라이언트의 요청 처리를 위해 특정 스레드가 `Blocking` 상태에 빠지지 않고, 
비동기 방식으로 이후 이벤트되고 `DeferredResult` 에 결과값이 설정 됐을 떄 다시 `Servlet Thread` 가 결과 값 응답을 위해 사용된다.  

### ResponseBodyEmitter
`Spring 4.2` 에 추가된 `ResponseBodyEmitter` 는 기존 `HTTP` 요청과 응답이 `1:1` 관계인것과 달리, 
1개의 요청에 대해 1개 이상의 응답을 `Streaming` 방식으로 수행할 수 있다.  

```java
public class ResponseBodyEmitter {

	@Nullable
	private final Long timeout;

	@Nullable
	private Handler handler;

	private final Set<DataWithMediaType> earlySendAttempts = new LinkedHashSet<>(8);

	private boolean complete;

	@Nullable
	private Throwable failure;

	private boolean sendFailed;

	private final DefaultCallback timeoutCallback = new DefaultCallback();

	private final ErrorCallback errorCallback = new ErrorCallback();

	private final DefaultCallback completionCallback = new DefaultCallback();

	public ResponseBodyEmitter() {
		this.timeout = null;
	}

	public ResponseBodyEmitter(Long timeout) {
		this.timeout = timeout;
	}
}
```  

`ResponseBodyEmitter.send()` 를 통해 스트리밍으로 결과를 전달하고, 완료되면 `ResponseBodyEmitter.complete()` 메소드를 호출해 주면된다.  

테스트 코드는 아래와 같다.  

```java
@RestController
@Slf4j
public static class MyController {

    @GetMapping("/async/emitter/{count}/{delay}")
    public ResponseBodyEmitter emitterAsync(@PathVariable int count, @PathVariable long delay) {
        final int emittercount = count;
        ResponseBodyEmitter emitter = new ResponseBodyEmitter();

        Executors.newSingleThreadExecutor().submit(() -> {
            for (int i = 1; i <= emittercount; i++) {
                String result = "stream : " + i + ", " + System.currentTimeMillis() + "\n";
                printAndAddLog("start emitter " + i);
                emitter.send(result);
                printAndAddLog("end emitter " + i);
                Thread.sleep(delay);
            }

            emitter.complete();

            return null;
        });

        return emitter;
    }
}

@Test
public void emitter() throws Exception {
    // given
    StopWatch sw = new StopWatch();

    // when
    sw.start();
    String actual = this.restTemplate.getForObject(this.serverUrl + "/async/emitter/10/100", String.class);
    sw.stop();
    long millis = sw.getTotalTimeMillis();

    // then
    assertThat(millis, allOf(greaterThan(1000L), lessThan(1500L)));
    String[] strs = actual.split("\n");
    assertThat(strs, arrayContaining(
            startsWith("stream : 1"),
            startsWith("stream : 2"),
            startsWith("stream : 3"),
            startsWith("stream : 4"),
            startsWith("stream : 5"),
            startsWith("stream : 6"),
            startsWith("stream : 7"),
            startsWith("stream : 8"),
            startsWith("stream : 9"),
            startsWith("stream : 10")
    ));
}
```  

```
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 1
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 1
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 2
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 2
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 3
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 3
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 4
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 4
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 5
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 5
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 6
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 6
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 7
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 7
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 8
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 8
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 9
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 9
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start emitter 10
INFO 15380 --- [pool-1-thread-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end emitter 10
```  

아래는 웹 브라우저에서 `/async/emitter/100/200` 로 호출 했을 때의 결과이다.  

![그림 1]({{site.baseurl}}/img/spring/practice-asychronous-web-1.gif)   


### ListenableFuture
`Spring 4.0` 에 추가된 `ListenableFuture`는 비동기 동작을 콜백으로 제어하는 특징을 가지고 있다. 
`Spring MVC` 의 `Controller` 에서 `ListenableFuture` 를 리턴을 하면, 
`Servlet 3.1` 의 방식과 같이 콜백 방식으로 `Non-blocking` 으로 요청을 처리할 수 있다.  

먼저 테스트를 위해 외부 서비스를 호출해서 결과를 제공하는 `ExternalService` 가 있다. 
외부 서비스 요청과 응답 처리에 따른 `Blocking` 은 `sleep(100)` 을 통해 표현한다. 
`ExternalService` 의 메소드들은 `AsyncRestTemplate`, `WebClient` 와 같이 비동기 웹 요청을 수행할 수 있는 라이브러리들을 
사용해서 외부 `API`를 호출해서 결과를 리턴하는 동작을 수행한다고 가정한다. 
그중 `requestToAsyncListenableFuture`  메소드는 `ListenableFuture` 방식으로 `Blocking` 처리에 대한 결과를 리턴하는 메소드이다.  

```java
@Slf4j
@Service
public static class ExternalService {
    @Async
    public ListenableFuture<String> requestToAsyncListenableFuture(String param, int seq) {
        String result = param + "/resultListenableFuture-" + seq;

        if(param.equals("error")) {
            throw new RuntimeException("this is error");
        }

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return new AsyncResult<>(result);
    }
}
```  


```java
@RestController
@Slf4j
public static class MyController {

    @Autowired
    private ExternalService externalService;

    @GetMapping("/async/external/listenable/{i}")
    public ListenableFuture<String> asyncExternalServiceListenable(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        ListenableFuture<String> serviceResult = this.externalService.requestToAsyncListenableFuture(result, 1);
        printAndAddLog("end " + i);

        return serviceResult;
    }
}


@Test
public void asyncExternalServiceListenable() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/listenable");

    // then
    Thread.sleep(1000L);
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "result 1/resultListenableFuture-1",
            "result 2/resultListenableFuture-1",
            "result 3/resultListenableFuture-1",
            "result 4/resultListenableFuture-1",
            "result 5/resultListenableFuture-1",
            "result 6/resultListenableFuture-1",
            "result 7/resultListenableFuture-1",
            "result 8/resultListenableFuture-1",
            "result 9/resultListenableFuture-1",
            "result 10/resultListenableFuture-1"
    ));
}
```  

```
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 9
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 9
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 6
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 6
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 7
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 7
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 3
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 3
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 8
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 8
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 5
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 5
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 10
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 10
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 2
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 2
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 1
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 1
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 4
INFO 6000 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 4
INFO 6000 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 354 millis, result : result 6/resultListenableFuture-1
INFO 6000 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result 6/resultListenableFuture-1
INFO 6000 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 354 millis, result : result 9/resultListenableFuture-1
INFO 6000 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result 9/resultListenableFuture-1
INFO 6000 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 358 millis, result : result 3/resultListenableFuture-1
INFO 6000 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result 3/resultListenableFuture-1
INFO 6000 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 359 millis, result : result 7/resultListenableFuture-1
INFO 6000 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result 7/resultListenableFuture-1
INFO 6000 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 361 millis, result : result 8/resultListenableFuture-1
INFO 6000 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result 8/resultListenableFuture-1
INFO 6000 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 364 millis, result : result 2/resultListenableFuture-1
INFO 6000 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result 2/resultListenableFuture-1
INFO 6000 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 366 millis, result : result 4/resultListenableFuture-1
INFO 6000 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result 4/resultListenableFuture-1
INFO 6000 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 367 millis, result : result 10/resultListenableFuture-1
INFO 6000 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result 10/resultListenableFuture-1
INFO 6000 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 370 millis, result : result 5/resultListenableFuture-1
INFO 6000 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result 5/resultListenableFuture-1
INFO 6000 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 373 millis, result : result 1/resultListenableFuture-1
INFO 6000 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result 1/resultListenableFuture-1
INFO 6000 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 374 millis
```  

로그와 테스트 결과를 보면 10개의 요청을 `Servlet Thread` 1개가 처리하는데 소요된 시간이 `375` 밀리초 인것을 확인 할 수 있고, 
이는 모든 요청이 `Non-blocking` 으로 처리 됐음을 보여 준다.  
테스트 수행에 사용된 스레드는 `Servlet Thread` 인 `o-auto-1-exec-1` 스레드만 사용 됐다. 
(스레드 이름이 `loadtest` 인 것은 테스트 요청을 수행하기 위해 사용된 스레드이다.) 


### DeferredResult + ListenableFuture
`Spring MVC Controller` 에서 `ListenableFuture` 만 사용했을 때 모든 처리를 `Non-blocking` 하기 처리 할 수 있었다.  
하지만 `ListenableFuture` 는 비동기 동작의 결과를 받기 위해서는 `Blocking` 불가피 하게 수행되는 점이 있다. 
그래서 결과를 받아 가공하기 위해서는 `ListenableFuture.addCallback()` 메소드를 사용해서 결과 처리 콜백 메소드를 등록해야 하고, 
콜백 결과를 응답의 결과를 사용하기 위해서는 `DeferredResult` 를 사용하면 된다. 

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/external/deferred/listenable/{i}")
    public DeferredResult<String> asyncExternalServiceDeferredListenable(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        DeferredResult<String> dr = new DeferredResult<>();
        ListenableFuture<String> serviceResult = this.externalService.requestToAsyncListenableFuture(result, 1);
        serviceResult.addCallback(
                (r) -> {
                    dr.setResult(r + "~");
                },
                (e) -> {
                    dr.setErrorResult(e.getMessage());
                }
        );
        printAndAddLog("end " + i);
    
        return dr;
    }
}

@Test
public void asyncExternalServiceDeferredListenable() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/deferred/listenable");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "result 1/resultListenableFuture-1~",
            "result 2/resultListenableFuture-1~",
            "result 3/resultListenableFuture-1~",
            "result 4/resultListenableFuture-1~",
            "result 5/resultListenableFuture-1~",
            "result 6/resultListenableFuture-1~",
            "result 7/resultListenableFuture-1~",
            "result 8/resultListenableFuture-1~",
            "result 9/resultListenableFuture-1~",
            "result 10/resultListenableFuture-1~"
    ));
}
```  

```
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 8
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 8
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 1
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 1
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 2
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 2
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 10
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 10
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 9
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 9
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 6
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 6
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 4
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 4
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 5
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 5
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 7
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 7
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 3
INFO 11072 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 3
INFO 11072 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 362 millis, result : result 1/resultListenableFuture-1~
INFO 11072 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 361 millis, result : result 8/resultListenableFuture-1~
INFO 11072 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result 8/resultListenableFuture-1~
INFO 11072 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result 1/resultListenableFuture-1~
INFO 11072 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 365 millis, result : result 10/resultListenableFuture-1~
INFO 11072 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result 10/resultListenableFuture-1~
INFO 11072 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 367 millis, result : result 9/resultListenableFuture-1~
INFO 11072 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result 9/resultListenableFuture-1~
INFO 11072 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 370 millis, result : result 6/resultListenableFuture-1~
INFO 11072 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result 6/resultListenableFuture-1~
INFO 11072 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 372 millis, result : result 4/resultListenableFuture-1~
INFO 11072 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result 4/resultListenableFuture-1~
INFO 11072 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 374 millis, result : result 2/resultListenableFuture-1~
INFO 11072 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result 2/resultListenableFuture-1~
INFO 11072 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 375 millis, result : result 7/resultListenableFuture-1~
INFO 11072 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result 7/resultListenableFuture-1~
INFO 11072 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 378 millis, result : result 5/resultListenableFuture-1~
INFO 11072 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result 5/resultListenableFuture-1~
INFO 11072 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 380 millis, result : result 3/resultListenableFuture-1~
INFO 11072 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result 3/resultListenableFuture-1~
INFO 11072 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 381 millis
```  

결과를 보면 `ListenableFuture` 를 사용했을 때와 동일하게 모든 요청이 `Non-blocking` 하게 처리 됨을 확인 할 수 있고, 
`Controller` 에서 비동기 작업에 대한 결과 처리 또한 정상적으로 수행된 것을 확인 할 수 있다.  


하지만 만약 `ListenableFuture` 의 동작간에 의존성(A의 결과를 통해 B를 요청)이 있다거나, 
계속해서 비동기 작업 결과에 대한 가공이 필요한 상황이 발생한다면 아래와 같이 `Callback Hell` 이 발생한다.  

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/external/deferred/listenable3/{i}")
    public DeferredResult<String> asyncExternalServiceDeferredListenable3(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        DeferredResult<String> dr = new DeferredResult<>();
        ListenableFuture<String> serviceResult = this.externalService.requestToAsyncListenableFuture(result, 1);
        serviceResult.addCallback(
                (r) -> {
                    ListenableFuture<String> serviceResult2 = this.externalService.requestToAsyncListenableFuture(r, 2);
                    serviceResult2.addCallback(
                            (r2) -> {
                                ListenableFuture<String> serviceResult3 = this.externalService.requestToAsyncListenableFuture(r2, 3);
                                serviceResult3.addCallback(
                                        (r3) -> {
                                            dr.setResult(r3);
                                        },
                                        (e3) -> {
                                           dr.setErrorResult(e3.getMessage());
                                        }
                                );
                            },
                            (e2) -> {
                                dr.setErrorResult(e2.getMessage());
                            }
                    );
                },
                (e) -> {
                    dr.setErrorResult(e.getMessage());
                }
        );
        printAndAddLog("end " + i);
    
        return dr;
    }
}

@Test
public void asyncExternalServiceDeferredListenable3_callbackhell() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/deferred/listenable3");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "result 1/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 2/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 3/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 4/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 5/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 6/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 7/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 8/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 9/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 10/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3"
    ));
}
```  


### DeferredResult + CompletableFuture
`CompletableFuture` 는 `Java 1.8` 에서 추가된 비동기 작업의 흐름을 효율적으로 구성할 수 있는 `API` 이다. 
`Spring MVC Controller` 에서 사용하게 되면 `ListenableFuture` 에서 발생했던 `Callback Hell` 문제를 효과적으로 해결 할 수 있다.  

테스트를 위해 `CompletableFuture` 를 결과로 리턴하는 `ExeternalService` 메소드를 추가한다. 
`requestToAsyncCompletableFuture` 는 `ListenableFuture` 를 결과로 리턴하는 `requestToAsyncListenableFuture` 과 
수행하는 동작은 같지만 리턴 타입에만 차이가 있다.  

```java
@Slf4j
@Service
public static class ExternalService {
    @Async
    public CompletableFuture<String> requestToAsyncCompletableFuture(String param, int seq) throws RuntimeException {
        String result = param + "/requestToAsyncCompletableFuture-" + seq;

        if(param.equals("error")) {
            throw new RuntimeException("this is error");
        }

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return CompletableFuture.supplyAsync(() -> result);
    }
}
```  

위 메소드를 사용하는 `Controller` 와 테스트 코드를 작성하면 아래와 같다.  

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/external/deferred/completable/{i}")
    public DeferredResult<String> asyncExternalServiceDeferredCompletable(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        DeferredResult<String> dr = new DeferredResult<>();

        this.externalService.requestToAsyncCompletableFuture(result, 1)
                .thenCompose(r -> this.externalService.requestToAsyncCompletableFuture(r, 2))
                .thenCompose(r -> this.externalService.requestToAsyncCompletableFuture(r, 3))
                .thenAccept(r -> dr.setResult(r));

        printAndAddLog("end " + i);

        return dr;
    }
}

@Test
public void asyncExternalServiceDeferredCompletable() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/deferred/completable");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "result 1/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 2/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 3/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 4/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 5/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 6/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 7/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 8/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 9/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3",
            "result 10/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3"
    ));
}
```  

```
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 2
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 2
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 4
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 4
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 10
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 10
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 7
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 7
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 8
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 8
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 9
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 9
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 6
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 6
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 1
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 1
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 3
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 3
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 5
INFO 11384 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 5
INFO 11384 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 555 millis, result : result 4/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : result 4/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 557 millis, result : result 2/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : result 2/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 559 millis, result : result 9/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : result 9/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 561 millis, result : result 8/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : result 8/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 564 millis, result : result 6/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : result 6/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 565 millis, result : result 10/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : result 10/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 568 millis, result : result 7/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : result 7/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 570 millis, result : result 5/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : result 5/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 573 millis, result : result 3/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : result 3/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 575 millis, result : result 1/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : result 1/requestToAsyncCompletableFuture-1/requestToAsyncCompletableFuture-2/requestToAsyncCompletableFuture-3
INFO 11384 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 576 millis
```  

모든 요청을 `Servlet Thread` 1개를 사용해서 빠른 시간내에 처리하면서, 
`Callback Hell` 이 발생하지 않으면서 비동기 작업에 대한 흐름을 구성할 수 있다.  

아래는 비동기 작업 도중 에러가 발생했을 때는 에러 처리를 수행하고 싶은 비동기 작업 다음에 `CompletableFuture.exceptionally()` 메소드로 
에러 처리에 대한 작업을 정의해주면 된다.  

```java
@RestController
@Slf4j
public static class MyController {
    @GetMapping("/async/external/deferred/completable/error/{i}")
    public DeferredResult<String> asyncExternalServiceDeferredCompletableError(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        DeferredResult<String> dr = new DeferredResult<>();

        this.externalService.requestToAsyncCompletableFuture("error", 1)
                .thenCompose(r -> this.externalService.requestToAsyncCompletableFuture(r, 2))
                .thenCompose(r -> this.externalService.requestToAsyncCompletableFuture(r, 3))
                .thenAccept(r -> dr.setResult(r))
                .exceptionally(e -> {
                    dr.setErrorResult(e.getMessage());
                    return null;
                });

        printAndAddLog("end " + i);

        return dr;
    }
}

@Test
public void asyncExternalServiceDeferredCompletableError() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/deferred/completable/error");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "java.lang.RuntimeException: this is error"
    ));
}
```  

```
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 9
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 9
INFO 11120 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-9, Elapsed : 252 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 8
INFO 11120 --- [     loadtest-9] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 8
INFO 11120 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-8, Elapsed : 257 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-8] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 3
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 3
INFO 11120 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-3, Elapsed : 262 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-3] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 7
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 7
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 10
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 10
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 1
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 1
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 2
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 2
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 4
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 4
INFO 11120 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-4, Elapsed : 279 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-4] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 5
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 5
INFO 11120 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-5, Elapsed : 284 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-5] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : start 6
INFO 11120 --- [o-auto-1-exec-1] c.w.r.springasyncweb.SpringAsyncWebTest  : end 6
INFO 11120 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-6, Elapsed : 291 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-6] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-7, Elapsed : 294 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-7] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-10, Elapsed : 296 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [    loadtest-10] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-1, Elapsed : 301 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-1] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : Thread-2, Elapsed : 304 millis, result : java.lang.RuntimeException: this is error
INFO 11120 --- [     loadtest-2] c.w.r.springasyncweb.SpringAsyncWebTest  : java.lang.RuntimeException: this is error
INFO 11120 --- [           main] c.w.r.springasyncweb.SpringAsyncWebTest  : total Elapsed : 305 millis
```  

다음으로 만약 비동기 작업이 `ListenableFuture` 를 리턴하는 상황에서 `CompletableFuture` 를 사용하는 방법은 아래와 같다.  

```java
@RestController
@Slf4j
public static class MyController {
    <T> CompletableFuture<T> lfToCf(ListenableFuture<T> lf) {
        CompletableFuture<T> cf = new CompletableFuture<>();
        lf.addCallback(
                r -> cf.complete(r),
                e -> cf.completeExceptionally(e)
        );

        return cf;
    }

    @GetMapping("/async/external/deferred/listenable/completable/{i}")
    public DeferredResult<String> asyncExternalServiceDeferredListenableCompletable(@PathVariable int i) throws Exception {
        String result = "result " + i;
        printAndAddLog("start " + i);
        DeferredResult<String> dr = new DeferredResult<>();

        lfToCf(this.externalService.requestToAsyncListenableFuture(result, 1))
                .thenCompose(r -> lfToCf(this.externalService.requestToAsyncListenableFuture(r, 2)))
                .thenCompose(r -> lfToCf(this.externalService.requestToAsyncListenableFuture(r, 3)))
                .thenAccept(r -> dr.setResult(r))
                .exceptionally(e -> {
                    dr.setErrorResult(e.getMessage());
                    return null;
                });

        printAndAddLog("end " + i);

        return dr;
    }
}

@Test
public void asyncExternalServiceDeferredListenableCompletable() throws Exception {
    // when
    long actual = this.loadTest(this.serverUrl + "/async/external/deferred/listenable/completable");

    // then
    assertThat(actual, allOf(greaterThan(200L), lessThan(800L)));
    assertThat(resultQueue, hasSize(30));
    assertThat(resultQueue, hasItems(
            "result 1/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 2/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 3/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 4/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 5/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 6/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 7/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 8/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 9/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3",
            "result 10/resultListenableFuture-1/resultListenableFuture-2/resultListenableFuture-3"
    ));
}
```  


---
## Reference
[DeferredResult (Spring Framework 5.3.6 API)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html)  
[Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async)  