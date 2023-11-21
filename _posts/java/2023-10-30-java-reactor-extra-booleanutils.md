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

## BooleanUtils
[BooleanUtils](https://projectreactor.io/docs/extra/release/api/index.html?reactor/function/TupleUtils.html)
는 `Reactor Extra` 에서 제공하는 유틸성 클리스 중 하나로, 
`Boolean` 을 방출하는 `Mono` 2개d의 `upstream` 을 구독해 `or`, `and` 와 같은 논리 연산의 수행해 결과값을 `downstream` 으로 방출하는 정적 메소드를 제공한다.  

아래는 `Boolean` 을 반환하는 2개의 `upstream` 의 결과를 `and` 연산을 수행하는 예제이다.  

```java
Mono.zip(Mono.just(true), Mono.just(false))
        .map(tuple2 -> tuple2.getT1() && tuple2.getT2())
```  

2개 이상의 결과를 소스로 결과를 만들어야 하기 때문에 `zip` 연산을 사용해 2개의 `Boolean` 결과를 병합하고, 
`downstream` 인 `map` 에서 `and` 연산을 수행하는 방법으로 구현할 수 있다.  
