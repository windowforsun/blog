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


### Mono.zip
`Mono Stream` 을 결합하는 여러 `Mono.zip` 관련 메소드에 대해 알아본다. 

#### Mono.zip tuple
`zip` 연산 중 가장 간단한 형태로, 
n개의 `Mono upstream` 소스의 결과를 받아 하나의 `Tuple` 로 반환하는 `zip` 함수이다. 
`zip` 의 인수의 개수와 동일한 요소를 가지는 `Tuple` 을 반환한다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zip-reactor.core.publisher.Mono-reactor.core.publisher.Mono-
 */
@Test
public void mono_zip_return_tuple() {
    Mono.zip(Mono.just("first"), Mono.just("second"))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", "second"))
            .verifyComplete();
}
```  

`first` 를 방출하는 `Mono` 와 `second` 를 방출하는 `Mono` 를 `Tuple` 로 결합한 예시이다.  

#### Mono.zip combinator
여러 `Mono upstream` 소스의 결과를 바탕으로 원하는 타입과 값으로 결합해서 최종 결과를 방출 할 수 있다. 
`Mono.zip Tuple` 과는 지정된 결과 타입만 도출하는 것이 아니라, 원하는 결과 타입을 도출할 수 있도록 직접 `combinator` 를 구현하면 된다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zip-java.lang.Iterable-java.util.function.Function-
 */
@Test
public void mono_zip_combinator() {
    Mono.zip(List.of(Mono.just("first"), Mono.just("second")),
                    // combinator
                    objects -> Map.of("key1", objects[0], "key2", objects[1]))
            .as(StepVerifier::create)
            .expectNext(Map.of("key1", "first", "key2", "second"))
            .verifyComplete();
}
```  

`first` 를 방출하는 `Mono` 와 `second` 를 방출하는 `Mono` 를 `combinator` 를 사용해서 `Map` 르로 결합한 예시이다.  

만약 N개의 `Mono` 로 구성된 `List` 를 하나로 결합하고 싶은 경우에도 `Mono.zip combinator` 를 사용할 수 있는데, 
그 예시는 아래와 같다.  

```java
Mono.zip(List.of(Mono.just("first"), Mono.just("second")),
                // combinator
                Tuples.fn2());
```  


#### Mono.zipWhen
`Mono.zipWhen` 은 입력 값으로 받은 `Mono upstream` 소스가 방출되는 시점에 결합 할 수 있다. 
`Mono upstream` 소스가 방출될 때, 방출된 값을 메소드의 인자로 전달해 수행 한 뒤 그 결과와 함께 결합하는 등의 방식으로 활용 할 수 있다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zipWhen-java.util.function.Function-
 */
@Test
public void mono_zipWhen() {
    Mono.just("first")
            .zipWhen(s -> Mono.just(s + ":second"))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", "first:second"))
            .verifyComplete();
}
```  

`first` 를 방출하는 `Mono` 가 방출 될때 `zipWhen` 이 수행되고, 
`zipWhen` 에서는 방출된 값이 `first` 를 바탕으로 `first:second` 라는 결과를 만들어 낸다. 
그러면 최종적으로 `Tuple` 에 `first` 와 `zipWhen` 의 결과인 `first:second` 가 담기게 된다.  


#### Mono.zipWith
`Mono.zipWith` 는 `Mono upstream` 이 방출 될때 추가적인 `Mono` 와 결합 할 수 있다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zipWith-reactor.core.publisher.Mono-
 */
@Test
public void mono_zipWith() {
    Mono.just("first")
            .zipWith(Mono.just("second"))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", "second"))
            .verifyComplete();
}
```  

`first` 를 방출하는 `Mono` 가 결과를 방출 하고 나서, 
`zipWith` 에서 `second` 를 방출하는 `Mono` 와 결합한다. 
최종적으로 `Tuple` 에는 `first` 와 `second` 가 담기게 된다.  


#### Mono.zipDelayError
`Mono.zipDelayError` 는 결합할 `Mono upstream` 소스들이 오류를 반환 하더라도 결과를 바로 반환하지 않고, 
모든 소스가 완료 될떄까지 결과 방출을 보류한다.  

먼저 일반적인 `Mono.zip` 에서 오류를 방출되면 어떻게 동작하는지 확인하면, 
측정 소스 하나가 오류를 방출하자 마자 다른 소스의 결과는 기다리지 않고 에러 결과를 방출 한다.  

```java
@Test
public void mono_zip_error() {
    Mono.zip(Mono.just("first"),
                    Mono.error(new RuntimeException("exception1")),
                    Mono.error(new RuntimeException("exception2")))
            .as(StepVerifier::create)
            .verifyErrorMatches(throwable -> throwable.getMessage().equals("exception1"));
}
```  

`Mono.zipDelayError` 는 위 `Mono.zip` 에서 에러가 발생 했을 때와는 다르게, 
모든 소스에서 결과를 방출 할떄까지 기다리므로 여러 소스에서 에러가 발생했음을 추가로 확인 할 수 있다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#zipDelayError-java.util.function.Function-reactor.core.publisher.Mono...-
 */
@Test
public void mono_zipDelayError() {
    Mono.zipDelayError(Mono.just("first"),
                    Mono.error(new RuntimeException("exception1")),
                    Mono.error(new RuntimeException("exception2")))
            .as(StepVerifier::create)
            .verifyErrorMatches(throwable -> throwable.getMessage().equals("Multiple exceptions") &&
                    throwable.getSuppressed()[0].getMessage().equals("exception1") &&
                    throwable.getSuppressed()[1].getMessage().equals("exception2"));
}
```  

`Multiple exception` 이라는 메시지와 함께 어떤 에러들이 방생했는지 추가로 확인 가능하다.  

#### Mono.zip fallback 및 기본값 처리
`Reactive Stream` 에서 일반적인 기본값과 예외처리를 살펴보면, 
특정 값을 반환하도록 작성해주는 경우도 있지만, `Mono.empty()` 를 방출하는 경우가 많다. 
`Mono.zip` 에서 특정 소스가 `Mono.empty()` 를 반환하면 
아래와 같이 다른 소스가 결과를 방출 하더라도 `Mono.empty()` 를 결과로 방출 한다.  

```java
@Test
public void mono_zip_empty() {
    Mono.zip(Mono.just("first"), Mono.empty())
            .as(StepVerifier::create)
            .verifyComplete();
}
```  

그러므로 `zip` 에 포함된 어느 하나의 소스라도 값을 방출 할떄, 
결과를 만들어 내야 한다면 아래와 같이 `Optional.empty()` 를 사용해서 `Mono.empty()` 인 경우에 기본값 설정을 해줄 수 있다.  

```java
@Test
public void mono_zip_empty_properly() {
    Mono.zip(Mono.just("first"), 
                    Mono.empty().switchIfEmpty(Mono.just(Optional.empty())))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", Optional.empty()))
            .verifyComplete();
}
```  

그리고 앞서 살펴 본것 처럼 `Mono.zip` 에서 하나의 소스라도 에러를 방출 한다면, 
`zip` 의 결과 또한 에러이므로 위 `Optional.empty()` 를 활용한 `Mono.zip` 기본값 처리를 사용해 `fallback` 처리를 하면 아래와 같다.  

```java
@Test
public void zip_error_fallback() {
    Mono.zip(Mono.just("first"),
                    Mono.error(new RuntimeException("exception1")).onErrorReturn(Optional.empty()),
                    Mono.error(new RuntimeException("exception2")).onErrorReturn(Optional.empty()))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("first", Optional.empty(), Optional.empty()))
            .verifyComplete();
}
```  

`Optional.empty()` 를 통한 기본 값 처리가 중요한게 아니라, 
에러 발생 상황에라도 어떠한 결과를 내려줘야 한다면, 
`zip` 에 포함된 소스들에 `onErrorReturn` 과 같은 예외처리를 꼭 필요하다.  
