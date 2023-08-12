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


