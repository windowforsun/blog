--- 
layout: single
classes: wide
title: "[Java 개념] "
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
    - Asynchronous
	- ReactiveStreams
toc: true
use_math: true
---  

## Reactive Streams Core Features
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
스트림이라는 표현보다는 시퀀스라는 용어를 사용한다.  

`Mono`, `Flux` 에서는 `CorePublisher<T>` 라는 인터페이스가 사용되는데 

### Flux
`Flux<T>` 는 `0 ~ N` 개의 아이템을 생산하는 비동기 시퀀스를 나타내는 `Publisher<T>` 의 구현체이다.  

```java
public abstract class Flux<T> implements CorePublisher<T> {
    // ...
}
```  

`Flux` 아이템의 흐름을 도식화하면 아래와 같다.  

[그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-1.png)

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

만약 별개의 `Mono<T>` 두개가 있을 때 이를 하나의 `Flux<T>` 로 합치는 등의 연산도 가능하다. 
그리고 `Mono` 는 아이템은 존재하지 않고 시퀀스의 완료만 존재하는 구성으로도 사용할 수 있다. 
이는 `Runnable` 과 유사한 하나의 비동기 처리 표현으로 `Mono<Void>` 와 같이 만들 수 있다.  









### Hot, Cold Publisher
hot, cold publisher 먼저 정리하고 나서
다시 아래 mono 만들기 다시 정리하기 ..


















### Mono 만들기
`Mono` 는 `Publisher` 의 구현체인 만큼 데이터를 만들어내는 생산자의 역할을 한다.
`Mono` 를 만드는 방법 중 가장 간단한 방법은 각 클래스내에 있는 팩토리 메소드를 사용하는 것이다. 
먼저 최대 1개 아이템을 생산하는 `Mono` 를 생성하는 방법에 대해서 알아본다.  

실제 테스트코드를 살펴보기 전에 이후 진행되는 설명은 예제 코드를 바탕으로 진행된다. 
예제 코드에서 `Mono` 와 `Flux` 의 구독해서 사용하는 아래 2가지 `Subscirber` 를 사용한다.  

먼저 `log()` 는 아래의 사진과 같이 `Reactive Streams` 의 시퀀스에서 `Publisher`, `Subscirber` 간에 주고받는 신호를 모니터링 가능하도록 로그로 보여주는 `Subscriber` 이다.

[그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-log.svg)

다음으로 `block()` 은 아래 사진과 같이 `Mono` 시퀀스에 존재하는 아이템을 `Blocking` 방식으로 모두 가져오는 `Subscriber` 이다.

[그림 1]({{site.baseurl}}/img/java/concept-reactive-streams-mono-flux-block.svg)


#### empty, never, error

```java
@Test
public void mono_create_empty() {
	// given
	Mono<String> mono = Mono.empty();

	// when
	String actual = mono.log().block();

	// then
	assertThat(actual, nullValue());
}
```  

```
[main] INFO reactor.Mono.Empty.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Mono.Empty.1 - request(unbounded)
[main] INFO reactor.Mono.Empty.1 - onComplete()	
```  
`empty()` 는 비어있는(0개 아이템) `Mono` 시퀀스를 생성하는 메소드로 `block()` 메소드로 아이템을 리턴받아보면 `null` 임을 확인할 수 있다. 
그리고 비어 있기 때문에 `reqeust()` 로 아이템을 요청하는 경우 바로 `onComplete()` 이 호출 된다.  

```java
@Test
public void mono_create_never() {
	// given
	Mono<Long> mono = Mono.never();

	// when
	IllegalStateException actual = assertThrows(IllegalStateException.class, () -> mono.log().block(Duration.ofMillis(100L)));

	// then
	assertThat(actual.getMessage(), startsWith("Timeout on blocking read"));
}
```  

```
[main] INFO reactor.Mono.Never.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Mono.Never.1 - request(unbounded)
[main] INFO reactor.Mono.Never.1 - cancel()
```  
그리고 `never()` 는 시퀀스의 종료를 알리는 이벤트를 호출하지 않는 시퀀스를 생성하는 메소드이다.
종료 이벤트인 `onError()`, `onComplete()` 가 호출되지 않기 때문에 무한한 시퀀스를 생성한다. 
만약 이때 `block()` 을 호출하면 무한정 시퀀스를 대기하게 되기 때문에, 
테스트 코드에서는 `block(Duration)` 을 사용해서 무한정 시퀀스를 대기하지 않고 `IllegalStateException` 과 함께 종료 될 수 있도록 진행했다.  
출력된 로그를 보면 `cancel()` 호출을 통해 무한 시퀀스를 강제로 종료한 것을 확인 할 수 있다.  

```java
@Test
public void mono_create_error() {
	// given
	Mono<Long> mono = Mono.error(new Exception("mono error"));

	// when
	Exception actual = assertThrows(Exception.class, () -> mono.log().block());

	// then
	assertThat(actual.getMessage(), is("java.lang.Exception: mono error"));
}
```  

```
[main] INFO reactor.Mono.Error.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Mono.Error.1 - request(unbounded)
[main] ERROR reactor.Mono.Error.1 - onError(java.lang.Exception: mono error)
[main] ERROR reactor.Mono.Error.1 - 
java.lang.Exception: mono error	
```  

마지막으로 `error()` 는 `request()` 를 통해 아이템을 요청하는 즉시 시퀀스 생성시에 정의한 에러를 바탕으로 `onError()` 이벤트를 전송하는 시퀀스를 생성한다. 
출력된 로그를 보면 `request()` 이후 곧 바로 `onError()` 가 호출되었고, 전달된 예외의 메시지는 `mono error` 인 것을 확인 할 수 있다.  


### just

```java
@Test
public void mono_create_just() throws Exception {
	// given
	Mono<Long> mono = Mono.just(System.currentTimeMillis());
	Thread.sleep(100);
	long current = System.currentTimeMillis();

	// when
	Thread.sleep(100);
	long actual = mono.log().block();

	// then
	assertThat(actual, lessThan(current));
}
```  

```
[main] INFO reactor.Mono.Just.1 - | onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO reactor.Mono.Just.1 - | request(unbounded)
[main] INFO reactor.Mono.Just.1 - | onNext(1623581924528)
[main] INFO reactor.Mono.Just.1 - | onComplete()
```  

`just()` 는 파라미터로 전달된 값을 아이템으로 사용하는 `Mono` 시퀀스를 즉시 생성한다. 
여기서 즉시 생성한다는 것은 `just()` 메소드가 호출된 시점에 시퀀스가 바로 생성된다는 것을 의미한다. 
`just()` 의 선언 즉시 시퀀스가 생성되는 동작은 이후 `fromXXX` 메소드의 동작과 비교 가능하므로 기억해 두는 것이 좋다. 
위와 관련된 동작 테스트를 위해 `just()` 를 생성할때 현재 시간의 타임스템프를 가져오는 메소드를 사용했고, 
이후 100밀리초 이후 `current` 라는 변수에 현재 시간의 타임스템프를 할당했다. 
그리고 `block()` 메소드를 사용해서 `Mono` 시퀀스의 아이템을 가져와 비교하면 `current` 보다 작은 타임스템프를 갖는 것을 확인 할 수 있고, 
이는 `current` 에 타임스템프가 할당되기 전에 이미 `Mono` 시퀀스에 아이템이 생성되었다는 것을 의미한다.  


```java
@Test
public void mono_create_justOrEmpty() {
	// given
	Mono<String> mono = Mono.justOrEmpty(Optional.empty());
//    Mono<String> mono = Mono.justOrEmpty(null);

	// when
	String actual = mono.log().block();

	// then
	assertThat(actual, nullValue());
}
```  

```
[main] INFO reactor.Mono.Empty.1 - onSubscribe([Fuseable] Operators.EmptySubscription)
[main] INFO reactor.Mono.Empty.1 - request(unbounded)
[main] INFO reactor.Mono.Empty.1 - onComplete()
```  

`justOrEmpty()` 는 `just()` 와 대부분 비슷하고 차이점으로는 `null` 값이나 ,
`Otpional.empty()` 와 같은 빈값을 아이템으로 사용해서 `Mono` 를 생성할수 있다는 점이다. 
실제로 `just()` 에 `null` 값을 파라미터로 사용하면 `NullPointException` 이 발생한다.  


### from

```java
@Test
public void mono_create_fromCallable() throws Exception {
	// given
	Mono<Long> mono = Mono.fromCallable(() -> System.currentTimeMillis());
	Thread.sleep(100);
	long current = System.currentTimeMillis();

	// when
	Thread.sleep(100);
	long actual = mono.log().block();

	// then
	assertThat(actual, greaterThan(current));
}
```  

```
[main] INFO reactor.Mono.Callable.1 - | onSubscribe([Fuseable] Operators.MonoSubscriber)
[main] INFO reactor.Mono.Callable.1 - | request(unbounded)
[main] INFO reactor.Mono.Callable.1 - | onNext(1623607371982)
[main] INFO reactor.Mono.Callable.1 - | onComplete()
```  

`fromCallable()` 은 아이템을 생성할 때 `Callable` 의 `call()` 메소드가 리턴하는 값으로 `Mono` 를 생성하는 메소드이다. 
`just` 와 달리 `fromCallable()` 선언 시점에 `Mono` 시퀀스를 생성하지 않고, `Mono` 시퀀스를 사용하는 구독 시점에 시퀀스가 실제로 생성된다. (`Lazy` 처리)
이런 특징으로 테스트 코드를 확인하면 `Mono` 시퀀스에서 생성한 타임스탬프(`actual`)가 `current` 보다 값이 큰 것을 확인 할 수 있고, 
이는 `Mono` 시퀀스의 생성이 `current` 타임스탬프보다 이후에 수행되었다는 것을 알 수 있다.  

```java
@Test
public void mono_create_fromRunnable() throws Exception {
	// given
	Mono<Void> mono = Mono.fromRunnable(() -> log.info("fromRunnable: {}", System.currentTimeMillis()));
	Thread.sleep(100);
	log.info("testMethod: {}", System.currentTimeMillis());

	// when
	Thread.sleep(100);
	Void actual = mono.log().block();

	// then
	// console print log
	assertThat(actual, nullValue());
}
```  

```
[main] INFO com.windowforsun.reactor.CreateMonoTest - testMethod: 1623607950685
[main] INFO reactor.Mono.Runnable.1 - onSubscribe(MonoRunnable.MonoRunnableEagerSubscription)
[main] INFO reactor.Mono.Runnable.1 - request(unbounded)
[main] INFO com.windowforsun.reactor.CreateMonoTest - fromRunnable: 1623607950799
[main] INFO reactor.Mono.Runnable.1 - onComplete()
```  

`fromRunnable()` 은 `Mono` 시퀀스 상에 아이템을 생성 및 제공하지는 않고, 
`Runnable` 의 `run()` 메소드에 정의된 동작을 수행하는 메소드이다. 
아이템은 생성하지 않기 때문에 `null` 을 리턴한다. 
그리고 `fromCallable()` 과 동일하게 실제 동작 수행 시점이 `Mono` 시퀀스 선어 시점이 아닌, 
구독 이후에 `Runnable` 에 정의된 동작이 수행되는 것을 확인 할 수 있다.  

```java
@Test
public void mono_create_fromSupplier() throws Exception {
	// given
	Mono<Long> mono = Mono.fromSupplier(() -> System.currentTimeMillis());
	Thread.sleep(100);
	long current = System.currentTimeMillis();
	
	// when
	Thread.sleep(100);
	long actual = mono.log().block();

	// then
	assertThat(actual, greaterThan(current));
}
```  

```
[main] INFO reactor.Mono.Supplier.1 - | onSubscribe([Fuseable] Operators.MonoSubscriber)
[main] INFO reactor.Mono.Supplier.1 - | request(unbounded)
[main] INFO reactor.Mono.Supplier.1 - | onNext(1623608196332)
[main] INFO reactor.Mono.Supplier.1 - | onComplete()
```  

`fromSuplier()` 는 `Java Streams` 에서 `Functional Interface` 중 하나인 `Suplier` 를 바탕으로 `Mono` 시퀀스를 생성하는 메소드이다. 
이 메소드 또한 `Mono` 시퀀스 선언 시점이 아닌, 구독이 된 시점에 실제 시퀀스 생성이 이뤄지기 때문에 아이템의 값(`actual`) 이 `current` 보다 큰 것을 확인 할 수 있다.  

```java
@Test
public void mono_create_defer() throws Exception {
	// given
	Mono<Long> mono = Mono.defer(() -> Mono.just(System.currentTimeMillis()));
	Thread.sleep(100);
	long current = System.currentTimeMillis();

	// when
	Thread.sleep(100);
	long actual = mono.log().block();

	// then
	assertThat(actual, greaterThan(current));
}
```  

```
[main] INFO reactor.Mono.Defer.1 - onSubscribe([Synchronous Fuseable] Operators.ScalarSubscription)
[main] INFO reactor.Mono.Defer.1 - request(unbounded)
[main] INFO reactor.Mono.Defer.1 - onNext(1623608704763)
[main] INFO reactor.Mono.Defer.1 - onComplete()
```  

`fromDefer` 




### Mono, Flux 만들기










### Mono, Flux 구독하고 사용하기




### 생산자와 소비자
전에 `Reactive Streams` 에서 언급했던 것과 같이 `Reactive Streams` 는 생산자와 소비자의 관계로 구성된다. 
생산자가 아무리 좋고 많은 아이템을 가지고 있더라도 소비자가 이를 받아 사용하지 않으면 아이템들은 의미가 없고 존재하지 않는 것과 마찬가지이다.  

위 예시와 같이 `Mono`, `Flux` 도 마찬가지이다. 
시퀀스를 만든다는 것은 시퀀스에 정의된 동작까지 수행되는 것을 의미하지 않는다. 
`Subscriber` 까지 등록해 줘야 시퀀스 동작의 실행까지 이뤄진다.  

시퀀스를 생성할 수 있는 많은 팩토리 메소드 중 `just()` 와 시퀀스를 구독해서 아이템을 받을 수 있는 `subscribe()` 메소드를 
사용해서 구성한 테스트 코드는 아래와 같다.  

```java
@Test
public void subscribe() throws Exception {
	// given
	Mono<Long> mono = Mono.just(System.currentTimeMillis());
	Thread.sleep(100);
	long created = System.currentTimeMillis();

	// when
	mono.subscribe(item -> {
		// then
		assertThat(item, lessThan(created));
	});
}
```

테스트 코드에서 `Mono` 의 값으로 현재시간의 타임스탬프 값을 설정한다. 
테스트 결과를 보면 `created` 의 값이 `Mono` 의 `item` 값보다 100 밀리초 정도 더 큰 것을 확인 가능하다. 
`Mono` 의 선언은 `created` 보다 100 밀리초 먼저 선언되었지만, 
구독은 100 밀리초 후에 수행되었고 이때 선언한 `Mono` 에 등록된 수행되어서 위와 같은 결과가 나온 것이다.  

### Flux 비동기 시퀀스 생성하기


### Mono 비동기 시퀀스 생성하기


### 구독하기
- 취소

### 구독 구현하기
BaseSubscriber


### 코드로 시퀀스 생성하기



---
## Reference
[4. Reactor Core Features](https://projectreactor.io/docs/core/release/reference/#core-features)  
[Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)  
[Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)  
[Intro To Reactor Core | Baeldung](https://www.baeldung.com/reactor-core)  

