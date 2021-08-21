--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Create Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 시퀀스를 생성하는 Operator 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Create a New Sequence
toc: true 
use_math: true
---  

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Creating a New Sequence(시퀀스 생성)

메소드|타입|설명
---|---|---
just|Mono, Flux|인자값을 시퀀스 아이템으로 생성
justOrEmpty|Mono, Flux|`just` 동작에서 `null` 값 포함
defer|Mono,Flux|지연처리로 시퀀스 생성
empty|Mono,Flux|바로 완료되는 시퀀스 생성
error|Mono,Flux|바로 실패하는 시퀀스 생성
never|Mono,Flux|어떤 시그널도 발생하지 않는 시퀀스 생성(무한 시퀀스)
using|Mono,Flux|일회용 리소스를 바탕으로 시퀀스 생성
create|Mono,Flux|비동기 프로그래밍 방식으로 시퀀스 생성
fromArray|Flux|배열로 시퀀스 생성
fromIterable|Flux|`Iterable`, `Collection` 로 시퀀스 생성
fromStream|Flux|`Stream` 으로 시퀀스 생성
range|Flux|범위의 정수 값으로 시퀀스 생성
generate|Flux|동기식 프로그래밍 방식으로 시퀀스 생성
fromSupplier|Mono|지연처리로 시퀀스 생성
fromRunnable|Mono|`Runnable` 객체로 시퀀스 생성(결과값 X)
fromFuture|Mono|`CompletableFuture` 객체로 시퀀스 생성(결과값 O)

### defer(), fromSupplier()
`defer(Publisher)`, `fromSupplier(Supplier)` 를 사용하면 시퀀스 생성을 지연처리 할 수 있다. 
여기서 지연처리하는 것은 `Cold Sequence` 의 특징을 갖는다.  

```java
@Test
public void mono_defer() throws Exception {
	// given
	Mono<Long> source = Mono.defer(() -> Mono.just(System.currentTimeMillis()));
	long current = System.currentTimeMillis();
	Thread.sleep(100);

	// when
	long actualFirst = source.block();
	Thread.sleep(100);
	long actualSecond = source.block();

	// then
	assertThat(actualFirst, greaterThan(current));
	assertThat(actualSecond, allOf(
			greaterThan(current),
			greaterThan(actualFirst)
	));
}

@Test
public void flux_defer() throws Exception {
	// given
	Flux<Long> source = Flux.defer(() -> Flux
			.just(System.currentTimeMillis(), System.currentTimeMillis())
			.delayElements(Duration.ofMillis(10)));
	long current = System.currentTimeMillis();
	Thread.sleep(100);

	// when
	List<Long> actualFirst = source.collectList().block();
	Thread.sleep(100);
	List<Long> actualSecond = source.collectList().block();

	// then
	assertThat(actualFirst, everyItem(greaterThan(current)));
	assertThat(actualSecond, allOf(
			everyItem(greaterThan(current)),
			not(actualFirst)
	));
}

@Test
public void mono_fromSupplier() throws Exception {
	// given
	Mono<Long> source = Mono.fromSupplier(() -> System.currentTimeMillis());
	long current = System.currentTimeMillis();
	Thread.sleep(100);

	// when
	long actualFirst = source.block();
	Thread.sleep(100);
	long actualSecond = source.block();

	// then
	assertThat(actualFirst, greaterThan(current));
	assertThat(actualSecond, allOf(
			greaterThan(current),
			greaterThan(actualFirst)
	));
}
```  

`defer()`, `fromSupplier()` 의 테스트를 보면 실제 구독이 이뤄지는 시점에 시퀀스 생성이 실제로 이뤄지고, 
구독을 할때마다 새로운 시퀀스를 생성하는 것을 확인 할 수 있다.  

### create(), generate()
`create()` 와 `generate()` 를 사용하면 프로그래밍 방식으로 직접 시퀀스의 이벤트를 조작하며 시퀀스를 생성할 수 있다.  

```java
@Test
public void flux_generate() {
	// given
	Flux<String> source = Flux.generate(
			() -> "",
			(s, synchronousSink) -> {
				if (s.length() < 3) {
					s = s + s.length();
					synchronousSink.next(s);
				} else {
					synchronousSink.complete();
				}

				return s;
			}
	);

	StepVerifier
			.create(source)
			.expectNext("0", "01", "012")
			.verifyComplete();
}

@Test
public void flux_generate_error() {
	// given
	Flux<String> source = Flux.generate(
			() -> "",
			(s, synchronousSink) -> {
				if (s.length() < 3) {
					s = s + s.length();
					synchronousSink.next(s);
				} else if (s.length() == 3) {
					synchronousSink.error(new Exception("my exception"));
				} else {
					synchronousSink.complete();
				}

				return s;
			}
	);

	StepVerifier
			.create(source)
			.expectNext("0", "01", "012")
			.expectErrorMessage("my exception")
			.verify();
}

@Test
public void flux_create() {
	// given
	Flux<String> source = Flux.create(stringFluxSink -> {
		String s = "";

		while (true) {
			if (s.length() < 3) {
				s = s + s.length();
				stringFluxSink.next(s);
			} else {
				stringFluxSink.complete();
				break;
			}
		}
	});

	StepVerifier
			.create(source)
			.expectNext("0", "01", "012")
			.verifyComplete();
}

@Test
public void flux_create_error() {
	// given
	Flux<String> source = Flux.create(stringFluxSink -> {
		String s = "";

		while (true) {
			if (s.length() < 3) {
				s = s + s.length();
				stringFluxSink.next(s);
			} else if (s.length() == 3) {
				stringFluxSink.error(new Exception("my exception"));
				break;
			} else {
				stringFluxSink.complete();
				break;
			}
		}
	});

	StepVerifier
			.create(source)
			.expectNext("0", "01", "012")
			.expectErrorMessage("my exception")
			.verify();
}
```

---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


