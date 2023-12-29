--- 
layout: single
classes: wide
title: "[Java 실습] Reactive Stream, Reactor Debugging"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 Debugging 을 하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Debugging
  - Reactor Extra
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


### ReactorDebugAgent.init()
먼저 알아본 `Hooks.onOperatorDebug()` 그 비싼 비용으로 인해 환경에 제약없이 사용할 수 있는 도움을 받을 수 있는 디버깅 방식이 없는 것은 아니다. 
`Spring` 과 `Reactor` 문서에서 권장하는 방식인 `reactor-tools` 의 기능 중 하나인 `ReactorDebugAgent` 를 사용하면, 
런타임에 추가적인 비용 없이 `StackTrace` 를 좀 더 자세히 확인해 볼 수 있다. 

사용을 위해서는 아래와 같이 의존성 추가가 필요하다. 
``

```groovy
dependencies {
  implementation "io.projectreactor:reactor-tools:{version}"
}
```  

> `Spring WebFlux` 를 사용한다면 별도의 의존성 추가가 필요 없고, 
> `2.2.0` 이후 버전아리면 이후 예제에서 나오는 `init` 메소드를 따로 실행해 주지 않아도 기본으로 호출된다. 
> 만약 비활성화가 필요하다면 `spring.reactor.debug-agent-enable` 값을 `false` 로 설정하면 된다. 

아래 코드의 실행 결과로 어떤식로 에러를 출력해주는지 확인해 본다.  

```java
public class ReactorDebugAgentTest {
    @BeforeAll
    public static void init() {
        ReactorDebugAgent.init();
        ReactorDebugAgent.processExistingClasses();
    }

    @Test
    public void reactor_debug_agent_stack_trace() {
        Mono.just(2)
                .flatMap(s -> Mono.just(s)
                        .flatMap(TestSource::produceSquare))
                .subscribeOn(Schedulers.parallel())
                .as(StepVerifier::create)
                .expectErrorSatisfies(Throwable::printStackTrace)
                .verify();
    }
}
```  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoFlatMap] :
	reactor.core.publisher.Mono.flatMap
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

`Hooks.onOperatorDebug()` 와 동일한 `StackTrace` 를 출력해주었다. 
`Spring Webflux` 에서 기본 설정이 활성화시키기 때문에 리얼 환경에서도 큰 부담없이, 
`Reactive Stream` 처리 과정에서 에러가 발생했을 때 디버깅 정보로 활용 할 수 었을 것같다.  


### checkpoint()
[checkpoint()](https://projectreactor.io/docs/core/3.5.4/api/reactor/core/publisher/Flux.html#checkpoint--) 
는 특정 `Operator` 체인 내의 `StackTrace` 만 캡쳐하는 방식이다. 
`checkpoint()` 사용 할때는 에러가 발생하는 `downstream` 에 `checkpoint()` 가 위치해야 한다. 
아래 코드에서 에러 발생지점은 `TestSource.produceSquare` 이다.  

```java
@Test
public void no_checkpoint() {
    Mono.just(2)
            .checkpoint()
            .flatMap(TestSource::produceSquare)
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

위 코드를 실행하면 아래의 에러 메시지와 같이 아무런 `StackTrace` 가 되지 않은 결과가 나온다.  

```
java.lang.IllegalArgumentException: test exception
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

정상적인 에러 발생 위치를 `StackTrace` 에 캡쳐하고 싶다면 에러 발생이 예상되는 지점을 기준으로 `downstream` 에 `checkpoint()` 를 위치시켜야 한다.  

```java
@Test
public void checkpoint() {
    Mono.just(2)
            .checkpoint()
            .flatMap(TestSource::produceSquare)
            .checkpoint()
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoFlatMap] :
	reactor.core.publisher.Mono.checkpoint(Mono.java:2190)
	com.windowforsun.reactor.debug.CheckpointDebugTest.checkpoint(CheckpointDebugTest.java:13)
Error has been observed at the following site(s):
	*__checkpoint() ⇢ at com.windowforsun.reactor.debug.CheckpointDebugTest.checkpoint(CheckpointDebugTest.java:13)
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

`checkpoint()` 로 부터 `upstream` 에서 에러가 발생하자 해당 `checkpoint` 지점이 `StackTrace` 에 찍힌 것을 확인 할 수 있다. 
이제 해당 지점으로 부터 `upstream` 에 대해 다시 에러 발생 위치 검증을 수행해 주면 된다.    

디버깅을 위해 `checkpoint()` 를 여기저기 위치시키다 보면 출력된 `StackTrace` 를 파악하는 것도 쉽지 않을 것이다. 
이런 경우를 위해 `checkpoint(String)` 을 사용해서 식별자를 추가할 수 있다.  

```java
@Test
public void checkpoint_description() {
    Mono.just(2)
            .checkpoint()
            .flatMap(TestSource::produceSquare)
            .checkpoint("here")
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Error has been observed at the following site(s):
	*__checkpoint ⇢ here
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

출력결과를 보면 입력한 식별자가 정상적으로 출력된 것을 확인 할 수 있다.  

하지만 위와 같이 식별자를 추가해버리면 `StackTrace` 의 내용을 자세히 알수 없다. 
어느 클래스 어느 위치에서 발생했는지에 대한 정보가 모두 `description` 으로 대체됐기 때문이다. 
이럴 때는 `true/false` 인수 하나를 더 념겨 줘서 `description` 과 자세한 `StackTrace` 정보를 함께 받아 볼 수 있다.  

```java
@Test
public void checkpoint_description_stacktrace() {
    Mono.just(2)
            .checkpoint()
            .flatMap(TestSource::produceSquare)
            .checkpoint("here", true)
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

```
java.lang.IllegalArgumentException: test exception
	at com.windowforsun.reactor.debug.TestSource.square(TestSource.java:15)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoFlatMap], described as [here] :
	reactor.core.publisher.Mono.checkpoint(Mono.java:2255)
	com.windowforsun.reactor.debug.CheckpointDebugTest.checkpoint_description_stacktrace(CheckpointDebugTest.java:46)
Error has been observed at the following site(s):
	*__checkpoint(here) ⇢ at com.windowforsun.reactor.debug.CheckpointDebugTest.checkpoint_description_stacktrace(CheckpointDebugTest.java:46)
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

에러 출력을 확인하면 `description` 과 `StackTrace` 정보가 모두 정상적으로 출력된 것을 확인 할 수 있다.  



### log()
`Reactive Stream` 을 디버깅 하는 가장 전통적인 방법은 `Reactor` 의 `Log Operator` 를 사용하는 것이다. 
`log()` 는 `Reactive stream` 에서 발생하는 모든 `Signal` 의 전파되는 과정과 흐름을 확인 할 수 있는 `Operator` 이다.  

```java
@Test
public void log_on_upstream() {
    Mono.just(2)
            .log()
            .flatMap(TestSource::produceSquare)
            .log()
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

위 소스는 `flatMap()` 에서 에러가 발생하는데, `log()` 는 `flatMap()` 을 기준으로 `upstream 에 위치해 있다.

```
16:08:14.916 [main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
16:08:14.917 [main] INFO reactor.Mono.Just.1 - | request(unbounded)
16:08:14.917 [main] INFO reactor.Mono.Just.1 - | onNext(2)
16:08:14.922 [main] INFO reactor.Mono.Just.1 - | onComplete()
```  

출력된 로그를 보면 `just()` 를 구독하고 `request()` 를 사용해서 데이터를 `just()` 에세 요청하면 
`onNext()` 를 통해 `just()` 로 부터 방출된 데이터가 2인 것을 확인 할 수 있다.
그리고 `log()` 는 `onComplete()` 과 함께 역할이 종료된다.  

하지만 그 이후에 에러가 발생했지만 `log()` 에는 해당 내용이 담겨져 있지 않다. 
`log()` 는 자신을 기준으로 `upstream` 에 대한 시그널을 로그로 출력하기 때문에, 
위 소스는 `log()` 기준으로 `downstream` 에서 에러가 발생했기 때문에 에러 관련 로그는 아무것도 출력되지 않은 것이다.  

```java
@Test
public void log() {
    Mono.just(2)
            .log()
            .flatMap(TestSource::produceSquare)
            .log()
            .as(StepVerifier::create)
            .expectErrorSatisfies(Throwable::printStackTrace)
            .verify();
}
```  

위 소스는 에러가 발생하는 위치인 `flatMap()` 의 `downstream` 에도 `log()` 를 추가했다. 

```
16:12:25.317 [main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
16:12:25.318 [main] INFO reactor.Mono.FlatMap.2 - | onSubscribe([Fuseable] MonoFlatMap.FlatMapMain)
16:12:25.319 [main] INFO reactor.Mono.FlatMap.2 - | request(unbounded)
16:12:25.319 [main] INFO reactor.Mono.Just.1 - | request(unbounded)
16:12:25.319 [main] INFO reactor.Mono.Just.1 - | onNext(2)
16:12:25.324 [main] INFO reactor.Mono.Just.1 - | onComplete()
16:12:25.325 [parallel-1] INFO com.windowforsun.reactor.debug.TestSource - execute getResult : 2
16:12:25.327 [parallel-1] ERROR reactor.Mono.FlatMap.2 - | onError(java.lang.IllegalArgumentException: test exception)
16:12:25.329 [parallel-1] ERROR reactor.Mono.FlatMap.2 - 
java.lang.IllegalArgumentException: test exception
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

출력된 로그를 보면 `just()` 는 `onComplete()` 를 통해 모든 시그널이 정상적으로 완료됐지만, 
`flatMap()` 에서 `onErorr()` 가 발생하며 에러를 방출한 것을 확인 할 수 있다. 
이후 더 자세한 원인 분석이 필요하다면 `flatMap()` 내부에서 방출하는 시그널에 대해서 다시 `log()` 를 통해 더 자세한 에러 발생 위치를 파악할 수 있을 것이다.  



---
## Reference
[Debugging Reactor](https://projectreactor.io/docs/core/release/reference/#debugging)  

