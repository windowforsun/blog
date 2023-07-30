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

## Reactive Stream Zip Operation
`Reactive Stream` 에서 `Zip` 연산자는 서비스에서 그룹 데이터에 대한 결과가 필요할 때 유용하다. 
여기서 그룹 데이터란 여러 데이터를 병합한 데이터로, 
`Reactive Stream` 입장에서는 여러 `Stream` 의 데이터를 모아오는 것이라고 할 수 있다.  

예제 및 설명은 `Reactor Project` 의 `Zip Operation` 을 기반으로 진행한다.  

`Reactive Stream` 에서 `zip` 연산은 여러 `Reactive Stream` 의 결과를 하나로 결합한다는 점에서 
아래 예시인 `flatMap` 을 사용한 것과 결과적으론 동일하다.  

```java
@Test
public void zip_of_flatMap() {
    Mono.just("first")
            .flatMap(s -> Mono.just("second").map(s1 -> Tuples.of(s, s1)))
            .flatMap(tuples2 -> Mono.just("third").map(s -> Tuples.of(tuples2.getT1(), tuples2.getT2(), s)))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", "second", "third"))
            .verifyComplete();
}
```  

`first`, `second`, `third` 를 방출하는 `Mono stream` 3개를 `flatMap` 을 사용해 결합하는 코드이다. 
최종적으로 수행된 결과는 `zip` 연산과 동일하지만, 
`zip` 을 사용하면 수행하고자 하는 동작을 더욱 명확하게 표현할 수 있고 코드의 간결성도 얻을 수 있다.  

### How mono zip work
`Reactor Proejct` 의 `Zip` 이 어떻게 동작하는지 살펴보자. 
아래는 [Mono.zip](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zip-reactor.core.publisher.Mono-reactor.core.publisher.Mono-)
의 `API Document` 에서 발췌한 마블 다이어그램이다.  

reactive-stream-reactor-zip-1.svg

`Mono.zip` 연산은 2개 이상의 스트림을 사용해서 구성할 수 있다. 
수행되는 시그널을 간력하게 설명하면 아래와 같다.  

1. `Mono.zip` 에 포함된 모든 `Stream` 에 `Subscribe` 시그널과 함께 구독 수행
2. 모든 `Stream` 이 결과를 방출 할 때까지 대기
3. 모든 `Stream` 이 완료
4. 모든 결과를 포함한 `Tuple`(혹은 특정 객체)를 생성
5. `downstream` 으로 `Complete` 시그널 전송

`Reactor Project` 에서 `Mono.zip` 은 2개에서 부터 최대 8개 까지 `Stream` 을 `zip` 메소드 인수로 사용할 수 있다. 
만약 8개 이상의 `Stream` 에 대한 처리가 필요한 경우 `Iterable` 타입으로 전달 할 수 있다. 
또한 결과 타입은 기본적으로 `Tuple` 로 제공되고, 필요한 경우 `Combinator` 를 사용해 직접 필요한 객체로 병합하는 동작을 수행 할 수 있다. 
`Tuple` 또한 최소 2개에서 최대 8개의 결과를 담을 수 있도록 제공된다.  

### How flux zip work
`Reactor Proejct` 의 `Zip` 이 어떻게 동작하는지 살펴보자. 
먼저 살펴본 `Mono.zip` 의 동작 방식과 크게 다르지 않다. 
아래는 [Flux.zip](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#zip-org.reactivestreams.Publisher-org.reactivestreams.Publisher-)
의 `API Document` 에서 발췌한 마블 다이어그램이다. 

reactive-stream-reactor-zip-2.svg

`Flux.zip` 또한 2개 이상의 무한 스트림을 사용해서 구생할 수 있다. 
수행되는 시그널에 대한 설명은 아래와 같다.  

1. `Flux.zip` 에 포함된 모든 `Stream` 에 `Subscribe` 시그널과 함께 구독 수행
2. 모든 `Stream` 이 각 하나의 결과를 방출 할 떄까지 대기
3. 모든 `Stream` 의 결과를 하나씩 포함한 `Tuple`(혹은 특정 객체)를 생성
4. 어느 하나 `Stream` 에서 `Complete` 시그널이 전달 될떄까지 2번부터 다시 수행

`Flux.zip` 또한 최소 2개에서 최대 8개까지의 `Stream` 을 사용할 수 있고, 
그 이상은 `Iterable` 타입으로 전달이 필요한 것과, 
기본적으론 `Tuple` 이 결과이고 필요한 경우 `Combinator` 를 통해 필요한 객체를 바탕으로 병합이 가능하다는 내용까지는 `Mono.zip` 과 동일하다.  

`Flux.zip` 에서 더욱 중요한 부분이 바로 데이터가 방출되는 타이밍이다. 
결과(`Tuple`)를 생성할 때 `Flux.zip` 은 포함된 `Stream` 중 가장 느린 `Stream` 과 
데이터의 수가 가장 적은 `Stream` 을 기반으로 동작 한다는 점을 기억해야 한다. 
자세한 내용은 추후 예제에서 관련 내용을 확인 할 수 있다.  


### All or Nothing
`Mono`, `Flux` 의 `zip` 연산은 `All or Nothing` 성질을 가지고 있다. 
즉 전체 스트름이 아닌 아닌 어느 부분의 스트림 결과는 제공하지 않는 다는 의미이다. 
`zip` 에 포함된 전체 스트림 중 하나라도 결과를 보내지 않는다면 해당 `zip` 연산은 완료되지 않는다. 
`empty` 결과가 오더라도 동일하고, 스트림 중 어느 하나라도 실패한다면 전체 `zip` 연산은 실패 결과를 내보낸다. 
이런 성질이 있는 상태에서 기본 값 혹은 예외처리가 필요하면 `Optional` 을 사용해서 어느정도 회픠할 수 있다. 
이처럼 보다 정교한 결과 수행을 위해서는 성질에 맞는 추가 구현이 필요할 수 있다.  

이후 예제에서 기본 결과를 만드는 방법이나, 
예외처리를 수행하는 방법에 대해 알아본다.  


---
## Reference
[]()  


