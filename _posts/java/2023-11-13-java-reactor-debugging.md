--- 
layout: single
classes: wide
title: "[Java 실습] "
header:
  overlay_image: /img/java-bg.jpg 
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Reactor Extra
  - BooleanUtils
toc: true 
use_math: true
---  

## Debugging Reactor
[Debugging Reactor](https://projectreactor.io/docs/core/release/reference/#debugging) 
`Reactor` 공식문서 중 `Debugging` 관련 챕터를 보면 `Reactor` 를 `Debugging` 하는 방법에 대해 설명이 있다. 
이번 포스트에서는 이를 기반으로 `Reactor` 를 `Debugging` 하는 다양한 방법에 대해 알아본다.  

비동기 프로그래밍을 접할 떄 가장 어려운 부분이 `Debugging` 이지만, `Reactor` 에서는 
이를 간단하게 수행 할 수 있는 몇가지 도구를 제공한다.  

이후 디버깅예제에서 사용할 예제인 `TestSource` 를 먼저 살펴본다.  

```java
@Slf4j
public class TestSource {
    private static int square(int i) {
        log.info("execute getResult : {}", i);

        if(i % 2 == 0) {
            throw new IllegalArgumentException("test exception");
        }

        return i * i;
    }

    public static Mono<Integer> produceSquare(int i) {
        return Mono.just(i)
                .flatMap(s -> Mono.just(s)
                        .flatMap(s2 -> Mono.just(square(s2))))
                .delayElement(Duration.ofMillis(10))
                .subscribeOn(Schedulers.parallel());
    }
}
```  

`square(int)` 메소드는 이름 그대로 제곱의 결과를 리턴하는데, 인수로 받은 값이 짝수인 경우에는 예외를 던지게 된다. 
그리고 `square(int)` 를 사용해서 `Reactive Stream` 을 생성하는 `produceSquare(int)` 의 스트림 구성은 
고의적으로 복잡한 구성을 해 놓았다. 
이후 예제에서 이런 `Reactive Stream` 구성에서 어떤식으로 디버깅이 가능한지 알아본다.  

### 일반적인 Stacktrace
가장 먼저 아무런 설정과 옵션을 추가하지 않는 `Reactor` 의 기본 `StackTrace` 를 살펴본다.  

```java
public class PlainStackTraceTest {
    @Test
    public void reactor_plain_stack_trace() {
        TestSource.produceSquare(2)
                .as(StepVerifier::create)
                .expectErrorSatisfies(Throwable::printStackTrace)
                .verify();
    }
}
```  

위 코드를 수행하면 아래와 같은 `StackTrace` 가 출력된다.  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:14)
	at com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$0(TestSource.java:23)
	at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:152)
	at reactor.core.publisher.MonoFlatMap.subscribeOrReturn(MonoFlatMap.java:53)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4480)
	at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:200)
	at reactor.core.publisher.MonoFlatMap.subscribeOrReturn(MonoFlatMap.java:53)
	at reactor.core.publisher.Mono.subscribe(Mono.java:4480)
	at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.run(MonoSubscribeOn.java:126)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
```  

`produceSquare` 메소드에서 장황하게 스트림을 구성했지만 `StackTrace` 에 출력된 애플리케이션 관련 내용은 아래의 단 2줄만 출력되는 것을 확인 할 수 있다.  

```
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:14)
	at com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$0(TestSource.java:23)
```  

`Reactor` 를 통해 구성한 `Reactive Stream` 관련 `StackTrace` 는 모두 제외 됐고, 
일반적인 메소드 단위 `StackTrace` 만 출력된 결과이다. 
그러므로 `Reactive Stream` 의 어떤 경로에서 에러가 발생했는지 찾기에는 내용이 부족하다. 


### Hooks.onOperatorDebug()
`Reactor API` 문서를 보면 `Hooks` 라는 클래스가 있는데, 내부에 선언된 전역 메소드 중 
[onOperatorDebug()](https://projectreactor.io/docs/core/3.5.4/api/reactor/core/publisher/Hooks.html#onOperatorDebug--)
를 사용하면 간단하게 디버그 모드를 활성화 할 수 있다.  

이는 런타임 `StackTrace` 로 부터 조립 정보를 가져오는 것으로, 
`Reactor` 의 `Operator` 가 생성 될 때마다 스택을 캡쳐해서 유지하는 방식이다. 
그리고 특정 `Operator` 에서 에러가 발생하면 `onError` 로 전달되는 `Throwable` 이 `StackTrace` 에
[Suppressed Exception](https://www.baeldung.com/java-suppressed-exceptions) 와 같이 추가된다. 
이러한 동작을 통해 일반적인 `StackTrace` 와 비교했을 때, 
`Reactive Stream` 측면에서 좀 더 상세한 정보와 경로를 확인 할 수 있다.  

```java
public class HookOperatorDebugStackTraceTest {
    @Test
    public void reactor_hook_operator_debug_stack_trace() {
        Hooks.onOperatorDebug();

        TestSource.produceSquare(2)
                .as(StepVerifier::create)
                .expectErrorSatisfies(Throwable::printStackTrace)
                .verify();
    }
}
```  

비교를 위해 `onOperatorDebug()` 를 적용한 위 코드를 실행하면 아래와 같은 `StackTrace` 를 확인 할 수 있다.  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoFlatMap] :
	reactor.core.publisher.Mono.flatMap(Mono.java:3100)
	com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$1(TestSource.java:24)
Error has been observed at the following site(s):
	*_______Mono.flatMap ⇢ at com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$1(TestSource.java:24)
	*_______Mono.flatMap ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:23)
	|_ Mono.delayElement ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:25)
	|_  Mono.subscribeOn ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:26)
Original Stack Trace:
		at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
		at com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$0(TestSource.java:24)
		at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:152)
		at reactor.core.publisher.MonoFlatMap.subscribeOrReturn(MonoFlatMap.java:53)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4480)
		at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:200)
		at reactor.core.publisher.MonoFlatMap.subscribeOrReturn(MonoFlatMap.java:53)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4480)
		at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.run(MonoSubscribeOn.java:126)
		at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
		at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
		at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
		at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
		at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
		at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
		at java.base/java.lang.Thread.run(Thread.java:829)
```  

앞선 `일반적인 StackTrace` 와 비교 했을 떄, 상단 `StackTrace` 에는 아래와 같은 내용이 추가 된 것을 확인 할 수 있다.  

```
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoFlatMap] :
	reactor.core.publisher.Mono.flatMap(Mono.java:3100)
	com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$1(TestSource.java:24)
Error has been observed at the following site(s):
	*_______Mono.flatMap ⇢ at com.windowforsun.reactor.debug.TestSource.lambda$produceSquare$1(TestSource.java:24)
	*_______Mono.flatMap ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:23)
	|_ Mono.delayElement ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:25)
	|_  Mono.subscribeOn ⇢ at com.windowforsun.reactor.debug.TestSource.produceSquare(TestSource.java:26)
```  

예외가 발생한 위치 그리고 메소드의 콜스택과 비슷하게, 
예외가 발생한 위치로 부터 `Reactive Stream` 의 경로를 확인해 볼 수 있다.  

하지만 `Hooks.onOperatorDebug()` 는 애플리케이션에서 사용되는 모든 `Operator` 에 대해 
`Operation Hook` 이 걸리면서 연산자맏 `StackTrace` 캡쳐에 대한 오버해드가 발생한다. 
이는 코드에 에러가 없는 경우에도 발생하기 때문에, 비용이 비싼 동작이므로 실제 환경에서 사용하기에는 무리가 있으므로 
개발이나 로컬 환경처럼 성능에 민감하지 않는 곳에서 사용해야 한다.  

