--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Mono, Flux"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Reactor 의 Mono 와 Flux에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - ReactiveStreams
  - Reactor
  - Mono
  - Flux
toc: true
use_math: true
---  

## Reactor Mono, Flux
[Reactive Streams](({{site.baseurl}}{% link _posts/java/2021-04-18-java-concept-reactive-before-and-after.md %}))
에서는 `Reactive Streams` 의 개념에 대해서 알아보았고,
[Reactive Streams 활용](({{site.baseurl}}{% link _posts/java/2021-04-24-java-concept-ractivestreams-advanced.md %}))
에서는 `Reactive Streams` 를 활용하는 방법에 대해서 알아보았다. 
[Reactor 3 Reference Guide - Reactor Core Features](https://projectreactor.io/docs/core/release/reference/index.html#core-features)
를 보면 `Reactive Streams` 를 효과적으로 활용할 수 있는 `Publisher` 의 구현체인 `Mono` 와 `Flux` 가 있다. 
- `Mono` : 단일 값 또는 빈값(0 ~ 1) 을 나타낸다. 
- `Flux` : `0 ~ N` 개의 요소가 있는 리엑티브 시퀀스를 나타낸다. 

`Reactive Streams` 에서는 `Mono` 와 `Flux` 가 담는 요소의 수를 `카디널리티` 라고 칭한다. 
그리고 스트림이라는 용어는 `Java 8` 의 스트림과 혼동의 여지가 있으므로 `Reactive Streams` 자체를 표현할 때가 아니면, 
스트림이라는 표현보다는 시퀀스(`Sequence`)라는 용어를 사용한다.  

`Mono`, `Flux` 에서는 `CorePublisher<T>` 라는 인터페이스가 사용되는데 

### Flux
`Flux<T>` 는 `0 ~ N` 개의 아이템을 생산하는 비동기 시퀀스를 나타내는 `Publisher<T>` 의 구현체이다.  

```java
public abstract class Flux<T> implements CorePublisher<T> {
    // ...
}
```  

`Flux` 아이템의 흐름을 도식화하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-1.svg)

`Flux` 는 `Publisher` 와 동일하게 몇가지 메소드(이벤트)를 통해 구독자(`Subscriber`)가 발생시키면 그에 따라 동작을 수행한다. 
위 그림은 `1, 2 3` 데이터는 `onNext()` 를 통해 데이터가 전달된 것이고 `4` 는 전달과정에서 에러가 발생해서 `onError()` 이벤트가 발생했다. 
그리고 만약 `Flux` 의 모든 아이템이 전달된 경우에는 `onComplete()` 이벤트를 통해 시퀀스는 종료된다.  

`Flux` 를 사용할 때 모든 이벤트를 전달해서 사용할 필요는 없다. 
만약 `onNext()` 이벤트 없이 `onComplete()` 이벤트만 발생한다면 비어있는 시퀀스가 되고, 
`onComplete()` 이벤트 없이 `onNext()` 이벤트만 계속 수행된다면 무한 시퀀스가 될 수 있다.  

### Mono
`Mono<T>` 는 `0 ~ 1` 개의 아이템을 생산하는 비동기 시퀀스를 나타내는 `Publisher<T>` 의 구현체이다.  

```java
public abstract class Mono<T> implements CorePublisher<T> {
    // ...
}
```  

`Mono<T>` 는 최대 1개의 아이템 생산에 적합한 `Publisher<T>` 의 구현체로, 
`onNext()` 이벤트 호출 후 `onComplete()` 이벤트 호추롤 시퀀스가 종료되거나, 
`onError()` 이벤트로 시퀀스가 종료되는 경우가 있다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-1.svg)

만약 별개의 `Mono<T>` 두개가 있을 때 이를 하나의 `Flux<T>` 로 합치는 등의 연산도 가능하다. 
그리고 `Mono` 는 아이템은 존재하지 않고 시퀀스의 완료만 존재하는 구성으로도 사용할 수 있다. 
이는 `Runnable` 과 유사한 하나의 비동기 처리 표현으로 `Mono<Void>` 와 같이 만들 수 있다.  


### 의존성 추가하기
`Mono`, `Flux` 를 통한 `Reactive Streams` 를 사용하기 위해서는 의존성 추가가 필요하다. 
만약 스프링을 사용하지 않는 다면 아래와 같이 `reactor-core` 를 의존성을 추가해서 `Java` 환경에서
`Mono`, `Flux` 를 사용한 애플리케이션을 개발할 수 있다. 

```groovy
dependencies {
	implementation group: 'io.projectreactor', name: 'reactor-core', version: '3.4.5'
}
```  

그리고 `Spring Boot` 환경에서는 위와 같이 `reactor-core` 의존성을 추가해도 되지만, 
`WebFlux` 의존성을 추가한다면 `Spring` 환경에서 `Mono`, `Flux` 를 사용한 웹 애플리케이션 개발이 가능하다.  

```groovy
dependencies {
	implementationb 'org.springframework.boot:spring-boot-starter-webflux:2.4.2'
}
```  

### Mono, Flux 시퀀스 생성하기
`Mono`, `Flux` 시퀀스를 생성하는 가장 간단한 방법은 각 클래스에서 제공하는 팩토리 메소드를 사용하는 것이다. 
아주 많은 방식으로 시퀀스를 생성할 수 있는 메소드를 제공하지만, `just()` 메소드에 대해서만 다뤄본다. 

#### just()

`Mono`, `Flux` 에서 공통으로 제공하는 `just()` 메소드는 파라미터로 받은 값을 사용해서 시퀀스를 생성해주는 팩토리 메소드이다. 
먼저 `Mono` 에서의 동작 흐름은 아래와 같이 한개의 인자를 사용해서 사용할 수 있다.

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-just.svg)  

```java
// long 형 아이템 1을 생성하는 Mono 시퀀스 생성
Mono<Long> mono = Mono.just(1L);

```  

그리고 `Flux` 에서는 아래와 같이 한개 이상의 파라미터를 사용해서 사용할 수 있다. 

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-just.svg)

```java
// long 형 아이템 1, 2, 3을 순서대로 생성하는 Flux 시퀀스 생성
Flux<Long> flux = Flux.just(1L, 2L, 3L);
```  

`Mono` 에는 `justOrEmpty()` 라는 메소드도 있는데 비어있는 `Mono` 시퀀스를 생성할 때 사용할 수 있다. 
인자로 받을 수 있는 값은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-justorempty.svg)

```java
Mono<Long> mono = Mono.justOrEmpty(1L);

Mono<Long> emptyMonoNull = Mono.justOrEmpty(null);

Mono<Long> emptyMonoOptional = Mono.justOrEmpty(Optional.empty());

Mono<Long> mono = Mono.justOrEmpty(1L);

Mono<Long> monoOptional = Mono.justOrEmpty(Optional.of(1L));
```  

### 구독 메소드 
`Mono`, `Flux` 의 시퀀스를 생성했다면 이를 구독해야 시퀀스의 데이터를 받아 처리할 수 있다. 
`Mono` 와 `Flux` 에서는 아주 다양한 구독 관련 팩토리 메소드를 지원하지만, 
이번 포트스테서는 구독관련 메소드 또한 간단한 몇가지에 대해서만 다뤄본다. 

#### Mono block()
`Mono` 의 `block()` 은 메소드 이름에서 알 수 있는 것처럼 시퀀스가 끝나거나 아이템하나를 받을 때까지 무한정 `Blocking` 방식으로 데이터를 받는 것을 의미한다. 
테스트용이나 정말 해당 메소드가 필요한 경우가 아니라면 사용을 하지 않는 것이 좋을 것같다. 
`Reactive Streams` 라는 `Non-Blocking` 흐름에서 하나의 `block()` 메소드로 인해 `Non-Blocking` 의 흐름이 깨져버릴 수 있기 때문이다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-block.svg)

위 그림처럼 `Mono` 시퀀스에 아이템이 존재하는 경우 해당 아이템을 `Blocking` 방식으로 리턴하고, 
비어있는 시퀀스는 `null` 을 리턴하고 발생된 예외는 그대로 전파된다.
아래는 간단한 `block()` 메소드의 예시이다. 

```java
@Test
public void mono_block_item() {
	// given
	Mono<Long> mono = Mono.just(1L);

	// when
	Long actual = mono.block();

	// then
	assertThat(actual, notNullValue());
	assertThat(actual, is(1L));
}

@Test
public void mono_block_empty() {
	// given
	Mono<Long> mono = Mono.empty();

	// when
	Long actual = mono.block();

	// then
	assertThat(actual, nullValue());
}

@Test
public void mono_block_exception() {
	// given
	Mono<Long> mono = Mono.error(new Exception("mono block exception"));

	// when
	Exception actual = assertThrows(Exception.class, () -> mono.block());

	// then
	assertThat(actual, notNullValue());
	assertThat(actual.getMessage(), is("java.lang.Exception: mono block exception"));
}
```  

#### Flux blockFirst(), blockLast()
`Flux` 에는 `blockFirst()` 와 `blockLast()` 메소드를 사용해서 시퀀스내 아이템을 `Blocking` 방식으로 리턴 받을 수 있다. 
이 또한 모드 시퀀스가 종료되거나 아이템 하나를 받을 때까지 `Blocking` 되기 때문에 사용에는 주의가 필요하다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-blockfirst.svg)

`blockFirst()` 는 말그대로 `Flux` 시퀀스에서 가장 첫번째 아이템을 리턴하는 메소드이다. 

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-blocklast.svg)

그리고 `blockLast()` 는 마지막 아이템을 리턴한다. 
다른 특징은 `Mono` 의 `block()` 과 동일하게 비어있는 시퀀스는 `null` 을 리턴하고 발생된 예외는 그대로 전파된다. 
아래는 `blockFirst()` 와 `blockLast()` 의 간단한 예시이다. 

```java
@Test
public void flux_blockFirst_item() {
	// given
	Flux<Long> flux = Flux.just(1L, 2L, 3L);

	// when
	Long actual_1 = flux.blockFirst();
	Long actual_2 = flux.blockFirst();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_1, is(1L));
	assertThat(actual_2, notNullValue());
	assertThat(actual_2, is(1L));
}

@Test
public void flux_blockLast_item() {
	// given
	Flux<Long> flux = Flux.just(1L, 2L, 3L);

	// when
	Long actual_1 = flux.blockLast();
	Long actual_2 = flux.blockLast();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_1, is(3L));
	assertThat(actual_2, notNullValue());
	assertThat(actual_2, is(3L));
}

@Test
public void flux_blockFirst_empty() {
	// given
	Flux<Long> flux = Flux.empty();

	// when
	Long actual = flux.blockFirst();

	// then
	assertThat(actual, nullValue());
}

@Test
public void flux_blockFirst_error() {
	// given
	Flux<Long> flux = Flux.error(new Exception("flux block exception"));

	// when
	Exception actual = assertThrows(Exception.class, () -> flux.blockFirst());

	// then
	assertThat(actual, notNullValue());
	assertThat(actual.getMessage(), is("java.lang.Exception: flux block exception"));
}
```  

`Flux` 의 `blockFirst()`, `blockLast()` 메소드를 보면 계속 동일한 첫번재 아이템과 마지막 아이템을 리턴하는 것을 확인 할 수 있다.  


#### log()
`Mono`, `Flux` 에 모두 존재하는 `log()` 는 시퀀스내에서 이뤄지는 동작을 로그 출력을 통해 확인할 수 있는 구독 메소드이다. 
`Reactive Streams` 에서 이뤄지는 모든 시그널을 `INFO`(`default`) 레벨 로그로 관찰 할수 있어 디버깅과 같은 용도로 사용하기에 좋다.  

아래는 `Mono` 의 `log()` 의 동작을 도식화 한 그림이다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-log.svg)  

아래는 `Flux` 의 `log()` 의 동작을 도식화 한 그림이다. 

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-log.svg)  

`Mono`, `Flux` 모두 동작과 메소드 구성은 동일하다. 
`log()` 전용 카테고리도 추가할 수 있고, 필요에 따라 로그 레벨을 수정하는 등 다양한 설정으로 모니터링을 구성할 수 있다.  

```java
@Test
public void mono_log() {
	// given
	Mono<Long> mono = Mono.just(1L);

	// when
	Long actual = mono.log().block();

	// then
	assertThat(actual, notNullValue());
	assertThat(actual, is(1L));
}

/*
[main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO reactor.Mono.Just.1 - | request(unbounded)
[main] INFO reactor.Mono.Just.1 - | onNext(1)
[main] INFO reactor.Mono.Just.1 - | onComplete()
 */

@Test
public void mono_log_category() {
	// given
	Mono<Long> mono = Mono.just(1L);

	// when
	Long actual = mono.log("myCategory.").block();

	// then
	assertThat(actual, notNullValue());
	assertThat(actual, is(1L));
}

/*
[main] INFO myCategory.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO myCategory.Mono.Just.1 - | request(unbounded)
[main] INFO myCategory.Mono.Just.1 - | onNext(1)
[main] INFO myCategory.Mono.Just.1 - | onComplete()
 */

@Test
public void mono_log_category_level() {
	// given
	Mono<Long> mono = Mono.just(1L);

	// when
	Long actual = mono.log("myCategory.", Level.WARNING).block();

	// then
	assertThat(actual, notNullValue());
	assertThat(actual, is(1L));
}

/*
[main] WARN myCategory.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] WARN myCategory.Mono.Just.1 - | request(unbounded)
[main] WARN myCategory.Mono.Just.1 - | onNext(1)
[main] WARN myCategory.Mono.Just.1 - | onComplete()
 */

@Test
public void flux_log() {
	// given
	Flux<Long> flux = Flux.just(1L, 2L, 3L);

	// when
	Long actual_1 = flux.log().blockFirst();
	Long actual_2 = flux.log().blockFirst();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_1, is(1L));
	assertThat(actual_2, notNullValue());
	assertThat(actual_2, is(1L));
}

/*
[main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO reactor.Flux.Array.1 - | request(unbounded)
[main] INFO reactor.Flux.Array.1 - | onNext(1)
[main] INFO reactor.Flux.Array.1 - | cancel()
[main] INFO reactor.Flux.Array.2 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO reactor.Flux.Array.2 - | request(unbounded)
[main] INFO reactor.Flux.Array.2 - | onNext(1)
[main] INFO reactor.Flux.Array.2 - | cancel()
 */
```  

`log()` 가 출력하는 로그를 보면 `block()` 메소드의 동작을 위해 내부적으로 수행되는 시그널들을 확인할 수 있다. 
`onSubscribe()` 로 시퀀스를 구독하고, `request()` 로 아이템을 요청한 후, `onNext()` 로 아이템을 받으면,
`Mono` 의 경우 `onComplete()` 로 시퀀스 구독을 완료하고, 
`Flux` 는 `cancel()` 로 구독을 취소하는 것을 확인 할 수 있다.  

`log()` 를 통해 `Flux` 의 `blockFirst()` 를 2번 호출 했을 때 동일한 첫번째 아이템을 리턴하는 이유를 시그널을 통해 확인 할 수 있다. 
현재 `blockFirst()` 는 시퀀스를 구독하고 1개를 요청한 후 구독을 취소하는 동작만을 반복하기 때문에 계속해서 첫번째 아이템만 리턴하게 된다.


#### subscribe()
`subscribe()` 는 시퀀스내에 존재하는 모든 아이템을 소비하는 구독메소드이다. 
주로 시퀀스 아이템에 대한 처리가 구현된 `Consumer<? super T>` 객체를 파라미터로 전달해서 수행된다. 
그리고 시퀀스내에서 발생할 수 있는 예외는 `Consumer<? super Throwable>` 구현체를 사용해서 처리할 수 있다. 
그 외에도 다양한 목적에서 사용할 수 있도록 오버로딩 돼있다. 
이번 포스트에서는 그중 기본적인 몇가지에 종류에 대해서만 알아본다.  

먼저 인자값으 받지 않는 `Mono`, `Flux` 의 `subscribe()` 는 아래와 같다.

```java
@Test
public void mono_subscribe() {
	// given
	Mono<Long> mono = Mono.just(1L);

	// when
	mono.log().subscribe();
}

/*
[main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO reactor.Mono.Just.1 - | request(unbounded)
[main] INFO reactor.Mono.Just.1 - | onNext(1)
[main] INFO reactor.Mono.Just.1 - | onComplete()
 */


@Test
public void flux_subscribe() {
	// given
	Flux<Long> flux = Flux.just(1L, 2L, 3L);
	
	// when
	flux.log().subscribe();
}

/*
[main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO reactor.Flux.Array.1 - | request(unbounded)
[main] INFO reactor.Flux.Array.1 - | onNext(1)
[main] INFO reactor.Flux.Array.1 - | onNext(2)
[main] INFO reactor.Flux.Array.1 - | onNext(3)
[main] INFO reactor.Flux.Array.1 - | onComplete()
 */
```  

`Mono`, `Flux` 시퀀스에 존재하는 모든 아이템을 구독해서 소비하는 것을 로그로 확인할 수 있다.  

다음으로는 시퀀스에서 제공하는 아이템에 대한 처리를 `Consumer` 구현 객체로 처리할 수 있는 `subscribe()` 는 아래와 같다.  

아래는 `Mono` 의 `subscribe(Consumer<? super T> consumer)`  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-subscribe-consumer.svg)

아래는 `Flux` 의 `subscribe(Consumer<? super T> consumer)`  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-subscribe-consumer.svg)


```java
@Test
public void mono_subscribe_consumer() {
	// given
	Mono<Long> mono = Mono.just(1L);
	List<Long> actual = new ArrayList<>();

	// when
	mono.log().subscribe(aLong -> actual.add(aLong));

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0), is(1L));
}

/*
[main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO reactor.Mono.Just.1 - | request(unbounded)
[main] INFO reactor.Mono.Just.1 - | onNext(1)
[main] INFO reactor.Mono.Just.1 - | onComplete()
 */

@Test
public void flux_subscribe_consumer() {
	// given
	Flux<Long> flux = Flux.just(1L, 2L, 3L);
	List<Long> actual = new ArrayList<>();

	// when
	flux.log().subscribe(aLong -> actual.add(aLong));

	// then
	assertThat(actual, hasSize(3));
	assertThat(actual.get(0), is(1L));
	assertThat(actual.get(1), is(2L));
	assertThat(actual.get(2), is(3L));
}

/*
[main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO reactor.Flux.Array.1 - | request(unbounded)
[main] INFO reactor.Flux.Array.1 - | onNext(1)
[main] INFO reactor.Flux.Array.1 - | onNext(2)
[main] INFO reactor.Flux.Array.1 - | onNext(3)
[main] INFO reactor.Flux.Array.1 - | onComplete()
 */
```  

마지막으로 시퀀스에서 발생하는 예외 처리에 대한 `Consumer` 까지 인자값으로 받는 `subscribe()` 는 아래와 같다.  

아래는 `Mono` 의 `subscribe(Consumer<? super T> consumer, Consumer<? super Throwable> errorConsumer)`  

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-mono-subscribe-consumer-error.svg)

아래는 `Flux` 의 `subscribe(Consumer<? super T> consumer, Consumer<? super Throwable> errorConsumer)`

![그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-flux-subscribe-consumer-error.svg)

```java
@Test
public void mono_subscribe_consumer_error() {
	// given
	Mono<Long> mono = Mono.error(new Exception("mono error"));
	List<Long> actual = new ArrayList<>();
	List<Throwable> actualException = new ArrayList<>();

	// when
	mono.log().subscribe(
			aLong -> actual.add(aLong),
			throwable -> actualException.add(throwable)
	);

	// then
	assertThat(actual, hasSize(0));
	assertThat(actualException, hasSize(1));
	assertThat(actualException.get(0).getMessage(), is("mono error"));
}

/*
[main] INFO reactor.Mono.Error.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Mono.Error.1 - request(unbounded)
[main] ERROR reactor.Mono.Error.1 - onError(java.lang.Exception: mono error)
[main] ERROR reactor.Mono.Error.1 - 
java.lang.Exception: mono error
 */

@Test
public void flux_subscribe_consumer_error() {
	// given
	Flux<Long> flux = Flux.error(new Exception("flux error"));
	List<Long> actual = new ArrayList<>();
	List<Throwable> actualException = new ArrayList<>();

	// when
	flux.log().subscribe(
	aLong -> actual.add(aLong),
	throwable -> actualException.add(throwable)
	);

	// then
	assertThat(actual, hasSize(0));
	assertThat(actualException, hasSize(1));
	assertThat(actualException.get(0).getMessage(), is("flux error"));
}

/*
[main] INFO reactor.Flux.Error.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Flux.Error.1 - request(unbounded)
[main] ERROR reactor.Flux.Error.1 - onError(java.lang.Exception: flux error)
[main] ERROR reactor.Flux.Error.1 - 
java.lang.Exception: flux error
 */
```  


---
## Reference
[4. Reactor Core Features](https://projectreactor.io/docs/core/release/reference/#core-features)  
[Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)  
[Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)  
[Intro To Reactor Core Baeldung](https://www.baeldung.com/reactor-core)  

