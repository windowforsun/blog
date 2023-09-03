--- 
layout: single
classes: wide
title: "[Java 실습] Reactive Stream, Reactor Backpressure"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactive Stream 흐름상에서 데이터의 유량을 제어 할 수 있는 Backpressure 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Backpressure
  - Flux
toc: true 
use_math: true
---  

## Reactor Backpressure
`Reactive Stream` 에서 `Backpressure` 란 `Publisher` 와 `Subscriber` 사이에서 서로 과부하기 걸리지 않도록, 
제어하는 것을 의미한다. 
이는 `Reactive Stream` 에서는 매우 중요한 특성으로, 
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

`Reactor Flux` 는 일반적으로 `256` 크기의 `Small Buffer` 를 가진다.   

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

### Reactor backpressure strategy
앞선 예제에서 처럼 `Publisher` 와 `Subscriber` 간 데이터를 전달 할때, 
`Buffer` 가 꽉차버린 상황을 `Overflow` 라고 한다. 
`Reactor` 는 이런 `Overflow/Backpressure` 상황을 좀더 `Gracefully` 하게 적용 할 수 있는 전략으로 아래와 같은 `Backpressure Strategy`를 제공한다.  


Strategy|Desc
---|---
IGNORE|`Downstream` 의 `Backpressure` 를 무시한다.
IGNORE|`Downstream` 에 데이터를 전달 할떄 사용하는 `Buffer` 가 가득 찬 경우, 초과되는 데이터를 무시하고 계속 처리한다. 
ERROR|`Downstream` 에 데이터를 전달 할떄 사용하는 `Buffer` 가 가득 찬 경우, 예외를 발생시켜 처리를 중단한다.
DROP|`Donwstream` 에 데이터를 전달 할떄 사용한는 `Buffer` 가 가득 찬 경우, 초과된 데이터를 무시하고 처리하지 않는다. 
LASTEST|`Downstream` 에 데이터를 전달 할때 사용하는 `Buffer` 가 가득 찬 경우, 초과된 데이터 중 가장 최신 값부터 처리한다. 
BUFFER|`Downstream` 에 데이터를 전달 할때 사용하는 `Buffer` 가 가득 찬 경우, 버퍼에 있는 데이터를 버리고 새롭게 방출된 데이터를 버퍼에 넣는다. 
BUFFER|`Downstream` 에 데이터를 전달 할때 사용하는 `Buffer` 가 가득 찬 경우, 버퍼에 저장해 나중에 처리하도록 한다. 

이러한 처리 전략은 `onBackpressureXXX()` 를 사용해서 `Reactive Stream` 에 쉽게 적용 할 수 있다.  


#### DROP
`DROP` 전략은 `Downstream` 이 너무 `Upstream` 의 데이터 방출을 따라가지 못할 떄, 
가장 최근의 다음 값을 버리는 전략이다. (버퍼 밖에서 대기하는 데이터 중 먼저 방출된 데이터 부터 버린다.)
`DROP` 전략으로 버려진 값들은 `Consumer` 구현을 통해 별도 처리를 수행할 수 있도록 제공한다. 

아래 코드는 보면 `Producer` 는 `0 ~ 499` 까지 데이터를 방출하고,
`Subcriber` 인 `concatMap()` 은 방출 된 데이터를 `10ms` 마다 소비한다. 
이후 다른 전략들에도 동일한 코드에서 `backpressure` 전략만 변경해 어떻게 수행 결과가 달라지는지 살펴본다.  


```java
@Test
public void backpressure_overflow_drop() {
    Flux.<Integer>create(fluxSink -> {
        for(int i = 0; i < 500; i++) {
            log.info("publish : {}", i);
            fluxSink.next(i);
        }

        fluxSink.complete();
    }, FluxSink.OverflowStrategy.DROP)
            .onBackpressureDrop(integer -> log.info("drop : {}", integer))
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.boundedElastic())
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(10)))
            .doOnNext(integer -> log.info("next : {}", integer))
            .doOnError(throwable -> log.error("error", throwable))
            .as(StepVerifier::create)
            .expectNextSequence(IntStream.range(0, 256).boxed().collect(Collectors.toList()))
            .verifyComplete();
}
```  


위 시나리오에서 `DROP` 전략을 사용하면 아래와 같은 출력 결과가 나온다.
`Publisher` 는 앞서 ` 0 ~ 499` 까지 데이터를 모두 방출 한다.
그리고 `Subscriber` 는 비동기로 기본 버퍼 크기인 `256` 개의 데이터인 `0 ~ 255` 의 데이터는 구독자인 `concatMap()` 까지 모두 정상적으로 전달이 된다. 
하지만 그 이후의 데이터는 버퍼가 꽉찬 상태이기 때문에 버려져서 `drop` 로그에 찍힌 것을 확인 할 수 있다.  

```
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 0
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 1
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 2
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 254
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 255
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 256
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - drop : 256
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 257
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - drop : 257
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 258
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - drop : 497
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 498
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - drop : 498
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 499
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - drop : 499
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 0
[parallel-2] INFO com.windowforsun.bachpressure.BackpressureTest - next : 1
[parallel-3] INFO com.windowforsun.bachpressure.BackpressureTest - next : 2
...
[parallel-4] INFO com.windowforsun.bachpressure.BackpressureTest - next : 253
[parallel-5] INFO com.windowforsun.bachpressure.BackpressureTest - next : 254
[parallel-6] INFO com.windowforsun.bachpressure.BackpressureTest - next : 255
```  


#### LATEST
`LATEST` 전략은 `Downstream` 이 `Upstream` 의 전송 속도를 따라가지 못할 때, 
최신 값 부터 처리 하는 전략으로, 
버퍼 밖에서 대기하는 가종 최근에 방출된 데이터 부터 버퍼에 채우는 전략이다.  

```java
@Test
public void backpressure_overflow_latest() {
    Flux.<Integer>create(fluxSink -> {
                for(int i = 0; i < 500; i++) {
                    log.info("publish : {}", i);
                    fluxSink.next(i);
                }

                fluxSink.complete();
            }, FluxSink.OverflowStrategy.LATEST)
            // or
//          .onBackpressureLatest()
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.boundedElastic())
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(10)))
            .doOnNext(integer -> log.info("next : {}", integer))
            .doOnError(throwable -> log.error("error", throwable))
            .as(StepVerifier::create)
            .expectNextSequence(IntStream.range(0, 256).boxed().collect(Collectors.toList()))
            .expectNext(499)
            .verifyComplete();
}
```  

위 시나리오에서 `LATEST` 전략을 사용하면 아래와 같은 출력 결과가 나온다. 
`Publisher` 는 앞서 ` 0 ~ 499` 까지 데이터를 모두 방출 한다. 
그리고 `Subscriber` 는 비동기로 기본 버퍼 크기인 `0 ~ 255` 까지 데이터는 정상적으로 수신하지만, 
그 이후 데이터는 모두 무시되다가 마지막 데이터인 `499` 만 수신하고 스트림은 종료된다. 

```
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 0
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 1
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 2
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 497
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 498
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 499
[parallel-1] INFO com.windowforsun.bachpressure.BackpressureTest - next : 0
[parallel-2] INFO com.windowforsun.bachpressure.BackpressureTest - next : 1
[parallel-3] INFO com.windowforsun.bachpressure.BackpressureTest - next : 2
...
[parallel-5] INFO com.windowforsun.bachpressure.BackpressureTest - next : 254
[parallel-6] INFO com.windowforsun.bachpressure.BackpressureTest - next : 255
[parallel-7] INFO com.windowforsun.bachpressure.BackpressureTest - next : 499
```  


#### ERROR
`ERROR` 전략은 `Downstream` 이 `Upstream` 의 전송 속도를 따라가지 못할 때, 
예외를 발생시켜 `Downstream` 에서 예외처리를 수행 할 수 있다.  

```java
@Test
public void backpressure_overflow_error() {
    Flux.<Integer>create(fluxSink -> {
                for(int i = 0; i < 500; i++) {
                    log.info("publish : {}", i);
                    fluxSink.next(i);
                }

                fluxSink.complete();
            }, FluxSink.OverflowStrategy.ERROR)
            // or
//          .onBackpressureError()
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.boundedElastic())
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(100)))
            .doOnNext(integer -> log.info("next : {}", integer))
            .doOnError(throwable -> log.error("error", throwable))
            .as(StepVerifier::create)
            .expectErrorSatisfies(throwable -> {
                Assertions.assertTrue(throwable instanceof IllegalStateException);
                Assertions.assertTrue(throwable.getMessage().contains("The receiver is overrun by more signals than expected"));
            })
            .verify();
}
```  


위 시나리오에서 `ERROR` 전략을 사용하면 아래와 같은 출력 결과가 나온다.
`Publisher` 는 앞서 ` 0 ~ 499` 까지 데이터를 모두 방출을 시도한다. 
하지만 `Subscriber` 는 `10ms` 주기로 데이터를 소비하는데, 
`Publisher` 는 `10ms` 이내에 `Subscriber` 의 버퍼 크기보다 큰 데이터 개수를 방출하던 과정에서 기본 버퍼 크기에 가까운 `0 ~ 256` 까지는 정상적으로 방출 된다. 
하지만 `257` 부터는 `Overflow` 로 인해 버려지고 `OverflowException` 이 발생하게 되며 `onError()` 를 통해 `Overflow` 에 대한 예외 처리가 가능하다. 

```
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 0
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 1
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 2
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 254
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 255
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 256
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 257
[boundedElastic-2] DEBUG reactor.core.publisher.Operators - onNextDropped: 257
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 258
[boundedElastic-2] DEBUG reactor.core.publisher.Operators - onNextDropped: 258
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 498
[boundedElastic-2] DEBUG reactor.core.publisher.Operators - onNextDropped: 498
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 499
[boundedElastic-2] DEBUG reactor.core.publisher.Operators - onNextDropped: 499
[boundedElastic-1] ERROR com.windowforsun.bachpressure.BackpressureTest - error
reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
	at reactor.core.Exceptions.failWithOverflow(Exceptions.java:220)
	at reactor.core.publisher.FluxCreate$ErrorAsyncSink.onOverflow(FluxCreate.java:687)
	at reactor.core.publisher.FluxCreate$NoOverflowBaseAsyncSink.next(FluxCreate.java:652)
	at reactor.core.publisher.FluxCreate$SerializedFluxSink.next(FluxCreate.java:154)
	at com.windowforsun.bachpressure.BackpressureTest.lambda$backpressure_overflow_error$68(BackpressureTest.java:282)
	at reactor.core.publisher.FluxCreate.subscribe(FluxCreate.java:94)
	at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.run(FluxSubscribeOn.java:193)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
```  


#### IGNORE
`IGNORE` 전략은 `Downstream` 이 `Upstream` 의 전송 속도를 따라가지 못할 때, 
`Backpressure Strategy` 를 적용하지 않는다. 
이는 즉 `Subscriber` 측에서 `Overflow` 에 대한 처리를 해야하고, 
적절한 처리가 되지 않았다면 예외가 발생하게 된다.  

```java
@Test
public void backpressure_overflow_ignore() {
    Flux.<Integer>create(fluxSink -> {
                for(int i = 0; i < 500; i++) {
                    log.info("publish : {}", i);
                    fluxSink.next(i);
                }

                fluxSink.complete();
            }, FluxSink.OverflowStrategy.IGNORE)
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.boundedElastic())
            .concatMap(integer -> Mono.just(integer).delayElement(Duration.ofMillis(10)))
            .doOnNext(integer -> log.info("next : {}", integer))
            .doOnError(throwable -> log.error("error", throwable))
            .as(StepVerifier::create)
            .expectErrorSatisfies(throwable -> {
                Assertions.assertTrue(throwable instanceof IllegalStateException);
                Assertions.assertTrue(throwable.getMessage().contains("Queue is full: Reactive Streams source doesn't respect backpressure"));
            })
            .verify();
}
```  


위 시나리오에서 `IGNORE` 전략을 사용하면 아래와 같은 출력 결과가 나온다.
`Publisher` 는 앞서 ` 0 ~ 499` 까지 데이터를 모두 방출을 시도한다.
하지만 `Subscriber` 는 `10ms` 주기로 데이터를 소비하는데,
`Publisher` 는 `10ms` 이내에 `Subscriber` 의 버퍼 크기보다 큰 데이터 개수를 방출하던 과정에서 기본 버퍼 크기는 모두 차게 된다. 
`Publisher` 는 `Subscriber` 에 `Backpressure` 를 무시하고 아이템을 방출 하기 때문에 `Overflow` 로 `OverflowException` 가 출력된다. 


```
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 0
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 1
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 2
...
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 497
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 498
[boundedElastic-2] INFO com.windowforsun.bachpressure.BackpressureTest - publish : 499
reactor.core.Exceptions$OverflowException: Queue is full: Reactive Streams source doesn't respect backpressure
	at reactor.core.Exceptions.failWithOverflow(Exceptions.java:233)
	at reactor.core.publisher.FluxPublishOn$PublishOnSubscriber.onNext(FluxPublishOn.java:233)
	at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.onNext(FluxSubscribeOn.java:150)
	at reactor.core.publisher.FluxCreate$IgnoreSink.next(FluxCreate.java:618)
	at reactor.core.publisher.FluxCreate$SerializedFluxSink.next(FluxCreate.java:154)
	at com.windowforsun.bachpressure.BackpressureTest.lambda$backpressure_overflow_ignore$73(BackpressureTest.java:312)
	at reactor.core.publisher.FluxCreate.subscribe(FluxCreate.java:94)
	at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.run(FluxSubscribeOn.java:193)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)

```  
