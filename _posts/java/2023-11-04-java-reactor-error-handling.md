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
