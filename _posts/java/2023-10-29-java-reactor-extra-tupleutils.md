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
  - TupleUtils
  - Zip
toc: true 
use_math: true
---  

## TupleUtils
[TupleUtils](https://projectreactor.io/docs/extra/release/api/index.html?reactor/function/TupleUtils.html)
는 `Reactor Extra` 에서 제공하는 유틸성 클리스 중 하나로, 
`zip` 연산 이후 `upstream` 에서 병합된 `Tuple` 에 담긴 결과를 `downstream` 에서 받아 처리할 떄의 불편함을 해소 할 수 있는 유틸리티성 클래스이다. 
`downstream` 에서 결과가 머지된 `Tuple` 의 데이터를 변환이나 추가 처리를 위해서는 값을 아래와 같이 하나씩 가져 오는 작업이 필요하다.  

```java
Mono.zip(Mono.just("a"), Mono.just(1), Mono.just(1L), Mono.just(1d), Mono.just(true))
        .map(tuple5 -> {
            String str = tuple5.getT1();
            int i = tuple5.getT2();
            long l = tuple5.getT3();
            double d = tuple5.getT4();
            boolean b = tuple5.getT5();
            
            return new Result(str, i, l, d, b);
        })
```  
