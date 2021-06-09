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

