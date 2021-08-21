--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Filtering Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 시퀀스를 필터링하는 Operator 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Filtering a Sequence
toc: true 
use_math: true
---  

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Filtering a Sequence(필터링)

구분|메소드|타입|설명
---|---|---|---
시퀀스 필터링|filter|Mono,Flux|임의의 기준으로 필터링
 |filterWhen|Mono,Flux|임의의 기준으로 필터링(비동기)
 |ofType|Mono,Flux|타입을 기준으로 필터링
 |ignoreElements|Mono,Flux|모든 값을 무시
 |distinct|Flux|전체 시퀀스에 대해서 중복 값 무시
 |distinctUntilChanged|Flux|연달아 방출되는 중복 값 무시(연이은 중복 값 중 첫 번째 값 유지)
시퀀스에서 일부 값 만 유지|take(long)|Flux|시퀀스 앞에서 부터 `N`개 값 방출
 |take(Duration)|Flux|시퀀스 앞에서 부터 특정 시간 동안 방출되는 값
 |takeLast|Flux|시퀀스 마지막 부터 `N`개 값 방출
 |takeUntil(Predicate)|Flux|시퀀스 앞에서 부터 기준을 충족 할때 까지 방출
 |takeUntilOther(Publisher)|Flux|`Publisher` 방출 시그널이 수행된 시점 까지만 방출
 |takeWhile|Flux|시퀀스 앞에서 부터 기준을 만족하는 동안 방출
 |elementAt|Flux|시퀀스 앞에서 부터(0) `index`에 해당하는 값 방출
 |last|Flux|시퀀스의 가장 마지막 값 방출(존재하지 않으면 에러)
 |last(T)|Flux|시퀀스의 가장 마지막 값 방출(존재하지 않으면 기본 값 `T`)
 |skip(long)|Flux|시퀀스 앞에서 부터 `N`개를 건너뛰고 방출
 |skip(Duration)|Flux|시퀀스 앞에서 부터 특정 시간동안 방출되는 요소는 건너뛰고 방출
 |skipLast|Flux|시퀀스 뒤에서 부터 `N`개는 방출하지 않음 
 |skipUntil(Predicate)|Flux|조건을 충족 할떄까지 건너뛰고 그 이후부터 방출
 |skipUntilOther(Publisher)|Flux|`Publisher` 방출 시그널이 수행될 떄까지 건너뛰고 그 이후부터 방출
 |skipWhile|Flux|조건을 만족하는 동안 건너뛰고 이후부터 방출
 |sample(Duration)|Flux|주어진 주기에 해당하는 요소만 방출
 |sample(Publisher)|Flux|`Publisher` 에서 발생하는 시그널 직전에 방출된 요소만 방출
 |sampleTimeout(Function<T, Publisher>)|Flux|기존 시퀀스에서 요소가 방출될 때마다 수행되는 타임아웃 `Publisher` 시그널과 요소 방출 시그널이 겹치지 않으면 방출
요소가 최대 1개|single|Flux|비어있거나 2개이상은 에러
 |single(T)|Flux|2개이상은 에러, 비어있는 경우 기본 값 방출
 |singleOrEmpty|Flux|2개이상은 에러, 빈 시퀀스 허용


### distinct(), distinctUntilChanged() 

```java
@Test
public void flux_distinct() {
	// given
	Flux<String> source_1 = Flux.just("a", "b", "a", "b", "c", "c");
	Flux<String> source = source_1.distinct();

	StepVerifier
			.create(source)
			.expectNext("a")
			.expectNext("b")
			.expectNext("c")
			.verifyComplete();
}

@Test
public void flux_distinctUntilChanged() {
	// given
	Flux<String> source_1 = Flux.just("a", "b", "a", "b", "c", "c");
	Flux<String> source = source_1.distinctUntilChanged();

	StepVerifier
			.create(source)
			.expectNext("a", "b")
			.expectNext("a", "b")
			.expectNext("c")
			.verifyComplete();
}
```  

### takeUntil(), takeUntilOther(), skipUntil(), skipUntilOther()

```java
@Test
public void flux_takeUntil() {
	// given
	Flux<String> source_1 = Flux.just("filter-a", "filter-b", "c", "d");
	Flux<String> source = source_1.takeUntil(s -> !s.startsWith("filter"));

	StepVerifier
			.create(source)
			.expectNext("filter-a")
			.expectNext("filter-b")
			.expectNext("c")
			.verifyComplete();
}

@Test
public void flux_takeUntilOther() {
	// given
	Flux<String> source_1 = Flux
			.just("filter-a", "filter-b", "c", "d")
			.delayElements(Duration.ofMillis(100));
	Flux<String> source = source_1
			.takeUntilOther(
					Mono.just("").delayElement(Duration.ofMillis(250))
			);

	StepVerifier
			.create(source)
			.expectNext("filter-a")
			.expectNext("filter-b")
			.verifyComplete();
}

@Test
public void flux_skipUntil() {
	// given
	Flux<String> source_1 = Flux.just("filter-a", "filter-b", "c", "d");
	Flux<String> source = source_1.skipUntil(s -> !s.startsWith("filter"));

	StepVerifier
			.create(source)
			.expectNext("c")
			.expectNext("d")
			.verifyComplete();
}

@Test
public void flux_skipUntilOther() {
	// given
	Flux<String> source_1 = Flux
			.just("filter-a", "filter-b", "c", "d")
			.delayElements(Duration.ofMillis(100));
	Flux<String> source = source_1
			.skipUntilOther(
					Mono.just("").delayElement(Duration.ofMillis(250))
			);

	StepVerifier
			.create(source)
			.expectNext("c")
			.expectNext("d")
			.verifyComplete();
}
```  

### sample(), sampleTimeout()
`sample` 이름이 들어간 메소드는 모두 시퀀스에서 방출되는 요소를 특정 조건을 바탕으로 샘플링하는 역할을 수행한다. 
- `sample(Publisher)` : `Publisher` 요소 방출 직전에 기존 시퀀스에서 방출된 요소만 방출
- `sampleTimeout(Function<T, Publisher>)` : 기존 시퀀스 요소 방출이 `Publisher`요소 방출 주기 보다 큰 경우만 방출된다. 
`Publisher` 요소 방출 주기보다 기존 시퀀스 방출 주기가 적다면 타임아웃으로 방출되지 않음(마지막 요소는 항상 방출)

```java
@Test
public void flux_sample_pub() {
	// given
	Flux<String> source_1 = Flux
			.just("a", "b", "c", "d")
			.delayElements(Duration.ofMillis(100));
	Flux<String> source = source_1
			.sample(
					Flux.just("", "").delayElements(Duration.ofMillis(250))
			);

	StepVerifier
			.create(source)
			.expectNext("b")
			.expectNext("d")
			.verifyComplete();
}

@Test
public void flux_sampleTimeout_onlyLast() throws Exception {
	// given
	Flux<String> source_1 = Flux
			.just("a", "b", "c", "d")
			.delayElements(Duration.ofMillis(50));
	Flux<String> source = source_1
			.sampleTimeout(s -> Mono.just("").delayElement(Duration.ofMillis(100)));

	StepVerifier
			.create(source)
			.expectNext("d")
			.verifyComplete();
}

@Test
public void flux_sampleTimeout_all() throws Exception {
	// given
	Flux<String> source_1 = Flux
			.just("a", "b", "c", "d")
			.delayElements(Duration.ofMillis(50));
	Flux<String> source = source_1
			.sampleTimeout(s -> Mono.just("").delayElement(Duration.ofMillis(10)));

	StepVerifier
			.create(source)
			.expectNext("a")
			.expectNext("b")
			.expectNext("c")
			.expectNext("d")
			.verifyComplete();
}
```  


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


