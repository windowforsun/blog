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
toc: true 
use_math: true
---  

## Reactor Backpressure
`Reactive Stream` 에서 `Backpressure` 란 `Publisher` 와 `Subscriber` 사이에서 서로 과부하기 걸리지 않도록, 
제어하는 것을 의미한다. 
이는 `Reactive Stream` 에서는 매오 중요한 특성으로, 
`Publisher` 의 데이터를 받아 처리하는 `Subscriber` 간의 적절한 속도를 맞추는 것으로 
시스템 간의 `Overloading` 이 발생하는 것을 방지한다.  

`Backpressure` 는 `Reactive Stream` 에서만 존재하는 개념은 아니다. 
어떻게 보면 유체역학에서 유량제어에 관련된 [Backpressure](https://en.wikipedia.org/wiki/Back_pressure)
를 소프트웨어 데이터 흐름제어 관점으로 차용한 것이라고 할 수 있다.  

말그대로 `Backpressure` 는 `Publisher` 와 `Subscriber` 사이에서 오고가는 데이터의 양을 컨트롤 하는 것인데, 
아래와 같은 일반적인 특성을 기억해야 한다. 

- `Publisher` 에서 방출한 데이터는 `Common Buffer` 에 추가된다. 
- `Subscriber` 는 `Common Buffer` 에서 데이터를 읽어온다. 
- `Common Buffer` 의 크기는 제한돼 있다. 

`Reactor Flux` 는 기보적으로 `256` 크기의 `Small Buffer` 를 가진다.   

> `Backpressure` 의 동작은 `Multi-thread` 환경에서 비로서 의미가 있음을 기억해야 한다. 


### What is backpressure
`Backpressure` 에 대해 알아 볼떄, 
생산자인 `Publisher` 를 2가지 그룹으로 나누면 `Subscriber` 의 요구를 잘 들어주는 `Publisher` 와 
그렇지 않고 무시하는 `Publisher` 로 나눌 수 있다.  

일반적으로 `Hot Publisher` 는 `Subscriber` 의 요구를 무시하는 특성을 지닌다. 
`TV` 나 라디오 방송을 예를들면 라이브로 스트리밍되는 방송 데이터는 
이를 시청하는 `Subscriber` 의 요구와는 관계없이 계속해서 전송되고 
만약 `Subscriber` 가 느리다면 쉽게 압도될 수 있다.  

반대로 `Cold Publisher` 는 `Subscriber` 의 요구를 잘 들어줘서, 
구독이 수행될 때 요청받은 데이터만큼 전달하는 특성이 있다. 
이는 마치 `HTTP` 요청을 주고 받는 것과 비슷한데, 
`HTTP` 는 상대방이 요청을 해야 응답을 주는 것을 생각 할 수 있다.  

> 위 `Cold/Hot Publisher` 에 대한 예시는 절대적인 개념이 아님을 유의해아 한다. 
모든 `Hot Publisher` 가 `Subscriber` 의 요구를 무시하는 것이 아니고, 
모든 `Cold Publisher` 가 `Subscriber` 의 요구를 잘 들어주는 것이 아니다. 
앞선 설명은 일반적인 두 그룹의 특성에 대한 예시로 설명한 것이다.  

일반적인 `Reactive Stream` 을 보면 아래와 같다.  

```java
@Test
public void commonStream() {
    Flux.range(1, 300)
            .doOnRequest(value -> log.info("request : {}", value))
            .doOnNext(integer -> log.info("next : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(300)
            .verifyComplete();
}
```

위 코드를 실행하면 따로 설정이 되지 않으면 기본적으로 구독과 함께 `request` 를 통해 `Long.MAX_VALUE` 
의 개수를 요청해서 무한 스트림에 대해서 최대한 많은 데이터를 한번에 받아 온다.  

```
[main] INFO com.windowforsun.bachpressure.BackpressureTest - request : 9223372036854775807
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 1
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 2
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 3

...

[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 298
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 299
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 300
```  

이번에는 지연이 발생하는 `Subscriber` 를 시뮬레이션 하면서, 
`request` 를 1개씩 받아오는 예제는 아래와 같다.  

```java
@Test
public void respect_subscriber_request() {
    Flux.range(1, 300)
            .doOnRequest(value -> log.info("request : {}", value))
            .doOnNext(integer -> log.info("next : {}", integer))
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(10)), 1)
            .as(StepVerifier::create)
            .expectNextCount(300)
            .verifyComplete();
}
```  

코드 결과를 보면 알 수 있듯이, `range()` 는 `Subscriber` 의 요구를 잘 들어주는 `Publisher` 이다. 
`concatMap()` 를 통해 `10ms` 딜레이와 함께 `prefetch` 인수로 `request` 1을 수행해 데이터를 1개씩 받아도록 했다. 
로그를 보면 앞선 예제와 달리 스트림에서 최대한 많은 데이터를 한번에 가져오는 것이 아니라, 
순차적으로 1개씩 요청해 데이터를 가져오는 것을 확인 할 수 있다. 

```
[main] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 1
[main] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[main] INFO com.windowforsun.bachpressure.BackpressureTest - next : 2
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 3

...

[parallel-5] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[parallel-5] INFO com.windowforsun.bachpressure.BackpressureTest - next : 297
[parallel-6] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[parallel-6] INFO com.windowforsun.bachpressure.BackpressureTest - next : 298
[parallel-7] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[parallel-7] INFO com.windowforsun.bachpressure.BackpressureTest - next : 299
[parallel-8] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
[parallel-8] INFO com.windowforsun.bachpressure.BackpressureTest - next : 300
[parallel-9] INFO com.windowforsun.bachpressure.BackpressureTest - request : 1
```  

`Publisher` 의 아이템 방출 속도 보다 `Subscriber` 의 처리시간이 오래 걸릴떄 `Backpressure` 가 필요한 이유는 
`Reactive Stream` 내부에는 `Buffer` 가 있기 때문이다. 
이 버퍼는 무한한 크기를 갖는 것이 아니라 고정된 크기를 가지기 때문에, 
무작정 최대 데이터 스트림을 받아와 버퍼에 넣도록 하면 스트림 수행 도중 예외가 발생하게 된다.

```java
@Test
public void backpressure_buffer_overflow() {
    Flux.interval(Duration.ofMillis(1))
            .doOnRequest(value -> log.info("request : {}", value))
            .doOnNext(integer -> log.info("next : {}", integer))
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(10)))
            .doOnRequest(value -> log.info("after request : {}", value))
            .doOnNext(integer -> log.info("after next : {}", integer))
            .as(StepVerifier::create)
            .expectErrorSatisfies(throwable -> {
                Assertions.assertTrue(throwable instanceof IllegalStateException);
                Assertions.assertTrue(throwable.getMessage().contains("Could not emit tick 32 due to lack of requests"));
            })
            .verify();
}
```  

`concatMap()` 내부 코드를 보면 기본으로 `request` 값은 `concatMap()` 의 내부 큐 초기 용량인 `32` 이다. 
하지만 `interval()` `Publisher` 는 지정된 시간 주기마다 데이터를 방출하기 때문에, 
`Subscriber` 의 요청이 무시되는 `Publisher` 이다. 
이런 상황에서 `concatMap()` 의 `Subscriber` 는 `100ms` 마다 데이터를 처리하게 되면, 
내부 버퍼큐는 금방 용량이 차버려 아래와 같이 `OverflowException` 이 발생하게 된다.  


```
[main] INFO com.windowforsun.bachpressure.BackpressureTest - after request : 9223372036854775807
[main] INFO com.windowforsun.bachpressure.BackpressureTest - request : 32
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 0
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 1

...

[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 30
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 31

reactor.core.Exceptions$OverflowException: Could not emit tick 32 due to lack of requests (interval doesn't support small downstream requests that replenish slower than the ticks)

	at reactor.core.Exceptions.failWithOverflow(Exceptions.java:233)
	at reactor.core.publisher.FluxInterval$IntervalRunnable.run(FluxInterval.java:131)
	at reactor.core.scheduler.PeriodicWorkerTask.call(PeriodicWorkerTask.java:59)
	at reactor.core.scheduler.PeriodicWorkerTask.run(PeriodicWorkerTask.java:73)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.runAndReset(FutureTask.java:305)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:305)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
	Suppressed: java.lang.Exception: #block terminated with an error
		at reactor.core.publisher.BlockingSingleSubscriber.blockingGet(BlockingSingleSubscriber.java:99)
		at reactor.core.publisher.Flux.blockLast(Flux.java:2510)
		at com.windowforsun.bachpressure.BackpressureTest.backpressure_buffer_overflow(BackpressureTest.java:127)
```  

`interval()` 에서 32개의 데이터를 요청하면 `1ms` 마다 데이터를 방출해준다. 
방출된 데이터는 먼저 버퍼로 들어가게 되고 `Subscriber` 는 버퍼에서 `100ms` 마다 데이터를 가져와 처리한다. 
`0 ~ 31` 까지 데이터가 모두 크기가 32인 버퍼에 들어가 꽉차 있는 상태에서 
`interval()` 는 `Subscriber` 의 처리 속도와는 무관하게 `1ms` 마다 추가적으로 아이템을 방출하려 하기 때문에 
`OverflowException` 이 발생하게 되는 것이다.  


### Prefetch
https://beer1.tistory.com/18
