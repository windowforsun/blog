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

## Debugging Reactor
[Debugging Reactor](https://projectreactor.io/docs/core/release/reference/#debugging) 
`Reactor` 공식문서 중 `Debugging` 관련 챕터를 보면 `Reactor` 를 `Debugging` 하는 방법에 대해 설명이 있다. 
이번 포스트에서는 이를 기반으로 `Reactor` 를 `Debugging` 하는 다양한 방법에 대해 알아본다.  

비동기 프로그래밍을 접할 떄 가장 어려운 부분이 `Debugging` 이지만, `Reactor` 에서는 
이를 간단하게 수행 할 수 있는 몇가지 도구를 제공한다.  

이후 디버깅예제에서 사용할 예제인 `TestSource` 를 먼저 살펴본다.  

```java
@Slf4j
public class TestSource {
    private static int square(int i) {
        log.info("execute getResult : {}", i);

        if(i % 2 == 0) {
            throw new IllegalArgumentException("test exception");
        }

        return i * i;
    }

    public static Mono<Integer> produceSquare(int i) {
        return Mono.just(i)
                .flatMap(s -> Mono.just(s)
                        .flatMap(s2 -> Mono.just(square(s2))))
                .delayElement(Duration.ofMillis(10))
                .subscribeOn(Schedulers.parallel());
    }
}
```  

`square(int)` 메소드는 이름 그대로 제곱의 결과를 리턴하는데, 인수로 받은 값이 짝수인 경우에는 예외를 던지게 된다. 
그리고 `square(int)` 를 사용해서 `Reactive Stream` 을 생성하는 `produceSquare(int)` 의 스트림 구성은 
고의적으로 복잡한 구성을 해 놓았다. 
이후 예제에서 이런 `Reactive Stream` 구성에서 어떤식으로 디버깅이 가능한지 알아본다.  
