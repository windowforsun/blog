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

## Reactor Error Handling
`Reactor` 혹은 `Spring Webflux` 에서 `Mono`, `Flux` 를 통해 `Reactive Stream` 을 
처리하다 보면 `Exception` 발생으로 인한 `Error` 처리가 필요한 상황이 많다. 
발생한 `Exception` 을 그대로 던지는 것이 아니라, `Reactive Stream` 상에서 `Graceful` 하게 
`Error` 를 처리할 수 있는 전략에 대해 알아본다.  

`Reactive Stream` 상에서 예외가 발생하면, 발생한 `Reactive Stream` 위치를 기준으로 가장 가까운 `downstream` 의 
`Error Handler` 가 최종적으로 수행된다. 
즉 에러가 발생한 위치에서 `upstream` 에만 `Error Handler` 가 있다면 `Error Handling` 은 수행되지 않는다.

```java
@Test
public void error_handling() {
    Mono.just("2")
            .onErrorReturn("fallback0")
            .flatMap(this::getMonoResult)
            .onErrorReturn("fallback1")
            .onErrorReturn("fallback2")
            .as(StepVerifier::create)
            .expectNext("fallback1")
            .verifyComplete();
}

@Test
public void not_error_handing() {
    Mono.just("2")
            .onErrorReturn("fallback")
            .flatMap(this::getMonoResult)
            .as(StepVerifier::create)
            .expectError(IllegalArgumentException.class)
            .verify();
}
```  

위 경우를 보면 에러가 발생하는 `flatMap(this::getMonoResult)` 에서 가장 가까우면서 `downstream` 인 `fallback1` 
예외처리가 수행되는 것을 확인 할 수 있다.  

`Error` 처리 시뮬레이션을 위해 사용한 `getMonoResult()` 메서드의 형태와 자세한 내용은 아래와 같다. 
이후 예제에서 발생하는 모든 예외는 `i` 값이 짝수일 때 `RuntimeException` 를 상속하는 `IllagalArgumentException` 임을 기억하자. 

```java
public String getResult(String i) {
    log.info("execute getResult : {}", i);

    if(Integer.parseInt(i) % 2 == 0) {
        throw new IllegalArgumentException("test exception");
    }

    return "result:" + i;
}

public Mono<String> getMonoResult(String i) {
    return Mono.fromSupplier(() -> this.getResult(i));
}
```  

`Error Handling` 과 관련된 `Reactor` 메서드들은 아래와 같이 3가지의 조건이 다른 파라미터를 갖는 형태로 구성되는데, 
가장 먼저 알아볼 `onErrorReturn` 을 예시로 들면 아래와 같다.  

```java
// 발생한 모든 예외 대신 방출할 fallback 값을 설정한다. 
public final Mono<T> onErrorReturn(final T fallback);

// 특정 예외 티입이 발생 했을 떄 방출할 fallback 값 설정한다. 
public final <E extends Throwable> Mono<T> onErrorReturn(Class<E> type, T fallbackValue);

// 예외를 별도 Predicate 구현으로 판별해 fallback 값을 방출할지 결정한다. 
public final Mono<T> onErrorReturn(Predicate<? super Throwable> predicate, T fallbackValue);
```  

### onErrorReturn
`onErrorReturn` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때, 
방출 할 지정된 값인 `Fallback value` 를 사용해서 `Error Handling` 하는 방법이다.  

reactor-error-handling-1.svg

```java
@Test
public void mono_onErrorReturn() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorReturn(RuntimeException.class, "fallback")
            .as(StepVerifier::create)
            .expectNext("fallback")
            .verifyComplete();
}

@Test
public void flux_onErrorReturn() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorReturn("fallback")
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("fallback")
            .verifyComplete();
}
```  

`Mono` 예제에서는 `RuntimeException` 에 해당하는 예외에 대해서 `fallback` 이라는 값을 방출하도록
선언 했기 때문에 예외 대신 `fallback` 이라는 문자열이 방출되는 것을 확인 할 수 있다.  

`Flux` 예제를 보면 원본 `source` 에서는 3개의 아이템을 방출하지만, 
첫번째 아이템은 정상적으로 결과를 확인 할 수 있지만 두번째 아이템에서 예외 발생으로 `fallback` 이라는 
문자열이 방출되고 스트림은 종료된다.  

### onErrorResume
`onErrorResume` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때, 
이어서 방출할 `Fallback publisher` 를 사용해서 `Error Handling` 을 하는 방법이다. 

reactor-error-handling-2.svg

```java
@Test
public void mono_onErrorResume() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorResume(throwable -> throwable.getMessage().equals("test exception"),
                    e -> Mono.just("fallback"))
            .doOnNext(s -> log.info("{}", s))
            .as(StepVerifier::create)
            .expectNext("fallback")
            .verifyComplete();
}

@Test
public void flux_onErrorResume() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorResume(throwable -> Flux.fromIterable(List.of("3", "5"))
                    .flatMap(this::getMonoResult))
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:3")
            .expectNext("result:5")
            .verifyComplete();
}
```  

`Mono` 예제에서는 `Predicate` 구현을 통해 `Error message` 가 `test exception` 인 경우 
`fallback` 문자열을 방출하는 스트림을 구독해서 이어 방출하도록 `Error Handling` 을 수행했다.  

`Flux` 예제는 초기 `Source` 는 `1, 2, 3` 3개의 아이템을 방출하도록 하지만
두 번째 아이템인 `2`에서 예외가 발생하게 된다. 
그러면 `3, 5` 를 사용해서 결과를 만드는 스트림을 구독해서 이어 방출하도록 `Error Handling` 을 수행했다.  

### onErrorMap
`onErrorMap` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때,
던질 예외를 `Mapper` 를 사용해서 필요한 예외 타입으로 변경해서 `Error Handling` 을 하는 방법이다.

reactor-error-handling-3.svg

```java
@Test
public void mono_onErrorMap() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorMap(throwable -> new ClassNotFoundException())
            .as(StepVerifier::create)
            .expectError(ClassNotFoundException.class)
            .verify();
}

@Test
public void flux_onErrorMap() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorMap(throwable -> new ClassNotFoundException())
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectError(ClassNotFoundException.class)
            .verify();
}
```  
