--- 
layout: single
classes: wide
title: "[Java 실습] Reactive Stream, Reactor Flux Parallel"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactive Stream 의 Flux 에서 다량의 데이터 처리를 병렬로 수행하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Flux
  - parallel
  - ParallelFlux
  - runOn
  - rains
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


### Performance test
`Flux Stream` 을 병렬처리로 사용하려는 것은 성능적인 이점을 보기 위함일 것이다. 
직접 간단한 성능 테스트를 해보며 어느 정도의 성능 차이가 있는지 살펴본다. 
성능 테스트에 사용할 `Flux Stream` 은 아래와 같다.  

```java
Flux.range(1, 400)
        .map(integer -> {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            return integer;
        })
```  

`1 ~ 400` 의 정수를 방출하는 `Flux Stream` 인데 시퀀스 하나를 방출하기 까지 `10ms` 정도가 소요된다는 것을 가정했다. 
위 `Flux Stream` 에 대해서 소요시간을 확인해보면 아래의 검증 코드처럼 `4000ms` 정도가 소요되는 것을 확인 할 수 있다.  


```java
@Test
public void performance_exam() {
    Flux.range(1, 400)
            .map(integer -> {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }

                return integer;
            })
            .as(StepVerifier::create)
            .expectNextCount(400)
            .expectComplete()
            .verifyThenAssertThat()
            .tookMoreThan(Duration.ofMillis(4000))
            .tookLessThan(Duration.ofMillis(5000));
}
```  

#### publishOn, subscribeOn 과 비교

`Flux Parallel` 을 사용해보기 전에 정말로 `Scheduler` 의 대표적인 사용처인 `publishOn()` 과 `subscribeOn()` 을 사용하면, 
위와 같은 예제에서 성능적인 이점이 없는지 확인 해보고 넘어가 본다. 

```java
@Test
public void performance_bad() {
    Flux.range(1, 400)
            .map(integer -> {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }

                return integer;
            })
            .publishOn(Schedulers.boundedElastic())
            .subscribeOn(Schedulers.boundedElastic())
            .as(StepVerifier::create)
            .expectNextCount(400)
            .expectComplete()
            .verifyThenAssertThat()
            .tookMoreThan(Duration.ofMillis(4000))
            .tookLessThan(Duration.ofMillis(5000));
}
```  

위 코드를 보면 `publishOn()` 과 `subscribeOn()` 을 사용해서 `Scheduler` 를 할당해 주었다. 
하지만 앞서 설명한 것처럼 `publishOn()` 과 `subscribeOn()` 은 `Reactive Stream` 이 어느 `thread` 에서 실행 될지 할당해주는 역할이기 때문에,  
위 상황에서는 동일한 소요시간을 보여준다. 
해당 코드는 `10ms` 가 소요되는 `Blocking` 동작을 `parallel thread` 가 아닌 `elastic thread` 로 할당해 
여러 `Reactive Stream` 이 수행되는 전체 애플리케이션 관점에서는 성능적인 이점은 가져다 줄 수 있다.  

#### parallel, runOn 사용 결과

이제 `Flux parallel` 을 사용해 결과를 살펴보자. 
한가지 주의점은 아래와 같이 `parallel()` 과 `runOn()` 이 위치 한다면 기대하는 동작은 수행되지 못하고, 
동일한 `4000ms` 의 소요시간이 걸린다. 
즉 `Blocking` 혹은 `Delay` 가 발생하는 스트림 전에 `parallel()` 과 `runOn()` 을 사용해야 
그 이후 `Reactive Stream` 부터 병렬 처리가 가능하다.  

```java
// 기대하는 것처럼 병렬로 수행되지 못함
Flux.range(1, 400)
        .map(integer -> {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            return integer;
        })
        .parallel(4)
        .runOn(Schedulers.newParallel("myTest", 4))
```  

4개의 `Rails` 과 4개의 `thread pool` 을 할당해준다면 `rail` 당 100개의 시퀀스가 할당되고, 
시퀀스 마다 `10ms` 가 소요될 것이기 때문에 총 소요시간은 `1000ms` 정도를 기대할 수 있을 것이다. 
아래 코드를 통해 기대하는 것과 동일한 결과를 보여주는지 살펴보자.  

```java
@Test
public void performance_good() {
    Flux.range(1, 400)
            .parallel(4)
            .runOn(Schedulers.newParallel("myTest", 4))
            .map(integer -> {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }

                return integer;
            })
            .as(StepVerifier::create)
            .expectNextCount(400)
            .expectComplete()
            .verifyThenAssertThat()
            .tookMoreThan(Duration.ofMillis(1000))
            .tookLessThan(Duration.ofMillis(1500));
}
```  

최종 결과를 보면 앞서 계산한 것과 동일하게 `1000ms` 정도의 소요시간이 걸린 것을 확인 할 수 있다. 
이렇게 `parallel()` 과 `runOn()` 을 통해 `Flux Stream` 을 벙렬처리 한다면, 
기존 동작에 퍼포먼스를 개선 할 수 있을 것이다.  


### Rails parallelism, Scheduler parallelism
`Flux Stream` 을 병렬로 처리하는 방법을 위해서는 
`parallel()` 을 통해 `upstream` 을 몇개로 나눌지에 대한 `rail` 수를 지정하고, 
나눠진 `rail` 의 신호를 처리할 스레드를 `runOn()` 을 통해 `Scheduler` 를 사용해서 할당한다. 
이때 `rail` 의 수가 스레드의 수보다 많은 경우, 
스레드는 스레드 수에 맞는 `rail` 를 각 선턱해 해당 `rail` 의 데이터를 먼저 처리하고 
남은 데이터가 없으면 다른 `rail` 을 선택해서 처리하게 된다. 
즉 `rail` 에 있는 데이터의 수에 따라 스케줄러가 선택하는 `rail` 이 달라지게 된다.  

반대로 `rail` 수 보다 스레드의 수가 더 많으면 `rail` 은 각 스레드에 모두 할당 될 수 있지만, 
전체 스레드가 모두 `rail` 신호를 처리하는 스레드로 사용되지는 않는다.  

먼저 `rail` 수 > 스레드 수인 상황을 살펴보자.  

```java
@Test
public void railCount_Gt_ThreadCount() {
    Flux.range(1, 12)
            .parallel(4)
            .runOn(Schedulers.newParallel("myTest", 2))
            .doOnNext(integer -> log.info("receive : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(12)
            .verifyComplete();
}
```  

코드를 실행 했을 떄 출력되는 로그는 아래와 같다.  

```
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 1
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 2
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 5
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 6
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 9
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 10
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 3
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 7
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 11
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 4
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 8
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 12
```  

로그를 함께보며 `rail` 의 분류와 스레드 할당 과정을 함께 설명하면 아래와 같다. 
먼저 아래와 같이 `1 ~ 12` 의 데이터가 4개의 `rail` 로 분류 될 것이다.  

```
rail-0 : 1, 5, 9
rail-1 : 2, 6, 10
rail-2 : 3, 7, 11
rail-3 : 4, 9, 12
```  

가용 가능한 스레드인 `myTest-1` 과 `myTest-2` 스레드는 최초에 `rail-0` 과 `rail-1` 을 각각 선택해서 데이터를 처리한다.  

```
rail-0 : 1, 5, 9  (myTest-1)
rail-1 : 2, 6, 10 (myTest-2)
rail-2 : 3, 7, 11
rail-3 : 4, 9, 12
```  

`rail-0` 과 `rail-1` 에 데이터가 더 이상 없으면 `myTest-1` 과 `myTest-2` 은 `rail-2` 과 `rail-3` 을 각각 선택해서 이어서 데이터를 처리하면, 
위 로그와 동일한 순서로 데이터가 처리되는 것을 확인 할 수 있다.  

```
rail-0 : 
rail-1 : 
rail-2 : 3, 7, 11 (myTest-1)
rail-3 : 4, 9, 12 (myTest-2)
```  

다음은 `rail` 수 < 스레드 수 인 경우를 살펴보자. 

```java
@Test
public void railCount_Lt_ThreadCount() {
    Flux.range(1, 12)
            .parallel(2)
            .runOn(Schedulers.newParallel("myTest", 4))
            .doOnNext(integer -> log.info("receive : {}", integer))
            .as(StepVerifier::create)
            .expectNextCount(12)
            .verifyComplete();
}
```  

출력 로그는 아래와 같다.  

```
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 1
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 2
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 3
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 4
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 5
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 6
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 7
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 8
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 9
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 10
[myTest-1] INFO com.windowforsun.bachpressure.ParallelTest - receive : 11
[myTest-2] INFO com.windowforsun.bachpressure.ParallelTest - receive : 12
```  

최초에 `rail` 분류는 아래와 같다. 

```
rail-0 : 1, 3, 5, 7, 9, 11
rail-1 : 2, 4, 6, 8, 10, 12
```  

가용 가능한 스레드는 총 4개로 `myTest-1`, `myTest-2`, `myTest-3`, `myTest-4` 가 있지만, 
실제로 할당해서 데이터를 처리하는 스레드는 `myTest-1`, `myTest-2` 만이다.  

```
rail-0 : 1, 3, 5, 7, 9, 11 (myTest-1)
rail-1 : 2, 4, 6, 8, 10, 12 (myTest-2)
```  

위와 같은 상황은 만약 `Scheduler` 객체를 여러 스트림에서 공유한다면 다른 스트림에 대한 처리가, 
`myTest-3`, `myTest-4` 에 할당돼 여러 스트림을 병렬로 처리하는 식으로 공유 할 수 있다.  


### Rail buffer
`Flux Stream` 의 `Parallel` 동작에서 `Scheduler` 에 대한 정의를 해주는 [runOn()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/ParallelFlux.html#runOn-reactor.core.scheduler.Scheduler-int-) 
메서드에는 `prefetch` 라는 `Backpressure` 관련 인수값이 있다. 
`prefetch` 는 `rail` 당 `Queues.SMALL_BUFFER_SIZE` 만큼 데이터를 저장하는데, 
해당 값은 `reactor.bufferSize.small` 라는 시스템 프로퍼티 값으로 조정 가능하다. 
이 값이 지정돼있지 않으면 256 을 사용하고 16보다 작은 경우에는 16을 사용한다.  

정리하면 `parallel()` 과 `runOn()` 을 통해 `rail` 을 생성하면, 
각 `rail` 은 별도로 설정하지 않았을 때 기본으로 `request(256)` 으로 256 개의 데이터를 요청한다. 
확인을 위해 아래 코드와 로그 결과를 살펴보자.  

```java
@Test
public void default_buffer() {
    Flux.range(1, 1000)
            .parallel(2)
            .log()
            .runOn(Schedulers.newParallel("myTest", 2))
            .as(StepVerifier::create)
            .expectNextCount(1000)
            .verifyComplete();
}
```  

위 코드를 실행 했을 떄 로그를 보면 아래와 같이, 
`upstream` 으로 부터 `request(256)` 으로 256개의(이후에는 192) 데이터를 요청하고 버퍼에 받아 `downstream` 으로 필요한 만큼 다시 전달하게 된다.  

```
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSourceInner)
[main] INFO reactor.Parallel.Source.1 - request(256)
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSourceInner)
[main] INFO reactor.Parallel.Source.1 - request(256)

... onNext() ...

[myTest-1] INFO reactor.Parallel.Source.1 - request(192)
[myTest-2] INFO reactor.Parallel.Source.1 - request(192)

... onNext() ...

[myTest-1] INFO reactor.Parallel.Source.1 - request(192)
[myTest-2] INFO reactor.Parallel.Source.1 - request(192)

...
```  

`prefetch` 인수를 사용하면 처리에 필요한 만큼 버퍼 크기인 `Backpressure` 를 조정할 수 있다. 
아래 코드는 `prefetch` 를 10으로 지정했을 떄의 코드와 로그인다.  

```java
@Test
public void custom_buffer() {
    Flux.range(1, 1000)
            .parallel(2)
            .log()
            .runOn(Schedulers.newParallel("myTest", 2), 10)
            .as(StepVerifier::create)
            .expectNextCount(1000)
            .verifyComplete();
}
```  

위 코드는 `upstream` 으로부터 `request(10)` 으로 10개의(이후에는 8) 데이터를 요청하고 버퍼에 받아 `downstream` 으로 필요한 만큼 전달한다. 

```
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSourceInner)
[main] INFO reactor.Parallel.Source.1 - request(10)
[main] INFO reactor.Parallel.Source.1 - onSubscribe(ParallelSource.ParallelSourceMain.ParallelSourceInner)
[main] INFO reactor.Parallel.Source.1 - request(10)

... onNext() ...

[myTest-1] INFO reactor.Parallel.Source.1 - request(8)
[myTest-2] INFO reactor.Parallel.Source.1 - request(8)

... onNext() ...

[myTest-1] INFO reactor.Parallel.Source.1 - request(8)
[myTest-2] INFO reactor.Parallel.Source.1 - request(8)

...
```  

이렇게 `runOn()` 의 `prefetch` 인수를 사용하면 `Flux Stream` 의 병렬 처리에 있어서 필요한 양만큼 `Backpressure` 조정이 가능하다.  



---
## Reference
[Flux.parallel()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#parallel-int-)  
[ParallelFlux.runOn()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/ParallelFlux.html#runOn-reactor.core.scheduler.Scheduler-)  


