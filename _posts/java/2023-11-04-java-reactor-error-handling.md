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
