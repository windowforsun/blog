--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Asynchronous Programming"
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
    - @Async
    - ThreadPoolTaskExecutor
    - AsyncResult
    - ListenableFuture
toc: true
use_math: true
---  

## Spring Asynchronous Programming
[Java Asynchronous Programming]({{site.baseurl}}{% link _posts/java/2021-05-02-java-concept-asynchronous-programming.md %})
에서는 `Java` 에서 비동기 프로그래밍을 할때 사용할 수 있는 `API` 에 대해서 알아보았다. 
`Spring` 에서는 `Java` 에서 제공하는 기능에서 더 편하고 더 다양한 비동기 프로그래밍 방식을 제공한다. 

기능|버전
---|---
ThreadPoolTaskExecutor|Spring 2.0
@Async|Spring 3.0
AsyncResult|Spring 3.0
ListenableFuture|Spring 4.0

<!--
DeferredResult|Spring 3.2
AsyncRestTemplate|Spring 4.0
ResponseBodyEmitter|Spring 4.2
-->

이번 포스트에서는 `Spring` 에서 비동기 프로그래밍을 위해 제공하는 기능들을 비지니스 로직 및 요청/응답 처리에서 
어떻게 사용하는지에 대해 간단하게 다뤄 본다. 

- `build.gradle`

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
    implementation 'org.springframework.boot:spring-boot-starter'

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
`Spring` 에서는 대부분 `Bean` 을 생성하고 빈 간의 의존성 주입을 통해 애플리케이션이 동작한다. 
이러한 방식에서 어떻게 비동기 프로그래밍을 구성하는지 알아보기 전에 이후 예제에서 기본이 될 수 있는 
동기 방식의 프로그래밍 예제를 먼저 살펴본다.  

```java
@Slf4j
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class SpringAsyncTest {
    public static Queue<String> resultQueue = new ConcurrentLinkedQueue<>();

    public static void printAndAddLog(String msg) {
        log.info(msg);
        resultQueue.add(msg);
    }

    @Slf4j
    @Service
    public static class MyService {
        public String doSyncSome() throws Exception {
            printAndAddLog("start SyncSome");
            String result = "result doSyncSome";
            Thread.sleep(100);
            printAndAddLog("end SyncSome");
            return result;
        }
    }


    @TestConfiguration
    public static class Config {
        @Bean
        public MyService myService() {
            return new MyService();
        }
    }

    @Autowired
    private MyService myService;

    @BeforeEach
    public void setUp() {
        resultQueue.clear();
    }

    @Test
    public void sync() throws Exception {
        // when
        printAndAddLog("start");
        printAndAddLog(this.myService.doSyncSome());
        printAndAddLog("end");

        // then
        assertThat(resultQueue, hasSize(5));
        assertThat(resultQueue, contains(
                "start",
                "start SyncSome",
                "end SyncSome",
                "result doSyncSome",
                "end"
        ));
    }
}
```  

```
INFO 10824 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 10824 --- [           main] c.w.r.springasync.SpringAsyncTest        : doing SyncSome
INFO 10824 --- [           main] c.w.r.springasync.SpringAsyncTest        : result doSyncSome
INFO 10824 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
```  

`printAndAddLog()` 메소드는 로그를 출력 및 결과 검증 큐에 넣는 작업을 수행한다. 
`MyService` 의 `doSyncSome()` 메소드는 100 밀리초 슬립 후 `printAndAddLog()` 메소드를 통해 로그 작업을 수행하고, 
메소드의 결과를 리턴한다. 
그리고 테스트 코드에서는 `doSyncSome()` 메소드의 호출 전후에 `start`, `end` 로그를 남긴다. 
결과를 살펴보면 호출 된 순서가 동기적으로 수행된 것을 확인 할 수 있다.  


### @Async
`Spring 3.0` 부터 제공하는 `@Async` 는 `Spring` 에서 가장 대표적인 비동기를 지원하는 기능이다. 
`Annotation` 을 활용하느 기능인 만큼 간편하게 특정 메소드를 비동기로 실행 될 수 있도록 만들 수 있다는 큰 장점이 있다. 
단순 메소드 위에 해당 `Annotation` 을 선언해 주기만하면 되고, 그 외 실제 비동기에 대한 동작은 `AOP` 를 활용해서 `Sping` 이 처리해 준다.  

하지만 `@Async` 는 `AOP` 를 활용한 기능인 만큼 `AOP` 가 가지는 제약사항을 `@Async` 가 모두 가지게 된다. 
이러한 제약 사항은 기본 모드 `Proxy` 시 대신 `AspectJ` 를 사용하는 방법으로 해결할 수 있다.  


> 위와 같은 제약 사항은 `@Async` 에서만 발생하는 것이 아닌, `@Transaction` 과 같이 `Spring` 에서 `Annotation` 을 사용하는 기능에서는 대부분 공통적으로 발생한다. 

위와 같은 제약사항을 해결하는 방법으로는 `@Async` 를 `AOP` 방식이 아닌 `AspectJ` 방식으로 이용하는 것이다.  

먼저 `@Async` 를 사용하기 위해서는 `@EnableAsync` 를 클래스 레벨에 선언해 주어야 한다. 
그리고 `@Async` 는 사용 가능한 여러 리턴 타입이 존재하는데 리턴 타입에 따라 구현이 상이할 수 있다. 
- `void`
- `Future`
- `CompletableFuture`
- `ListenableFuture`

#### Return void
`@Async` 를 사용할 때 리턴 타입이 `void` 처럼 없는 경우에 대해서 살펴본다. 
리턴 타입이 없는 경우 비동기 작업의 결과는 받을 수 없지만, `@Async` 가 선언된 메소드의 동작은 비동기적으로 수행된다.  


```java
@Slf4j
@Service
public static class MyService {
    @Async
    public void doAsync() throws Exception {
        printAndAddLog("start AsyncSome");
        Thread.sleep(100);
        printAndAddLog("end AsyncSome");
    }
}

@Test
public void async() throws Exception {
    // when
    printAndAddLog("start");
    this.myService.doAsync();
    printAndAddLog("end");

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncSome",
            "end AsyncSome"
    ));
}
```  

```
INFO 2740 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 2740 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 2740 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : start AsyncSome
INFO 2740 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : end AsyncSome
```  

`main` 스레드에서 동작과 `task-1` 이라는 스레드에서 동작하는 흐름으로 비동기 작업이 수해되는 것을 확인 할 수 있다. 
비동기 작업에서 100 밀리초 슬립이 있기 때문에, `main` 스레드는 먼저 동작을 마치게 되고 이후 100 밀리초 이후에 `task-1` 비동기 작업이 마무리 된다.  

#### Return Future
`@Async` 를 사용할 때 비동기 작업 결과를 전달 받을 수 방법 중 가장 기본적인 방법인 `Future` 에 대해서 알아본다. 
`@Async` 를 사용하는 메소드에서 `Future` 를 통해 작업 결과를 받을 때는 `AsyncResult` 라는 클래스를 통해 작업 결과를 리턴 해주면 된다. 
결과를 전달 받기 위해서는 기존 `Future` 사용법과 동일하게 `Future.get()` 을 사용하기 때문에 `Blocking` 은 불가피하게 발생한다. 

```java
@Slf4j
@Service
public static class MyService {

    @Async
    public Future<String> doAsyncFutureSome() throws Exception {
        printAndAddLog("start AsyncFutureSome");
        String result = "result doAsyncFutureSome";
        Thread.sleep(100);
        printAndAddLog("end AsyncFutureSome");
        return new AsyncResult<>(result);
    }
}

@Test
public void async_future() throws Exception {
    // when
    printAndAddLog("start");
    Future<String> future = this.myService.doAsyncFutureSome();
    printAndAddLog("end");
    boolean beforeBlocking = future.isDone();
    printAndAddLog(future.get());   // blocking
    boolean afterBlocking = future.isDone();

    // then
    assertThat(beforeBlocking, is(false));
    assertThat(afterBlocking, is(true));
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncFutureSome",
            "end AsyncFutureSome",
            "result doAsyncFutureSome"
    ));
}
```  

```
INFO 11272 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 11272 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 11272 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : start AsyncFutureSome
INFO 11272 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : end AsyncFutureSome
INFO 11272 --- [           main] c.w.r.springasync.SpringAsyncTest        : result doAsyncFutureSome
```  

`Return void` 일 때와 대부분 동작 흐름이 비슷하다. 
비동기 메소드에서는 작업의 결과를 `AsyncResult` 객체를 통해 리턴하고, `Future.get()` 메소드를 사용해서 `main` 스레드에서 결과를 받아 올 수 있다는 차이점이 있다. 
이는 비동기 동작은 별도의 스레드에서 비동기적으로 수행되지만, 작업의 결과를 받기 위해서는 비동기 작업이 마무리되고 리턴 값을 받아 올 수 있을 떄 까지 `Blocking` 이 발생한다.  

`Spring 3.0` 에 추가된 `AsyndResult` 는 `ListenableFuture` 의 구현체 클래스로(`Spring 4.1` 부터) `Future` 타입의 비동기 결과 처리를 처리 할수 있도록 하는 클래스이다.  

#### Return ListenableFuture
앞서 살펴본 리턴으로 `Future` 을 사용할 때 비동기 작업 결과는 받아 올 수 있지만, `Blocking` 을 기반으로 하기 때문에 효율성 측면에서는 좋지 않았다. 
`ListenableFuture` 를 사용하면 비동기 작업 결과 및 예외처리를 `Non-blocking` 방식으로 처리할 수 있다.  

`Spring 4.0` 에 추가된 `ListenableFuture` 내용은 아래와 같다. 

```java
public interface ListenableFuture<T> extends Future<T> {

	void addCallback(ListenableFutureCallback<? super T> callback);

	void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback);

	default CompletableFuture<T> completable() {
		CompletableFuture<T> completable = new DelegatingCompletableFuture<>(this);
		addCallback(completable::complete, completable::completeExceptionally);
		return completable;
	}

}
```  

`addCallback()` 메소드를 사용해서 결과 처리, 예외 처리에 대한 콜백 메소드를 전달하는 방식으로 비동기 동작 제어를 가능하게 해주는 `API` 이다.  

```java
@Slf4j
@Service
public static class MyService {
    @Async
    public ListenableFuture<String> doAsyncListenableFutureSome() throws Exception {
        printAndAddLog("start AsyncListenableFutureSome");
        String result = "result doAsyncListenableFutureSome";
        Thread.sleep(100);
        printAndAddLog("end AsyncListenableFutureSome");
        return new AsyncResult<>(result);
    }
}

@Test
public void async_listenableFuture() throws Exception {
    // when
    printAndAddLog("start");
    ListenableFuture<String> listenableFuture = this.myService.doAsyncListenableFutureSome();
    printAndAddLog("end");
    listenableFuture.addCallback(
            result -> {
                printAndAddLog(result);
            },
            error -> {
                printAndAddLog(error.getMessage());
            }
    );

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncListenableFutureSome",
            "end AsyncListenableFutureSome",
            "result doAsyncListenableFutureSome"
    ));
}
```  

```
INFO 11908 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 11908 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 11908 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : start AsyncListenableFutureSome
INFO 11908 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : end AsyncListenableFutureSome
INFO 11908 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : result doAsyncListenableFutureSome
```  

`Return Future` 일 때와 결과 및 동작이 거의 비슷하다. 
차이점은 앞서 언급한 것과 같이 `Blocking` 이 발생하지 않고, `ListenableFuture` 에 `Callback` 메소드를 등록해서 `Non-blocking` 하게 결과 처리 및 예외 처리를 수행한다는 점이다. 
그리고 결과를 처리하는 `Callback` 또한 비동기 작업으로 수행되기 때문에 `main` 스레드가 아닌 `task-1` 스레드에서 결과 처리가 수행된 것을 알 수 있다.  

 
#### return CompletableFuture
`CompletableFuture` 형태로 리턴하는 경우 비동기 결과에 대한 처리를 `Blocking`, `Non-blocking` 형태로 모두 처리할 수 있다. 
`AsyncResult.completable()` 메소드를 호출해서 비동기 결과를 리턴할 때 `Future` 와 같이 `get()` 메소드로 결과를 받아 온다면 `Blocking` 하게 처리 가능하다. 
`Non-blocking` 하게 비동기 결과를 처리하고 싶다면 `CompletableFuture` 에서 제공하는 `thenAccept()`, `thenApply()` 등과 같은 비동기 작업 콜백을 등록 하는 방법으로 가능하다.  


```java
@Slf4j
@Service
public static class MyService {

    @Async
    public CompletableFuture<String> doAsyncCompletableFutureSome() throws Exception {
        printAndAddLog("start AsyncCompletableFutureSome");
        String result = "result doAsyncCompletableFutureSome";
        Thread.sleep(100);
        printAndAddLog("end AsyncCompletableFutureSome");
        return new AsyncResult<>(result).completable();
    }
}

@Test
public void async_completableFuture() throws Exception {
    // when
    printAndAddLog("start");
    CompletableFuture<String> completableFuture = this.myService.doAsyncCompletableFutureSome();
    printAndAddLog("end");
    boolean beforeBlocking = completableFuture.isDone();
    printAndAddLog(completableFuture.get());   // blocking
    boolean afterBlocking = completableFuture.isDone();

    // then
    assertThat(beforeBlocking, is(false));
    assertThat(afterBlocking, is(true));
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncCompletableFutureSome",
            "end AsyncCompletableFutureSome",
            "result doAsyncCompletableFutureSome"
    ));
}

@Test
public void async_completableFutureNonBlocking() throws Exception {
    // when
    printAndAddLog("start");
    CompletableFuture<String> completableFuture = this.myService.doAsyncCompletableFutureSome();
    printAndAddLog("end");
    completableFuture.thenAccept((result) -> printAndAddLog(result));

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncCompletableFutureSome",
            "end AsyncCompletableFutureSome",
            "result doAsyncCompletableFutureSome"
    ));
}
```  

```java
INFO 20392 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 20392 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 20392 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : start AsyncCompletableFutureSome
INFO 20392 --- [         task-1] c.w.r.springasync.SpringAsyncTest        : end AsyncCompletableFutureSome
INFO 20392 --- [           main] c.w.r.springasync.SpringAsyncTest        : result doAsyncCompletableFutureSome
```  

두 작업 모두 결과는 동일하지만, `get()` 으로 비동기 작업의 결과를 받아온 경우 `Blocking` 이 발생한다. 
그리고 콜백 메소드를 등록하는 방법을 사용하는 경우 `Non-blocking` 하게 처리 된다는 점과 추가 비동기 작업 및 처리에 더욱 용이하다는 장점이 있다. 
또한 `thenAccept()` 메소드를 사용해서 비동기 결과 처리 `Callback` 을 등록 했기 때문에 `main` 스레드에서 수행되는 것을 확인할 수 있다. 
만약 결과 처리도 비동기로 수행하고 싶다면 `thenAcceptAsync()` 메소드를 사용해서 `Callback` 을 등록해 주면 된다.  

#### Custom Thread Pool 설정
`ThreadPoolTaskExecutor` 를 사용해서 빈으로 등록해 주면 `@Async` 가 선언된 비동기 동작마다 필요한 스레드 풀을 설정해서 사용할 수 있다. 
아래와 같이 기본으로 사용할 `executor` 빈과 `myPool` 이라는 `ThreadPoolTaskExecutor` 의 빈을 등록 했다.  

```java
@TestConfiguration
public static class Config {
    @Bean
    public Executor myPool() {
        ThreadPoolTaskExecutor pool = new ThreadPoolTaskExecutor();
        // 초기 스레드 수
        pool.setCorePoolSize(10);
        // 맥스 스레드 수
        pool.setMaxPoolSize(100);
        // 처리 대기열 크기
        pool.setQueueCapacity(50);
        // thread decorator
        pool.setThreadNamePrefix("myPool-");
        // 자동으로 호출해 줌
        pool.initialize();

        return pool;
    }

    @Bean
    public Executor taskExecutor() {
        return new ThreadPoolTaskExecutor();
    }
}
```  

그리고 `MyService` 에 아래와 같이 `@Async` 를 선언한 후 테스트에 따른 결과는 아래와 같다.  

```java
@Slf4j
@Service
public static class MyService {
    // 기본 스레드 풀 사용
    @Async
    public Future<String> doAsyncFutureSome() throws Exception {
        printAndAddLog("start AsyncFutureSome");
        String result = "result doAsyncFutureSome";
        Thread.sleep(100);
        printAndAddLog("end AsyncFutureSome");
        return new AsyncResult<>(result);
    }

    // myPool 이라는 커스텀 스레드 풀 사용
    @Async(value = "myPool")
    public Future<String> doAsyncFutureSomeMyPool() throws Exception {
        printAndAddLog("start AsyncFutureSomeMyPool");
        String result = "result doAsyncFutureSomeMyPool";
        Thread.sleep(100);
        printAndAddLog("end AsyncFutureSomeMyPool");
        return new AsyncResult<>(result);
    }
}

@Test
public void async_future() throws Exception {
    // when
    printAndAddLog("start");
    Future<String> future = this.myService.doAsyncFutureSome();
    printAndAddLog("end");
    boolean beforeBlocking = future.isDone();
    printAndAddLog(future.get());   // blocking
    boolean afterBlocking = future.isDone();

    // then
    assertThat(beforeBlocking, is(false));
    assertThat(afterBlocking, is(true));
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncFutureSome",
            "end AsyncFutureSome",
            "result doAsyncFutureSome"
    ));
}

@Test
public void async_future_myPool() throws Exception {
    // when
    printAndAddLog("start");
    Future<String> future = this.myService.doAsyncFutureSomeMyPool();
    printAndAddLog("end");
    boolean beforeBlocking = future.isDone();
    printAndAddLog(future.get());   // blocking
    boolean afterBlocking = future.isDone();

    // then
    assertThat(beforeBlocking, is(false));
    assertThat(afterBlocking, is(true));
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start AsyncFutureSomeMyPool",
            "end AsyncFutureSomeMyPool",
            "result doAsyncFutureSomeMyPool"
    ));
}
```  

`asyncFutureSome()` 메소드를 사용하는 `async_future()` 테스트의 결과는 아래와 같이 기본 스레드 풀을 사용해서 비동기 작업이 수행된다.  

```
INFO 22484 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 22484 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 22484 --- [ taskExecutor-1] c.w.r.springasync.SpringAsyncTest        : start AsyncFutureSome
INFO 22484 --- [ taskExecutor-1] c.w.r.springasync.SpringAsyncTest        : end AsyncFutureSome
INFO 22484 --- [           main] c.w.r.springasync.SpringAsyncTest        : result doAsyncFutureSome
```  

`asyncFutureSomeMyPool()` 메소드를 사용하는 `async_future_myPool()` 테스트 결과는 커스텀 하게 저의한 `myPool` 에 의해 비동기 작업이 수행된다.  

```
INFO 23184 --- [           main] c.w.r.springasync.SpringAsyncTest        : start
INFO 23184 --- [           main] c.w.r.springasync.SpringAsyncTest        : end
INFO 23184 --- [       myPool-1] c.w.r.springasync.SpringAsyncTest        : start AsyncFutureSomeMyPool
INFO 23184 --- [       myPool-1] c.w.r.springasync.SpringAsyncTest        : end AsyncFutureSomeMyPool
INFO 23184 --- [           main] c.w.r.springasync.SpringAsyncTest        : result doAsyncFutureSomeMyPool
```  

#### Properties 설정(application.yaml)
앞서 `@Async` 동작이 커스텀한 스레드 풀에 의해 수행될 수 있도록 하는 방법에 대해 알아보았다. 
별도의 스레드 풀을 지정하지 않아도 `Spring` 에서 기본으로 생성하는 `ThreadPoolTaskExecutor` 을 사용할 수 있다. 
[여기](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#spring.task.execution.pool.allow-core-thread-timeout) 
를 확인하면 `Properties` 를 사용해서 기본 스레드 풀을 설정할 수 있는 요소에 대해 확인 할 수 있다.  

아래는 `Properties` 설정으로 스레드 풀 이름, 스레드 개수, 큐 사이즈를 설정해서 진행한 테스트이다.  

```java
@Slf4j
@SpringBootTest(properties = {
        "spring.task.execution.thread-name-prefix=myDefaultPool-",
        "spring.task.execution.pool.core-size=2",
        "spring.task.execution.pool.max-size=4",
        "spring.task.execution.pool.queue-capacity=6"
})
@ExtendWith(SpringExtension.class)
@EnableAsync
public class SpringAsyncDefaultThreadPoolTest {
	
    @Slf4j
    @Service
    public static class MyService {
        @Async
        public void doAsync(int index) throws Exception {
            printAndAddLog("start AsyncSome : " + index);
            Thread.sleep(100);
            printAndAddLog("end AsyncSome : " + index);
        }
    }

    @Test
    public void async_property() throws Exception {
        // when
        int doCount = 10;
        printAndAddLog("start");
        for(int i = 1; i <= doCount; i++) {
            this.myService.doAsync(i);
        }
        printAndAddLog("end");

        // then
        Thread.sleep(500);
        assertThat(resultQueue, hasSize(22));
        assertThat(resultQueue, hasItems(
                "start",
                "end",
                "start AsyncSome : 1",
                "end AsyncSome : 1",
                "start AsyncSome : 10",
                "end AsyncSome : 10"
        ));
    }
}
```  

```
INFO 21920 --- [           main] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start
INFO 21920 --- [           main] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 2
INFO 21920 --- [myDefaultPool-4] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 10
INFO 21920 --- [myDefaultPool-3] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 9
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 1
INFO 21920 --- [myDefaultPool-4] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 10
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 1
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 4
INFO 21920 --- [myDefaultPool-4] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 3
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 2
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 5
INFO 21920 --- [myDefaultPool-3] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 9
INFO 21920 --- [myDefaultPool-3] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 6
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 4
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 5
INFO 21920 --- [myDefaultPool-4] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 3
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 7
INFO 21920 --- [myDefaultPool-3] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 6
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : start AsyncSome : 8
INFO 21920 --- [myDefaultPool-2] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 7
INFO 21920 --- [myDefaultPool-1] c.w.r.s.SpringAsyncDefaultThreadPoolTest : end AsyncSome : 8
```  

테스트에서 비동기 작업은 총 10개를 동시에 수행한다. 
하지만 설정된 최대 스레드 수는 4개이므로 최대 4개의 스레드만 사용하고, 
스레드에 할당 받지 못한 작업은 큐에서 대기하게 된다. 
그리고 스레드 풀 이름도 설정한 이름으로 적용 된 것을 확인 할 수 있다.  


### @Async AspectJ
`@EnableAsync` 를 선언을 함으로써 `@Async` 기능을 사용할 수 있다. 
별도의 설정을 하지 않으면 `AdviceMode` 는 `PROXY` 모드로 동작한다. 

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
    // stub

    AdviceMode mode() default AdviceMode.PROXY;	

    // stub
}
```  

`AdviceMode` 는 `PROXY`, `ASPECTJ` 모드를 선택할 수 있다.  

```java
public enum AdviceMode {

	/**
	 * JDK proxy-based advice.
	 */
	PROXY,

	/**
	 * AspectJ weaving-based advice.
	 */
	ASPECTJ

}
```  

앞서 언근했던 것처럼 기본 모드인 `Proxy` 는 `@Async` 기능 사용에 있어 제약사항이 있다. 
`Proxy` 모드는 `Spring AOP` 의 기본 모드로 순수 `Java` 코드를 사용해서 `AOP` 를 구현하기 때문이다. 
비동기 처리의 관점(`Aspect`) 를 특정 메소드와 엮는(`Weaving`)이 코드가 실행되는 시점에 이뤄 진다. 
이러한 방식을 `RTW`(`Run-Time Weaving`)이라고 하고, 
특정 메소드가 호출되는 런타임 시기에 해당 메소드의 비동기 처리 여부가 결정되는 방식이다. 
이러한 특징으로 아래와 같은 제약 사항이 있다. 

- `public` 메소드만 가능
- 같은 객체내의 메소드 끼리 호출시 동작 불가능(`self-invocation`)
- `RTW` 로 수행되기 때문에 약간의 성능저하

이런 제약사항을 해결하기 위해서는 `AspectJ` 모드를 설정해 주면 된다. 
`AspectJ` 는 `Spring` 과 별도의 라이브러리이다. 
`Java` 언어를 사용한 코드라면 `Spring Frameowkr` 의 `AOP` 기능을 사용하지 않아도, 
`AspectJ` 를 사용해서 `AOP` 구성이 가능하다. 
`AspectJ` 는 아래와 같은 컴파일 과정을 통해 구성된다. 
1. `Java` 코드를 컴파일해서 `binary` 형태의 `class` 파일 만들기
1. `binary` 파일을 분석해서 필요한 부분에 `Aspect` 를 삽입하는 `Weaving` 과정 수행

즉 이러한 과정으로 `AspectJ` 는 `CTW`(`Compile-Time Weaving`) 으로 처리된다는 것을 알 수 있다.  


---
## Reference
[Creating Asynchronous Methods](https://spring.io/guides/gs/async-method/)  
[Effective Advice on Spring Async: Part 1](https://dzone.com/articles/effective-advice-on-spring-async-part-1)  