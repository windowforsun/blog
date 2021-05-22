--- 
layout: single
classes: wide
title: "[Java 개념] Java Asynchronous Programming"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java 에서 비동기 프로그래밍을 할때 사용 할 수 있는 API 에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Asynchronous
    - Multi Thread
    - Concurrent
    - Future
    - FutureTask
    - Executor
    - Runnable
    - Thread
    - CountDownLatch
    - CyclicBarrier
    - CompletableFuture
toc: true
use_math: true
---  

## Asynchronous Programming
오늘날 비동기(`Asynchronous`) 프로그래밍이 각광 받고 대중화 된 이유에 대해서 생각해보면, 
이전과 비교해서 급격하게 늘어난 사용자와 하나의 서비스를 구성하는 아키텍쳐의 변화에 있다고 생각한다. 
서비스 처리를 하다보면 중간에 오래 걸리는 연산이 수행되는 경우가 빈번하게 발생한다. 
`DB`, `API` (`Blocking IO`)등의 작업이 대부분일 것이다. 이러한 부분을 좀더 효율적으로 처리할 수 있도록 하는 것이 바로 
비동기 기법을 통해 동시성을 늘리는 방법이다.  

`Java` 에서는 이러한 비동기 처리를 위해 초기 버전부터 시작해서 다양한 `API` 를 제공해오고 있다.  

Class|Java Version
---|---
Thread|1.0
Runnable|1.0
Callable|1.5
Executor|1.5
Future|1.5
FutureTask|1.5
CountDownLatch|1.5
CyclicBarrier|1.5
Semaphore|1.5
RunnableFuture|1.6
CompletableFuture|1.8
CompletionStage|1.8


이번 포스트에서는 `Java` 비동기 동작 구성이 필요할 때 사용할 수 있는 `API` 대해 간략하게 알아 본다.  

### Synchronous
먼저 동기적(`Synchronous`) 흐름을 가지는 부분에서 오래걸리는 작업이 중간에 발생할 떄의 상황을 살펴 본다. 
여기서 동기적이라는 의미는 모든 처리가 동일한 스레드에서 모두 처리 된다고 할 수 있기 때문에 처리 순서에 대해 보장이 가능하다.  

```java
@Slf4j
public class JavaAsyncTest {
    public static Queue<String> resultQueue = new ConcurrentLinkedQueue<>();

    public static void printAndAddLog(String msg) {
        log.info(msg);
        resultQueue.add(msg);
    }

    @BeforeEach
    public void setUp() {
        resultQueue.clear();
    }

    @Test
    public void sync() throws Exception {
        // given
        Function runnable = (n) -> {
            try {
                printAndAddLog("start runnable");
                Thread.sleep(100);
                printAndAddLog("end runnable");
            } catch(Exception e) {
                e.printStackTrace();
            }

            return null;
        };

        // when
        printAndAddLog("start");
        runnable.apply(null);
        printAndAddLog("end");

        // then
        assertThat(resultQueue, hasSize(4));
        assertThat(resultQueue, contains(
                "start",
                "start runnable",
                "end runnable",
                "end"
        ));
    }
}
```  

`printAndAddLog()` 메소드는 로그를 찍고, 호출 순서를 검증하기 위해 로그를 큐에 넣는 동작을 수행한다. 
이후 진행되는 테스트는 모두 위 클래스에서 테스트 메소드를 추가하는 방법으로 진행된다.  

테스트 코드는 `sync()` 메소드에 작성돼 있다. 
오래 걸리는 작업은 `Function` 인터페이스의 구현체인 `runnable` 에서 0.1초 슬립으로 구현했다. 
실행 로그를 보면 아래와 같이 모드 `main` 스레드에서 순서대로 실행된 것을 확인 할 수 있다.  

```java
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
```  

### Thread, Runnable
`Java 1.0` 부터 존재하던 비동기 처리를 위한 기능이다. 
가장 상위 인터페이스인 `Runnable` 부터 살펴 보면 아래와 같다.  

```java
@FunctionalInterface
public interface Runnable {
    /**
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```  

`run()` 메소드를 통해 수행하고 싶은 비동기 작업을 구현해 주면된다. 
그리고 `Runnable` 의 구현체를 비동기로 동작시키기 위해서는 `Thread` 클래스가 필요하다.  

```java
public class Thread implements Runnable {
	public Thread() {
        this(null, null, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(Runnable target) {
        this(null, target, "Thread-" + nextThreadNum(), 0);
    }

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
    
    // stub
}
```  

`Thread` 클래스를 상속후 `run()` 메소드를 `Override` 해서 비동기 작업에 대한 구현을 진행할 수 있고, 
`Thread` 생성자에 `Runnable` 의 구현체를 전달하는 방식으로 사용할 수 있다. 
즉 `Thread` 와 `Runnable` 은 하나의 비동기 작업을 지칭하는 단위로도 볼수 있고, 하나의 스레드라고 봐도 무방하다.  

`Thread` 와 `Runnable` 을 사용해서 `Synchronous` 의 테스트를 동일하게 구성하면 아래와 같다.  

```java

@Test
public void thread_runnable() throws Exception {
    // given
    Runnable runnable = () -> {
      try {
        printAndAddLog("start runnable");
        Thread.sleep(100);
        printAndAddLog("end runnable");
      } catch(Exception e) {
          e.printStackTrace();
      }
    };
    Thread thread = new Thread(runnable, "MyThread");

    // when
    printAndAddLog("start");
    thread.start();
    printAndAddLog("end");
    thread.join();

    // then
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start runnable",
            "end runnable"
    ));
}
```  

테스트 코드를 보면 `thread.join()` 메소드를 통해 해당 스레드가 종료될 때까지 대기하도록 했다. 
`Thread.sleep(200)` 과 같이 해도 결과는 동일하다. 

출력된 로그를 살펴보면 아래와 같이 `main` 스레드와 함께 테스트 코드에서 생성한 `MyThread` 가 존재하고, 
`Runnable` 에 정의된 비동기 작업의 코드는 `MyThread` 에서 수행된 것을 확인 할 수 있다. 
그리고 동기 흐름과는 달리 2개의(`main`, `MyThread`) 독립된 작업 흐름이기 때문에 `main` 스레드에서 `Runnable` 의 비동기 작업 코드를 완료 할때까지 대기하거나 하는 상황은 발새하지 않는다. 
`main` 스레드는 `MyThread` 에 `Runnable` 의 작업을 맡기기만 한 후 바로 그 다음으로 넘어가는 흐름을 보이고 있다.  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[MyThread] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[MyThread] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
```  

### Executor
`Java 1.5` 에 등장한 `Executor` 는 비동기 작업을 좀더 효율적으로 관리할 수 있는 기능을 제공한다. 
`Executor` 는 하나의 `Thread Pool` 이라고 생각하면 간단하다. 
`Executor` 는 인터페이스로 함께 구성되는 다이어그램을 그려보면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept-asynchronous-programming-1.png)  

`Executors` 에서 정적 메소드를 사용하면 간단하게 필요한 `Thread Pool` 을 생성할 수 있다.  

```java
// 크기가 1인 Thread pool 생성 
ExecutorService singlePool = Executors.newSingleThreadExecutor();
// 크기가 4인 Thread pool 생성
ExecutorService pool4 = Executors.newFixedThreadPool(4);
// 필요할 때마다 스레드를 생성하고, 이미 생성된 스레드는 재사용 가능
ExecutorService cachedPool = Executors.newCachedThreadPool();
```  

`Executors` 의 정적 메소드를 통해 스레드 풀을 생성할 때 종류에 따른 설명은 아래와 같다. 
- `newSingleThreadExecutor` : 단일 스레드로 동작하는 `Executor` 이다. 
- `newFixedThreadPool` : 최대 스레드 수를 인자 값으로 받고, 작업이 등록 될때 마다 하나씩 스레드를 생성한다. 
그 수는 최대 스레드 수를 넘지 않는다. 
- `newCachedThreadPool` : 최대 스레드의 수의 제한을 두지 않는다. 대신 현재 생성한 스레드 수가 처리해야하는 작업보다 많은 경우 스레드를 종료한다. 
반대로 처리해야 되는 작업 보다 생성된 스레드 수가 더 적은 경우 필요한 만큼 스레드를 생성하게 된다. 
- `newScheduledThreadPool` : 스레드의 수가 고정돼 있고, 일정 시간 이후에 실행하거느 주기적으로 작업을 실행 할 수 있다. `Executor.Timer` 와 유사하다. 


![그림 1]({{site.baseurl}}/img/java/concept-asynchronous-programming-2.png)  


생성된 `Thread Pool` 에 앞서 살펴본 `Runnable` 구현체를 `execute()` 메소드로 전달해 주면, 
`Thread Pool` 내에서 실행 가능하다.  

```java
@Test
public void executor_runnable() throws Exception {
    // given
    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("ES-Runnable-");
    ExecutorService es = Executors.newSingleThreadExecutor(threadFactory);
    Runnable runnable = () -> {
        try {
            printAndAddLog("start runnable");
            Thread.sleep(100);
            printAndAddLog("end runnable");
        } catch(Exception e) {
            e.printStackTrace();
        }
    };

    // when
    printAndAddLog("start");
    es.execute(runnable);
    printAndAddLog("end");
    Thread.sleep(200);

    // then
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start runnable",
            "end runnable"
    ));
    es.shutdown();
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ES-Runnable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[ES-Runnable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
```  

### Callable, Future
`Executor` 와 함께 `Java 1.5` 에서 등장한 `Callable` 과 `Future` 는 `Executor` 에서 
사용할 수 있는 또 다른 하나의 비동기 작업을 구현할 수 있는 클래스이다.  

앞서 살펴본 `Runnable` 은 하나의 비동기 작업을 구현할 수 있지만 작업의 결과가 필요할 때, 
`DB` 와 같은 외부 저장소를 사용하지 않고 애플리케이션 내(`Memory`) 상에서는 처리의 어려움이 있다. 
하지만 `Callable` 은 `Runnable` 과 수행하는 역할은 동일하지만 비동기 작업에 대한 결과를 리턴할 수 있다는 부분에서 
차이점을 가지고 있다. 
또 다른 차이점이라면 `Runnable` 은 비동기 동작에서 예외를 던질 수 없고 따로 `try-catch` 문을 사용해야 하지만, 
`Callanble` 은 예외를 던져 처리할 수 있다.  

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```  

`Callable` 인터페이스를 보면 구현 메소드인 `call()` 에 리턴값이 존재하고, 예외 또한 던질 수 있는 것을 확인 할 수 있다.  

그렇다면 `Callable` 에서 리턴한 결과는 어떻게 받는지에 대해 고민해보면, `Future` 를 통해 결과를 받게 된다. 
`Future` 가 `Callable` 의 결과라 아니라 `Callable` 의 비동기적인 결과를 가져올 수 있는 핸들러로 이해하면 쉽다.  

```java
public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```  

`ExecutorService` 의 `submit()` 메소드에 비동기 작업인 `Callable` 의 전달하면 리턴값으로 `Future` 를 리턴해주게 된다. 
그리고 이후에 `Future.get()` 를 사용해서 `Callable` 의 결과를 받아로 수 있다.  

하지만 주의해야 할점은 `Future.get()` 메소드는 `Blocking` 으로 수행되는 메소드라는 점이다. 
비동기 작업은 병렬성을 가질 수 있지만, 작억의 결과를 받아오기 위해서는 결국 비동기 작업이 완료 될때까지 대기하는 `Blocking` 이 발생하게 된다.  


```java
@Test
public void executor_future_callable() throws Exception {
    // given
    ExecutorService es = Executors.newSingleThreadExecutor();
    threadFactory.setThreadNamePrefix("ES-Callable-");
    Callable<String> callable = () -> {
        printAndAddLog("start callable");
        Thread.sleep(100);
        String result = "callable";
        printAndAddLog("end callable");

        return result;
    };

    // when
    printAndAddLog("start");
    Future<String> future = es.submit(callable);
    printAndAddLog("end");
    String actual = future.get(); // blocking

    // then
    assertThat(actual, is("callable"));
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start callable",
            "end callable"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ES-Callable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[ES-Callable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
```  

### CountDownLatch, Semaphore, CyclicBarrier
`Java 1.5` 에 추가된 `CountDownLatch`, `Semaphore`, `CyclicBarrier` 는 비동기 작업을 위해 구성된 멀티 스레드 환경에서 
동기화에 대한 기능을 제공하는 클레스이다.  

#### CountDownLatch
걸쇠라는 `Latch` 단어가 포함된 것처럼 실행에 걸쇠를 걸어 잠깐 멈춰 놓는 기능이다. 
원하는 지점에서 `await()` 메소드를 호출해 코드 진행을 중단시키고, 
다른 스레드에서 `CountDownLatch` 에 설정된 횟수만큼 `countDown()` 메소드를 호출하면 그때 중단된 코드 흐름이 모두 진행된다.  

`CountDownLatch` 는 `CountDownLatch.await()` 가 호출된 스레드는 다른 스레드에서 `CountDownLath.countDown()` 메소드가 
정해진 횟수만큼 호출 될떄까지 대기하고 있다가, 호출이 모두 완료되면 그때 `await()` 의 그 다음 줄을 실행하게 된다. 
`CountDownLatch.countDown()` 을 호출하는 스레드는 대기 상태에 빠지지 않고 계속해서 비동기로 작업이 수행된다.   

```java
@Test
public void countDownLatch() throws Exception {
    // given
    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("ES-countdownLatch-");
    int count = 5;
    ExecutorService es = Executors.newFixedThreadPool(count, threadFactory);
    CountDownLatch countDownLatch = new CountDownLatch(count);
    for(int i = 0; i < count; i++) {
        int idx = i;
        es.submit(() -> {
            Thread.sleep(100);
            printAndAddLog("idx : " + idx);
            countDownLatch.countDown();

            return null;
        });
    }

    // when
    printAndAddLog("wait");
    countDownLatch.await();
    printAndAddLog("end");
    es.shutdown();

    // then
    assertThat(resultQueue, hasSize(7));
    assertThat(resultQueue, contains(
            is("wait"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx"),
            is("end")
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - wait
[ES-countdownLatch-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 0
[ES-countdownLatch-5] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 4
[ES-countdownLatch-3] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 2
[ES-countdownLatch-4] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 3
[ES-countdownLatch-2] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 1
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
```  

`CountDownLatch` 의 카운트 수, `ExecutorService` 스레드 풀 크기 모드 5이다. 
메인 스레드는 `countDown()` 이 5번 호출 될떄까지 `start` 로그를 호출하고 계속 대기하게 된다. 
`countDown()` 메소드는 스레드 풀에서 `100` 밀리초 슬립 하고 로그까지 출력 후 호출 되기 때문에 위 로그와 같이 가장 마지막에 `end` 로그가 찍히는 것을 확인 할 수 있다. 


#### CyclicBarrier
`CountDownLatch` 와 비슷한 기능을 수행하는 클래스이다. 
`CountDownLatch` 는 위 설명처럼 `await()` 메소드가 호출 된 스레드 만 대기상태에 빠지고, `countDown()` 메소드를 호출하는 스레드는 대기 상태에 빠지지 않는다.  

`CyclicBarrier` 는 이와 다르게 참여하는 다수의 스레드가 대기 상태에 빠지게 되고, 모든 스레드가 참여가 완료되면 대기 상태에서 함께 풀리고 비동기 작업이 수행된다. 
`CyclicBarrier` 는 스레드 풀에서 여러 비동기 작업을 수행할 때 스레드 풀의 작업이 초기화 작업 등으로 각기 다른 시간이 소요 될 수 있을 때, 
동시에 수행될 수 있도록 동기화를 맞출 수 있다.  

```java
@Test
public void cyclicBarrier() throws Exception {
    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("ES-cyclicBarrier-");
    int count = 5;
    ExecutorService es = Executors.newFixedThreadPool(count, threadFactory);
    CyclicBarrier cyclicBarrier = new CyclicBarrier(count + 1);
    for(int i = 0; i < count; i++) {
        int idx = i;
        es.submit(() -> {
            cyclicBarrier.await();
            printAndAddLog("idx : " + idx);

            return null;
        });
    }

    // when
    printAndAddLog("wait");
    cyclicBarrier.await();
    printAndAddLog("end");
    es.shutdown();

    // then
    assertThat(resultQueue, hasSize(7));
    assertThat(resultQueue, contains(
            is("wait"),
            is("end"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx"),
            startsWith("idx")
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - wait
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ES-cyclicBarrier-3] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 2
[ES-cyclicBarrier-2] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 1
[ES-cyclicBarrier-4] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 3
[ES-cyclicBarrier-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 0
[ES-cyclicBarrier-5] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - idx : 4
```

`ExecutorService` 스레드 풀 크기는 5이지만, `CyclicBarrier` 크기는 6으로 설정되었다. 
현재 테스트에서 구현하고자 하는 것은 스레드 풀과 메인 스레드가 동시에 `await()` 메소드 이후 부분을 수행하도록 하는 것이기 때문에, 
메인 스레드까지 `CyclicBarrier` 에 포함시키고자 크기가 6으로 설정 되었다. 
메인 스레드는 `start` 로그 까지 출력하고 대기하고, 스레드 풀은 모든 스레드가 `await()` 메소드를 호출 할떄 까지 대기하고 있다가 
메인 스레드 포함 총 6번 `await()` 가 호출되면 메인 스레드는 `end` 로그를 출력하고 스레드 풀의 비동기 작업들은 각 로그를 출력하게 된다.  

#### Semaphore
`Semaphore` 는 `synchronized` 키워드 함께 자주 설명된다. 
`synchronized` 는 동시성에 있었을 때 해당 블럭을 하나의 스레드만 접근해서 수행하고 다른 스레드의 접근은 막는 기능을 수행한다. 
이와 비교해서 `Semaphore` 는 동기화를 수행할 블럭에 접근할 수 있는 스레드의 수를 지정할 수 있는 기능을 제공한다.  


### FutureTask
`Java 1.5` 에 추가된 `FutureTask` 는 `RunnableTurue` 이고, 
`RunnableFuture` 는 `Runnable` 과 `Future` 의 결합체이다.  

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```  

`RunnableTask` 의 구현체인 `FutureTask` 는 `Runnable`, `Callable` 을 모두 사용가능한 
`Future` 객체라고 할 수 있다. 
먼저 `Callable` 을 사용한 `FutureTask` 테스트는 아래와 같다.  

```java
@Test
public void futureTask_callable() throws Exception {
    // given
    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("ES-futureTaskCallable-");
    ExecutorService es = Executors.newSingleThreadExecutor(threadFactory);
    FutureTask<String> futureTaskCallable = new FutureTask<String>(() -> {
        printAndAddLog("start callable");
        Thread.sleep(100);
        String result = "callable";
        printAndAddLog("end callable");

        return result;
    });

    // when
    printAndAddLog("start");
    es.execute(futureTaskCallable);
    printAndAddLog("end");
    String actual = futureTaskCallable.get(); // blocking

    // then
    assertThat(actual, is("callable"));
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start callable",
            "end callable"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ES-futureTaskCallable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start callable
[ES-futureTaskCallable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end callable
```  

`Runnable` 을 사용한 `FutureTask` 의 테스트는 아래와 같다. 

```java
@Test
public void futureTask_Runnable() throws Exception {
    // given
    CustomizableThreadFactory threadFactory = new CustomizableThreadFactory();
    threadFactory.setThreadNamePrefix("ES-futureTaskRunnable-");
    ExecutorService es = Executors.newSingleThreadExecutor(threadFactory);
    FutureTask<String> futureTaskRunnable = new FutureTask<String>(() -> {
        try {
            printAndAddLog("start runnable");
            Thread.sleep(100);
            printAndAddLog("end runnable");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }, "runnable");

    // when
    printAndAddLog("start");
    es.execute(futureTaskRunnable);
    printAndAddLog("end");
    String actual = futureTaskRunnable.get();   // blocking

    // then
    assertThat(actual, is("runnable"));
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start runnable",
            "end runnable"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ES-futureTaskRunnable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[ES-futureTaskRunnable-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
```  

앞서 설명한 것과 같이 `Runnable` 은 비동기 작업에 대한 결과를 받을 수 없다. 
하지만 `FutureTask` 를 사용하면 비동기 작업에 따른 결과를 얻을 순 없지만, `FutureTask` 생성자에 지정된 결과를 설정해두고 리턴 받을 수 있다.  


### CompletableFuture
`Java 1.8` 에 등작한 `CompletableFuture` 는 `Future` 와 `CompletionStage` 를 구현한 클래스이다.  
먼저 `CompletionStage` 는 비동기 작업의 완료 수행에 따른 다양한 동작을 제공하는 인터페이스이다. 
하나의 비동기 동작이 완전히 완료 될 수도 있고, 예외가 발생하면서 실패가 될떄 필요한 예외처리를 추가하는 등의 동작이 가능하다. 
또한 여러 비동기 작업을 파이프라인처럼 연결해 비동기 작업 처리 절차를 구성 할 수도 있다.  

간단하게 `Future` 와 비교를 했을 때, `Future` 는 비동기 작업에 대한 검사를 `isDone()`, `isCancled()` 를 통해서만 가능하다. 
하지만 복잡하고 예외사항이 더 많은 비동기 작업을 컨트롤하기 위해서는 이러한 부분으로는 부족하다. 
이런 부족한 부분은 더욱 다양한 인터페이스를 통해 기능을 제공해 주어서 우아한 처리가 가능하게 하는 것이 `CompletionStage` 이고 이를 구현한 구현체가 `CompletableFuture` 이다.  

가장 간단한 예로 `CompletableFuture` 를 사용하면, 
`Thread` 와 `Runnable` 의 비동기 작업에 따른 결과를 `Future` 와 같이 결과를 받아 올 수 있다.  

```java
@Test
public void completableFuture_runnable() throws Exception {
    // given
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
    Runnable runnable = () -> {
        String result = "runnable";

        try {
            printAndAddLog("start runnable");
            Thread.sleep(100);
            printAndAddLog("end runnable");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        completableFuture.complete(result);
    };

    // when
    printAndAddLog("start");
    new Thread(runnable).start();
    printAndAddLog("end");
    String actual = completableFuture.get();    // blocking

    // when
    assertThat(actual, is("runnable"));
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "start runnable",
            "end runnable"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[Thread-0] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start runnable
[Thread-0] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end runnable
```  

다음으로 `CompletableFuture` 에서 제공하는 메소드를 사용하면 좀더 다양한 동작을 비동기의 작업의 결과로 만들어 내면서 컨트롤 할 수 있다. 
기본적으로 `CompletableFuture` 는 `supplyAsync()` 를 통해 비동기 파이프라인에서 처리할 데이터를 제공할 수 있다. 
그리고 `Runnable` 과 같은 비동기 작업이 정의된 클래스는 `runAsync()` 를 사용해서 실행 할 수 있다. 
참고로 `CompletableFuture` 에 등록된 비동기 동작들은 `ForkJoinPool` 에서 실행 된다.  

```java
@Test
public void completableFuture_async() throws Exception {
    // when
    printAndAddLog("start");
    CompletableFuture<Void> completableFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "completableFuture";
            })
            .thenApply((str) -> {
                printAndAddLog(str);
                return "Prefix-" + str;
            })
            .thenApply((str) -> {
                printAndAddLog(str);
                return str;
            })
            .thenApply((str) -> str + "-Suffix")
            .thenAccept((str) -> printAndAddLog(str));
    printAndAddLog("end");

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "completableFuture",
            "Prefix-completableFuture",
            "Prefix-completableFuture-Suffix"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ForkJoinPool.commonPool-worker-19] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - completableFuture
[ForkJoinPool.commonPool-worker-19] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - Prefix-completableFuture
[ForkJoinPool.commonPool-worker-19] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - Prefix-completableFuture-Suffix
```  

`CompletableFuture` 에 등록된 작업들이 모두 메인 스레드와는 별대로 비동기적으로 수행된 것을 확인 할 수 있다. 
그리고 각 비동기 동작에 대한 결과를 바탕으로 다음 비동기 동작이 순차적으로 수행하는 파이프라인 동작 또한 정상적으로 동작 된 것을 확인 할 수 있다.  

만약 비동기 동작에 대한 결과 처리를 별도의 스레드에서 수행하고 싶을 경우 아래와 같이 `thenApplyAsync()` ,`thenAcceptAsync()` 메소드를 사용해서 가능하다.    

```java
@Test
public void completableFuture_async_threadpool() throws Exception {
    // given
    CustomizableThreadFactory threadFactory1 = new CustomizableThreadFactory();
    threadFactory1.setThreadNamePrefix("myPool-1-");
    ExecutorService myPool1 = Executors.newSingleThreadExecutor(threadFactory1);
    CustomizableThreadFactory threadFactory2 = new CustomizableThreadFactory();
    threadFactory2.setThreadNamePrefix("myPool-2-");
    ExecutorService myPool2 = Executors.newSingleThreadExecutor(threadFactory2);

    // when
    printAndAddLog("start");
    CompletableFuture<Void> completableFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(100);
                    printAndAddLog("async");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "completableFuture";
            })
            .thenApplyAsync((str) -> {
                printAndAddLog(str);
                return "Prefix-" + str;
            }, myPool1)
            .thenApplyAsync((str) -> {
                printAndAddLog(str);
                return str;
            }, myPool2)
            .thenApply((str) -> str + "-Suffix")
            .thenAccept((str) -> printAndAddLog(str));
    printAndAddLog("end");

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(6));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "async",
            "completableFuture",
            "Prefix-completableFuture",
            "Prefix-completableFuture-Suffix"
    ));
}
```  

```
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - start
[main] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - end
[ForkJoinPool.commonPool-worker-19] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - async
[myPool-1-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - completableFuture
[myPool-2-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - Prefix-completableFuture
[myPool-2-1] INFO com.windowforsun.reactivestreams.javasync.JavaAsyncTest - Prefix-completableFuture-Suffix
```  

`CompletableFuture` 비동기 파이프라인 작업에서 `supplyAsync()` 메소드는 기본 스레드 풀인 `ForkJoinPool` 에서 수행된다. 
그리고 다음 비동기 동작인 `thenApplyAsync()` 는 설정한 `myPool-1` 에서 수행되고, 
이후 비동기 동작들은 모두 `myPool-2` 에서 실행되는 것을 확인 할 수 있다.  

만약 `CompletableFuture` 를 사용해서 구성된 비동기 파이프라인에서 특정 동작의 리턴값이 `CompletableFuture` 이면, 
`CompletableFuture<CompletableFuture<String>` 과 같은 형태가 된다. 
이런 경우 `thenCompose()` 와 같은 메소드를 사용해서 처리 할 수 있다.  

```java
@Test
public void completableFuture_thenCompose() throws Exception {
    // when
    printAndAddLog("start");
    CompletableFuture<Void> completableFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "completableFuture";
            })
            .thenCompose((str) -> {
                printAndAddLog(str);
                return CompletableFuture.completedFuture("Prefix-" + str);  // 리턴 값이 CompletableFuture
            })
            .thenApply((str) -> {
                printAndAddLog(str);
                return str;
            })
            .thenApply((str) -> str + "-Suffix")
            .thenAccept((str) -> printAndAddLog(str));
    printAndAddLog("end");

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(5));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "completableFuture",
            "Prefix-completableFuture",
            "Prefix-completableFuture-Suffix"
    ));
}
```  

결과 값에 `Prefix` 를 붙이는 부분의 리턴값이 `CompletableFuture` 이다. 
만약 `thenCompose()` 가 아닌 `thenApply()` 을 사용한다면 이후 파이프라인 부터 사용하는 결과 값 또한 `CompletableFuture` 타입을 사용하게 된다.  

다음으로 `CompletableFuture` 비동기 파이프라인 실행 중 예외가 발생한다면 `exceptionally()` 메소드에 예외 처리에 대한 내용을 정의해 주면된다.  

```java
@Test
public void completableFuture_exceptionally() throws Exception {
    // when
    printAndAddLog("start");
    CompletableFuture<Void> completableFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "completableFuture";
            })
            .thenApply((str) -> {
                printAndAddLog(str);
                return "Prefix-" + str;
            })
            .thenApply((str) -> {
                if(1 == 1) {
                    throw new RuntimeException(str + "-Exception");
                }
                printAndAddLog(str);
                return str;
            })
            .thenApply((str) -> str + "-Suffix")
            .exceptionally((e) -> e.getMessage())
            .thenAccept((str) -> printAndAddLog(str));
    printAndAddLog("end");

    // then
    Thread.sleep(200);
    assertThat(resultQueue, hasSize(4));
    assertThat(resultQueue, contains(
            "start",
            "end",
            "completableFuture",
            "java.lang.RuntimeException: Prefix-completableFuture-Exception"
    ));
}
```  

파이프라인 결과값에 `Prefix-` 까지 붙이고 나서 로그 출력 전에 예외가 발생했다. 
그러므로 이후 파이프라인 스텝은 수행되지 않고, 
`exceptionally()` 에 정의된 예외처리가 수행되고 결과로 리턴 되는 것을 확인 할 수 있다.  



---
## Reference
[Asynchronous Programming in Java](https://rjlfinn.medium.com/asynchronous-programming-in-java-d6410d53df4d)  
[Asynchronous Programming in Java](https://www.baeldung.com/java-asynchronous-programming)  
[Package java.util.concurrent](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html)  

