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

## Flux Parallel
`Flux Parallel` 이란 일반적으로 `Reactive Stream` 에서 `Producer` 는 시퀀스를 순차적으로 `next` 신호를 발생하고, 
이를 구독하는 `Subscriber` 는 순차적으로 신호를 처리하게 된다. 
하나의 `Producer` 의 `next` 신호를 병렬로 처리할 수 있는데, 
바로 `Flux` 의 `parallel()` 과 `runOn()` 을 사용하는 것이다.  

> 여기서 `parallel()` 과 `runOn()` 의 병렬처리와 `publishOn()` 과 `subscribeOn()` 의
> 병렬 처리의 목적은 구분이 필요하다. 
> `publishOn()` 과 `subscribeOn()` 에서 수행하는 병렬 처리는 각기 다른 `Reactive Stream` 을 병렬로 처리할 수 있도록 하는데 목적이 있다. 
> 이와 다르게 `parallel()` 과 `runOn()` 의 병렬처리는 하나의 `Reactive Stream` 에서 `downstream` 으로 보내는 `next` 신호를 벙렬로 처리하도록 하는데 목적이 있다.  

### Flux.parallel()
[Flux.parallel()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#parallel--)
은 하나의 `Flux Stream` 을 인수로 받은 `parallelism` 의 수 만큼 `Round-robin` 방식으로 방출되는 신호를 나눠 `parallelism` 과 동일한 수의 `rails` 로 구성한다. 
여기서 `parallel()` 은 단순히 하나의 `Flux Stream` 을 여러 `rails` 로 나누는 역할만 하기 때문에 실제 병렬 처리를 위해서는 `runOn()` 과 함께 사용이 필요하다. 
`runOn()` 에 대해서는 추후에 더 살펴본다.  

```java
@Test
public void parallel_test() {
    Flux.range(1, 20)
            // 레일 수 지정
            .parallel(4)
            .log()
            .doOnNext(integer -> log.info("receive : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(20)
            .verifyComplete();
}
```  

위 코드를 실행한 출력 로그는 아래와 같다.  

```
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSource
[main] INFO reactor.Parallel.Source.1 - request(256)
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSource
[main] INFO reactor.Parallel.Source.1 - request(256)
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSource
[main] INFO reactor.Parallel.Source.1 - request(256)
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSource
[main] INFO reactor.Parallel.Source.1 - request(256)
[main] INFO reactor.Parallel.Source.1 - onNext(1)
[main] INFO com.windowforsun.flux.ParallelTest - receive : 1
[main] INFO reactor.Parallel.Source.1 - onNext(2)
[main] INFO com.windowforsun.flux.ParallelTest - receive : 2
[main] INFO reactor.Parallel.Source.1 - onNext(3)
[main] INFO com.windowforsun.flux.ParallelTest - receive : 3
[main] INFO reactor.Parallel.Source.1 - onNext(4)
[main] INFO com.windowforsun.flux.ParallelTest - receive : 4

.. 생략 ..

[main] INFO com.windowforsun.flux.ParallelTest - receive : 19
[main] INFO reactor.Parallel.Source.1 - onNext(20)
[main] INFO com.windowforsun.flux.ParallelTest - receive : 20
[main] INFO reactor.Parallel.Source.1 - onComplete()
[main] INFO reactor.Parallel.Source.1 - onComplete()
[main] INFO reactor.Parallel.Source.1 - onComplete()
[main] INFO reactor.Parallel.Source.1 - onComplete()
```  

`parallel(4)` 을 `Reactive Stream` 에 추가하면, 위 출력 로그와 같이 
`parallelism` 수인 4개의 `Subscriber` 가 `Flux Stream` 을 구독해서, 
방출하는 `1 ~ 20` 까지의 신호를 나눠서 수신하는 것을 확인 할 수 있다. 
그러므로 `onComplete()` 의 신호도 `parallelism` 수 만큼 이뤄지는 것을 확인 할 수 있다. 
하지만 모든 처리는 동일한 스레드인 `main` 에서 이뤄지기 때문에 실제 신호 처리는 병렬로 수행되지 않는 것을 확인 할 수 있다. 
즉 `parallel` 은 하나의 `Flux Stream` 을 나눠 여러 스트림에 구독 시키는 동작만 수행하는 것을 확인 할 수 있다.  

### ParallelFlux.runOn()
[ParallelFlux.runOn](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/ParallelFlux.html#runOn-reactor.core.scheduler.Scheduler-)
은 `Flux.parallel()` 의 리턴 값인 `ParallelFlux` 에서 `Schedulers` 를 기반으로 
병렬처리를 수행할 스레드를 할당 할 수 있다. 
`Flux.parallel()` 을 통해 나눠진 `rails` 의 실제 수행이 `runOn()` 에서 지정해준 `Schedulers` 를 통해 수행된다고 생각하면 된다.  


```java
@Test
public void parallel_runOn_test() {
    Flux.range(1, 20)
            // 레일 수 지정
            .parallel(4)
            // 레일에서 사용할 scheduler 지정
            .runOn(Schedulers.newParallel("myParallel", 4))
            .log()
            .doOnNext(integer -> log.info("receive : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(20)
            .verifyComplete();
}
```  

```
[main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[main] INFO reactor.Parallel.RunOn.1 - request(256)
[main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[main] INFO reactor.Parallel.RunOn.1 - request(256)
[main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[main] INFO reactor.Parallel.RunOn.1 - request(256)
[main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[main] INFO reactor.Parallel.RunOn.1 - request(256)
[myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(2)
[myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(1)
[myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 2
[myParallel-3] INFO reactor.Parallel.RunOn.1 - onNext(3)
[myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(6)
[myParallel-3] INFO com.windowforsun.flux.ParallelTest - receive : 3
[myParallel-4] INFO reactor.Parallel.RunOn.1 - onNext(4)
[myParallel-3] INFO reactor.Parallel.RunOn.1 - onNext(7)
[myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 1
[myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 6
[myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(5)
[myParallel-4] INFO com.windowforsun.flux.ParallelTest - receive : 4

.. 생략 ..

[myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(13)
[myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 13
[myParallel-4] INFO reactor.Parallel.RunOn.1 - onComplete()
[myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(17)
[myParallel-3] INFO reactor.Parallel.RunOn.1 - onNext(15)
[myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 17
[myParallel-3] INFO com.windowforsun.flux.ParallelTest - receive : 15
[myParallel-1] INFO reactor.Parallel.RunOn.1 - onComplete()
[myParallel-3] INFO reactor.Parallel.RunOn.1 - onNext(19)
[myParallel-3] INFO com.windowforsun.flux.ParallelTest - receive : 19
[myParallel-3] INFO reactor.Parallel.RunOn.1 - onComplete()
```  

`Flux.parallel()` 에서 사용한 예제에서 `parallel()` 아래 `runOn()` 을 추가하게 되면, 
위와 같이 `Flux Stream` 의 신호처리가 병렬로 수행되는 것을 확인 할 수 있다. 
`parallel(4)` 에서 `1 ~ 20` 으로 전달되는 신호를 4개의 `rails` 로 분리하면, 
`runOn(Schedulers.newParallel("myParallel", 4)` 를 통해 각 `rail` 당 하나의 스레드가 사용될 수 있도록 
지정해서 모든 레일이 병렬로 처리 될 수 있도록 했다.  

다시 한번 설명하면, 
`Flux.parallel()` 은 `Flux Stream` 을 여러 동시성을 위해 `rails` 로 분리하기만 하고 
`ParallelFlux.runOn()` 에서 분리된 `rails` 가 실제로 병렬로 처리 될 수 있도록하는 `Schedulers` 를 할당해야
우리가 기대한 것처럼 병렬처리가 수행된다.  

### Parallelism test
실제로 `Flux Stream` 에서 방출되는 모든 신호들이 원하는 `paralism` 에 맞춰 병럴로 수행되는지 검증을 해본다. 
검증 방식은 `1 ~ 10` 까지 수가 방출되는 `Flux Stream` 에서 짝수는 지연이 수행되지 않고, 
홀수인 경우에는 `100ms` 정도 지연을 발생시켜보면 아래와 같다.  

```java
@Test
public void parallelism_test() {
    Flux.range(1, 10)
            // 레일 수 지정
            .parallel(2)
            // 레일에 사용할 scheduler 지정
            .runOn(Schedulers.newParallel("myParallel", 2))
            .map(integer -> {
                try {
                    if(integer % 2 != 0) {
                        Thread.sleep(100);
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }

                return integer;
            })
            .doOnNext(integer -> log.info("receive : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(10)
            .verifyComplete();
}
```  

```
18:53:34.296 [main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:53:34.297 [main] INFO reactor.Parallel.RunOn.1 - request(256)
18:53:34.298 [main] INFO reactor.Parallel.RunOn.1 - onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:53:34.298 [main] INFO reactor.Parallel.RunOn.1 - request(256)
18:53:34.298 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(1)
18:53:34.298 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(2)
18:53:34.299 [myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 2
18:53:34.299 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(4)
18:53:34.299 [myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 4
18:53:34.299 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(6)
18:53:34.299 [myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 6
18:53:34.299 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(8)
18:53:34.299 [myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 8
18:53:34.299 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onNext(10)
18:53:34.299 [myParallel-2] INFO com.windowforsun.flux.ParallelTest - receive : 10
18:53:34.299 [myParallel-2] INFO reactor.Parallel.RunOn.1 - onComplete()
18:53:34.404 [myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 1
18:53:34.404 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(3)
18:53:34.507 [myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 3
18:53:34.508 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(5)
18:53:34.612 [myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 5
18:53:34.612 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(7)
18:53:34.717 [myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 7
18:53:34.718 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onNext(9)
18:53:34.823 [myParallel-1] INFO com.windowforsun.flux.ParallelTest - receive : 9
18:53:34.823 [myParallel-1] INFO reactor.Parallel.RunOn.1 - onComplete()
```  

`parallel(2)` 로 2개의 `rails` 를 지정하고, 
`runOn()` 을 통해 2개의 동시성을 갖는 `Scheduler` 를 지정해 주었다. 
각 `rail` 이 하나의 스레드와 매핑되어 사용되기 때문에,
짝수, 홀수에 대한 첫 `next` 신호는 동시에 발생 했다. 
하지만 짝수에 해당하는 `rail` 먼저 모든 시퀀스를 수신 한 다음 `onComplete` 신호까지 수신해 완료 되었다. 
짝수가 완료된 `100ms` 이후 부터 홀수에 대한 시퀀스가 `100ms` 주기로 발생해서 `onComplete` 신호까지 완료 되었다. 
이를 통해 `parallel()` 과 `runOn()` 을 사용하면 `Flux Stream` 에서 방출되는 신호를 원하는 동시성을 할당해,
기대하는 병렬처리 동작을 수행 할 수 있음을 다시 확인 해 보았다.  

