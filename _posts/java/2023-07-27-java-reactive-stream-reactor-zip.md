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


### Flux.zip
`Flux Stream` 을 결합하는 여러 `Flux.zip` 관련 메소드에 대해 알아본다.

#### Flux.zip tuple
`Flux` 의 `zip` 연산중 가장 간단한 형태를 지닌 메소드로, 
`Flux.zip` 은 N개 이상의 `Flux upstream` 소스를 결합해 결과를 `Tuple` 로 반환한다. 
입력한 `Flus upstream` 소스의 수 만큼의 요소를 지니는 `Tuple` 을 생성한다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#zip-org.reactivestreams.Publisher-org.reactivestreams.Publisher-
 */
@Test
public void flux_zip_tuple() {
    Flux.zip(Flux.fromIterable(List.of(1, 2, 3, 4)), Flux.fromIterable(List.of("a", "b", "c")))
            .as(StepVerifier::create)
            .recordWith(ArrayList::new)
            .thenConsumeWhile(tuple2s -> true)
            .consumeRecordedWith(tuple2s -> {
                assertThat(tuple2s, hasSize(3));
                assertThat(tuple2s, contains(Tuples.of(1, "a"), Tuples.of(2, "b"), Tuples.of(3, "c")));
            })
            .verifyComplete();
}
```  

`1, 2, 3, 4` 를 방출하는 `Flux` 스트림과 `a, b, c,` 를 방출하는 `Flux` 스트림을 결합하면, 
최종적으로 `(1, a), (2, b), (3, c)` 와 같은 총 3개의 `Tuple` 이 생성된다. 
가장 적은 데이터수를 가지는 `Flux upstream` 이 기준이 되기 때문에 `4` 는 결과로 반환되지 않는다.  


#### Flux.zip combinator
N개 이상의 `Flux upstream` 을 소스로 가지는 `zip` 연산의 결과를 원하는 결과타입을 생성해서 반환 할 수 있다. 
원하는 결과 타입을 생성하기 위해서 `combinator` 를 알 맞게 정의해줘야 한다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#zip-java.lang.Iterable-java.util.function.Function-
 */
@Test
public void flux_zip_combinator() {
    Flux.zip(List.of(Flux.fromIterable(List.of(1, 2, 3)),
                    Flux.fromIterable(List.of("a", "b", "c", "d")),
                    Flux.fromIterable(List.of("first", "second", "third"))),
                    objects -> Map.of("key1", objects[0], "key2", objects[1], "key3", objects[2]))
            .as(StepVerifier::create)
            .expectNext(Map.of("key1", 1, "key2", "a", "key3", "first"))
            .expectNext(Map.of("key1", 2, "key2", "b", "key3", "second"))
            .expectNext(Map.of("key1", 3, "key2", "c", "key3", "third"))
            .verifyComplete();
}
```  

`1, 2, 3` 을 방출하는 `Flux` 와 `a, b, c, d` 를 방출하는 `Flux`, `first, second, third` 를 방출하는 `Flux` 를 
`combinator` 에서 `Map` 형식으로 결합해 결과를 반환한다. 
그러면 최종적으로 `{key1 : 1, key2 : a, key3 : first}, {key1 : 2, key2 : b, key3 : second}, {key1 : 3, key2 : c, key3 : third}` 
와 같은 3개의 `Map` 이 결과로 방출 된다.  


#### Flux.zipWith
`Flux.zipWith` 는 `Flux upstream` 소스와의 결합을 수행한다. 
`Flux upstream` 소스와 자신의 `Flux stream` 을 결합한다는 점을 기억해야 하고, 
그러므로 `combinator` 를 따로 정의하지 않았다면 결과 타입은 2개의 요소를 갖는 `Tuple` 이다.  

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#zipWith-org.reactivestreams.Publisher-
 */
@Test
public void flux_zipWith() {
    Flux.fromIterable(List.of(1, 2, 3, 4))
            .zipWith(Flux.fromIterable(List.of("a", "b", "c", "d")))
            .zipWith(Flux.fromIterable(List.of("first", "second", "third")))
            .as(StepVerifier::create)
            .expectNext(Tuples.of(Tuples.of(1, "a"), "first"))
            .expectNext(Tuples.of(Tuples.of(2, "b"), "second"))
            .expectNext(Tuples.of(Tuples.of(3, "c"), "third"))
            .verifyComplete();
}
```  

`1, 2, 3, 4` 를 방출하는 `Flux` 와 `a, b, c, d` 를 방출하는 `Flux` 를 결합해서, 
`(1, a), (2, b), (3, c), (4, d)` 와 같은 4개의 `Tuple` 을 결과를 생성한다. 
그리고 `first, second, third` 를 방출하는 `Flux` 와 결합해서 
최종적으로 `((1, a), first), ((2, b), second), ((3, c), third)` 처럼 3개의 `Tuple` 이 중첩된 형태로 결과가 생성된다.  


#### Flux.zipWithIterable
`Flux.zipWithIterable` 는 바로 앞에서 알아본 `Flux.zipWith` 와 `Flux upstream` 과 결합한다는 점에서는 매우 비슷하다.
다른 점은 인수로 `Flux Stream` 이 아닌 방출 할 데이터를 `Iterable` 타입으로 전달해주면 된다는 점에서 차이가 있다.   

```java
/**
 * https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#zipWithIterable-java.lang.Iterable-
 */
@Test
public void flux_zipWithIterable() {
    Flux.fromIterable(List.of(1, 2, 3, 4))
            .zipWithIterable(List.of("a", "b", "c", "d"), List::of)
            .zipWithIterable(List.of("first", "second", "third"), (serializables, s) -> Stream.concat(serializables.stream(), Stream.of(s)).collect(Collectors.toList()))
            .as(StepVerifier::create)
            .expectNext()
            .expectNext(List.of(1, "a", "first"))
            .expectNext(List.of(2, "b", "second"))
            .expectNext(List.of(3, "c", "third"))
            .verifyComplete();

}
```  

`1, 2, 3, 4` 를 방출하는 `Flux` 와 `a, b, c, d` 를 방출하는 `Flux` 를 결합해서, 
`[1, a], [2, b], [3, c], [4, d]` 와 같은 2개의 원소를 갖는 4개 리스트를 결과로 생성한다. 
그리고 `first, second, thrid` 를 방출하는 `Flux` 와 결합해서 
최종적으로 `[1, a, first], [2, b, second], [3, c, third]` 와 같은 3개의 원소를 갖는 3개 리스트의 결과가 생성된다.  


#### Flux.zip fallback 및 기본값 처리
`Flux` 에서 예외상황등에 기본값 처리르 `Flux.empty` 를 사용할 수 있다. 
이처럼 `Flux.zip` 에 포함된 `Flux stream` 중 하나가 `Flux.empty` 를 방출할 떄 상황을 살펴 보자.  

```java
@Test
public void flux_zip_empty() {
    Flux.zip(Flux.fromIterable(List.of(1, 2, 3, 4)),
            Flux.concat(Mono.just("first"), Mono.empty()),
            Flux.fromIterable(List.of("a", "b", "c")))
            .as(StepVerifier::create)
            .expectNext(Tuples.of(1, "first", "a"))
            .verifyComplete();
}
```  

`Flux.zip` 소스 중 다른 소스들은 모두 정상적으로 결과를 방출 하지만, 
하나의 소스가 `empty` 를 방출하자 전체 스트림이 중단되었다. 
만약 처음 부터 `empty` 를 방출한다면 어떤 결과도 방출되지 않을 것이다.  

만약 위 처럼 `empty` 가 방출될 수 있는 상황에서 `zip` 을 구성하는 스트림 중 
정상적인 결과를 방출한 스트림이 있을 때도 이어 결과를 방출하고 싶다면 적절한 기본 값 처리가 필요하다. 
다양한 기본값 처리 방법이 있을 수 있겠지만, `Mono.zip` 에서 사용한 방법과 동일하게 
`Optional.empty()` 를 사용한 방법에 대해 살펴보자.  

```java
@Test
public void flux_zip_empty_properly() {
    Flux.zip(Flux.fromIterable(List.of(1, 2, 3, 4)),
                    Flux.empty().switchIfEmpty(Mono.just(Optional.empty()).repeat()),
                    Flux.fromIterable(List.of("a", "b", "c")))
            .as(StepVerifier::create)
            .expectNext(Tuples.of(1, Optional.empty(), "a"))
            .expectNext(Tuples.of(2, Optional.empty(), "b"))
            .expectNext(Tuples.of(3, Optional.empty(), "c"))
            .verifyComplete();
}
```  

`zip` 을 구성하는 `Flux stream` 중 `empty` 가 방출되는 상황을 재현한 코드를 구성했다. 
중요한 부분은 `swithIfEmpty` 를 통해 `Flux stream` 이 `empty` 를 방출 할 떄의 예외처리가 수행 될 수 있도록 했고, 
수행되는 예외처리는 `Mono.empty()` 를 계속해서 방출하는 스트림을 생성했다. 
위와 같은 처리를 구현한 이유는 `zip` 은 앞서 설명한 것처럼 가장 짧은 방출 데이터의 길이를 갖는 스트림을 기준으로 하기 때문에, 
`zip` 을 구성하는 `Flux stream` 중 어느 하나라도 `Complete` 시그널을 보낸다면 전체 스트림은 완료 된다는 성질을 이용한 것이다.  

`Flux.zip` 을 수행하던 과정에 예외가 발생한 상황을 가정해 본다. 
아래 처럼 특정 방출 시점에 예외가 발생한다면, 
`Mono.zip` 과는 다르게 `Flux.zip` 은 예외 자체를 결과 그대로 방출한다.  

```java
@Test
public void flux_zip_error() {
    RuntimeException testException = new RuntimeException("test exception");
    Flux.zip(Flux.fromIterable(List.of(1, 2, 3, 4, 5)),
                    Flux.fromIterable(List.of("a", testException, "c", testException)),
                    Flux.fromIterable(List.of(testException, "second", testException, "fourth")))
            .as(StepVerifier::create)
            .expectNext(Tuples.of(1, "a", testException))
            .expectNext(Tuples.of(2, testException, "second"))
            .expectNext(Tuples.of(3, "c", testException))
            .expectNext(Tuples.of(4, testException, "fourth"))
            .verifyComplete();
}
```  

만약 예외 발생시 기본값 처리와 같은 예외처리가 필요하다면, 
`onErrorResume`, `onErrorReturn` 등과 같은 `Operation` 을 사용해서 구현해 볼 수 있다. 
예외를 `Optional.empty()` 로 치환이 필요하다면 아래와 같이 `combinator` 에서 수행 해줄 수 있다.  

```java
@Test
public void flux_zip_error_fallback() {
    RuntimeException testException = new RuntimeException("test exception");
    Flux.zip(List.of(Flux.fromIterable(List.of(1, 2, 3, 4, 5)),
                    Flux.fromIterable(List.of("a", testException, "c", testException)),
                    Flux.fromIterable(List.of(testException, "second", testException, "fourth"))),
                    objects -> Tuples.fn3().apply(Arrays.stream(objects).map(o -> o instanceof Throwable ? Optional.empty() : o).toArray()))
            .as(StepVerifier::create)
            .expectNext(Tuples.of(1, "a", Optional.empty()))
            .expectNext(Tuples.of(2, Optional.empty(), "second"))
            .expectNext(Tuples.of(3, "c", Optional.empty()))
            .expectNext(Tuples.of(4, Optional.empty(), "fourth"))
            .verifyComplete();
}
```  


### Execution time of Zip
앞서 간략하게 언급 했지만, 
`Zip` 연산이 결과를 방출하기 까지 소요되는 시간은 구성하는 스트림 중 가장 느린 스트림을 기준으로 한다. 
그 예시에 해당하는 코드는 아래와 같다. 방출까지 1s, 100ms, 2s, 50ms 가 소요되는 4개의 `Mono` 를 `zip` 연산을 수행해본다. 


```java
@Test
public void zip_when_delay() {
    Mono<String> mono1s = Mono.fromCallable(() -> {
                Thread.sleep(1000);
                return "1s";
            })
            .doOnNext(s -> log.info("publish : {}", s));
    Mono<String> mono100ms = Mono.fromCallable(() -> {
                Thread.sleep(100);
                return "100ms";
            })
            .doOnNext(s -> log.info("publish : {}", s));
    Mono<String> mono2s = Mono.fromCallable(() -> {
                Thread.sleep(2000);
                return "2s";
            })
            .doOnNext(s -> log.info("publish : {}", s));
    Mono<String> mono50ms = Mono.fromCallable(() -> {
                Thread.sleep(50);
                return "50ms";
            })
            .doOnNext(s -> log.info("publish : {}", s));

    Mono.zip(mono1s, mono100ms, mono2s, mono50ms)
            .doOnNext(s -> log.info("{}", s))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("1s", "100ms", "2s", "50ms"))
            .expectComplete()
            .verifyThenAssertThat()
            .tookMoreThan(Duration.ofMillis(3000))
            .tookLessThan(Duration.ofMillis(3200));
}
```

위 코드가 수행되며 발생한 로그는 아래와 같다.  

```
19:59:43.897 [main] INFO com.windowforsun.reactor.zip.ZipTest - publish : 1s
19:59:44.002 [main] INFO com.windowforsun.reactor.zip.ZipTest - publish : 100ms
19:59:46.003 [main] INFO com.windowforsun.reactor.zip.ZipTest - publish : 2s
19:59:46.053 [main] INFO com.windowforsun.reactor.zip.ZipTest - publish : 50ms
19:59:46.053 [main] INFO com.windowforsun.reactor.zip.ZipTest - zip : [1s,100ms,2s,50ms]
```  

`Zip` 을 구성하는 4개의 스트림의 실제 실행 순서는 방출하고 스트림이 종료되는 것은 `Zip` 의 인자 순서대로 진행되었다. 
하지만 최종적으로 `Zip` 의 결과는 방출까지 가장 소요시간이 오래걸리는 `2s` 스트림이 완료 될때까지 다른 결과는 지연되는 것으 확인 할 수 있다. 
당연한 결과라고 할 수도 있지만 `Zip` 을 사용해서 결과를 만들어 낼때 하나의 스트림이라도 크게 지연된다면 전체 결과가 지연될 수 있으므로, 
이 부분은 주의해서 사용이 필요하다. 
최종적으로 `Zip` 의 결과는 4개의 모든 `Mono` 가 방출할 떄까지 소요된 시간의 합인 3.2초 정도가 소요된 것을 확인 할 수 있다. 
이는 `Zip` 연상을 수행하는 스레드와 동일한 스레드에서 모든 `Reactive Stream` 이 수행되기 때문에 발생한 결과이다. 

`Zip` 연산을 수행하는 스레드(`main`, ..)와는 별개로 스트림을 수행하는 병렬 처리 방식으로 `Schedulers` 를 사용해서, 
`Zip` 결과에 소요되는 시간을 `Zip` 을 구성하는 스트림 중 가장 오래걸리는 스트림의 시간까지 단축 시킬 수 있다.  

```java
@Test
public void zip_when_delay_4() {
    Mono<String> mono1s = Mono.fromCallable(() -> {
                Thread.sleep(1000);
                return "1s";
            })
            .doOnNext(s -> log.info("publish : {}", s))
            .subscribeOn(Schedulers.boundedElastic());
    Mono<String> mono100ms = Mono.fromCallable(() -> {
                Thread.sleep(100);
                return "100ms";
            })
            .doOnNext(s -> log.info("publish : {}", s))
            .subscribeOn(Schedulers.boundedElastic());
    Mono<String> mono2s = Mono.fromCallable(() -> {
                Thread.sleep(2000);
                return "2s";
            })
            .doOnNext(s -> log.info("publish : {}", s))
            .subscribeOn(Schedulers.boundedElastic());
    Mono<String> mono50ms = Mono.fromCallable(() -> {
                Thread.sleep(50);
                return "50ms";
            })
            .doOnNext(s -> log.info("publish : {}", s))
            .subscribeOn(Schedulers.boundedElastic());

    Mono.zip(mono1s, mono100ms, mono2s, mono50ms)
            .doOnNext(s -> log.info("zip : {}", s))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("1s", "100ms", "2s", "50ms"))
            .expectComplete()
            .verifyThenAssertThat()
            .tookMoreThan(Duration.ofMillis(2000))
            .tookLessThan(Duration.ofMillis(2100));
}
```  

```
20:09:58.297 [boundedElastic-4] INFO com.windowforsun.reactor.zip.ZipTest - publish : 50ms
20:09:58.347 [boundedElastic-2] INFO com.windowforsun.reactor.zip.ZipTest - publish : 100ms
20:09:59.247 [boundedElastic-1] INFO com.windowforsun.reactor.zip.ZipTest - publish : 1s
20:10:00.247 [boundedElastic-3] INFO com.windowforsun.reactor.zip.ZipTest - publish : 2s
20:10:00.247 [boundedElastic-3] INFO com.windowforsun.reactor.zip.ZipTest - zip : [1s,100ms,2s,50ms]
```

첫 번째 예제와 다른 점은 `Zip` 에 사용되는 2개의 `Mono` 에 `Schedulers` 를 통해 각 스트림이 병렬로 수행 될 수 있도록 설정 했다. 
로그는 `Zip` 인자의 순서대로가 아닌 소요시간이 짧은 것부터 차례로 방출되고,
결과 또한 방출까지 소요되는 최대 소요시간인 2초를 기준으로 `Mono.zip` 의 연산도 마무리 되는 것을 확인 할 수 있다.  


---
## Reference
[API Reference Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)  
[API Reference Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)  


