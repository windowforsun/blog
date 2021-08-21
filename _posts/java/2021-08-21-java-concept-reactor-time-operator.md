--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Time Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 시간 처리관련 Operator 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Working with Time
toc: true 
use_math: true
---  

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Working with Time(시간 작업)

구분|메소드|타입|설명
---|---|---|---
측정 시간 함께 방출|timed|Mono,Flux|시퀀스에서 요소를 방출할때 측정시간과 함께 방출
방출 지연 타임아웃 설정|timeout|Mono,Flux|방출 지연시간 제한을 설정 및 타임아웃시 `Fallback` 처리
일정한 간격으로 시간 값 방출|interval|Flux|정해진 지연시간마다 `Long` 값을 0부터 차례대로 방출
방출 지연시간 추가|delayElements|Mono,Flux|주어진 시퀀스의 요소 방출 지연시간 설정
 |delaySubscription|Mono,Flux|주어진 시퀀스에 `subscribe()` 이벤트까지 지연시간 설정

### timed()
`timed()` 메소드는 시퀀스에서 요소를 방출할때 소요시간 등을 측정할 수 있는 `Operator` 이다. 
`Timed` 라는 객체를 사용하는데 아래와 같은 소요시간 관련 값을 사용할 수 있다. 
- `Timed.elapsed()` : `onNext()` 이벤트 사이의 소요시간에 대한 값이다. 
  첫 `onNext()` 이벤트의 경우 `onSubscribe()` 부터 `onNext()` 까지의 소요시간이고, 
  이후 부터는 직전 `onNext()` 부터 현재 `onNext()` 까지의 소요시간이다. 
- `Timed.timestamp()` : `onNext()` 이벤트가 발생한 시간에 대한 값이다. 
- `Timed.elapsedSinceSubscription()` : `onSubscribe()` 이벤트 부터 현재 `onNext()` 이벤트까지의 소요시간이다. 

```java
@Test
public void flux_timed() throws Exception {
	// given
	Flux<Integer> source_1 = Flux.create(integerFluxSink -> {
		for (int i = 1; i <= 3; i++) {
			try {
				TimeUnit.MILLISECONDS.sleep(i * 100L);
				integerFluxSink.next(i);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}

		integerFluxSink.complete();
	});
	Flux<Timed<Integer>> source = source_1.timed();

	StepVerifier
			.create(source.log())
			.expectSubscription()
			.consumeNextWith(integerTimed -> {
				assertThat(integerTimed.get(), is(1));
				assertThat(integerTimed.elapsed().toMillis(), allOf(greaterThan(100L), lessThan(150L)));
				assertThat(integerTimed.elapsedSinceSubscription().toMillis(), allOf(greaterThan(100L), lessThan(180L)));
			})
			.consumeNextWith(integerTimed -> {
				assertThat(integerTimed.get(), is(2));
				assertThat(integerTimed.elapsed().toMillis(), allOf(greaterThan(200L), lessThan(250L)));
				assertThat(integerTimed.elapsedSinceSubscription().toMillis(), allOf(greaterThan(300L), lessThan(380L)));
			})
			.consumeNextWith(integerTimed -> {
				assertThat(integerTimed.get(), is(3));
				assertThat(integerTimed.elapsed().toMillis(), allOf(greaterThan(300L), lessThan(350L)));
				assertThat(integerTimed.elapsedSinceSubscription().toMillis(), allOf(greaterThan(600L), lessThan(680L)));
			})
			.thenCancel()
			.verify();
}
```  

### timeout()
`timeout()` 은 시퀀스에서 아이템을 방출할때 최대 지연시간을 설정할 수 있고, 
최대 지연시간을 초과해서 방출하게 되는 경우 `TimeoutException` 을 발생시키거나, 
다른 시퀀스를 방출 하도록 `Fallback` 처리를 할 수 있다.  

```java
@Test
public void flux_timeout() {
	// given
	Flux<Integer> source_1 = Flux.create(integerFluxSink -> {
		for (int i = 1; i <= 3; i++) {
			try {
				TimeUnit.MILLISECONDS.sleep(i * 100L);
				integerFluxSink.next(i);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}

		integerFluxSink.complete();
	});
	Flux<Integer> source = source_1.timeout(Duration.ofMillis(250));

	StepVerifier
			.create(source)
			.expectSubscription()
			.expectNext(1)
			.expectNext(2)
			.expectError(TimeoutException.class)
			.verify();
}

@Test
public void flux_timeout_fallback() {
	// given
	Flux<Integer> source_1 = Flux.create(integerFluxSink -> {
		for (int i = 1; i <= 3; i++) {
			try {
				TimeUnit.MILLISECONDS.sleep(i * 100L);
				integerFluxSink.next(i);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}

		integerFluxSink.complete();
	});
	Flux<Integer> fallback = Flux.just(11, 22, 33);
	Flux<Integer> source = source_1.timeout(Duration.ofMillis(250), fallback);

	StepVerifier
			.create(source)
			.expectSubscription()
			.expectNext(1)
			.expectNext(2)
			.expectNext(11, 22, 33)
			.verifyComplete();
}
```  


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


