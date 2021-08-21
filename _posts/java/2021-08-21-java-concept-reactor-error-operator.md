--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Error Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 에러를 처리하는 Operator에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Handling Errors
toc: true 
use_math: true
---  

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Handling Errors(에러 처리)

구분|메소드|타입|설명
---|---|---|---
시퀀스 에러 발생|error|Mono,Flux|시퀀스가 방출 될때 에러가 시그널 발생(에러를 던지는 개념)
 |error(Supplier<Throwable>)|Mono,Flux|`error` 동작에서 `Lazy` 처리
try/catch 처리|onErrorReturn|Mono,Flux|에러 시그널 발생시 주어진 기본값을 방출
 |onErrorResume|Mono,Flux|에러 시그널 발생시 주어진 시퀀스를 이어서 방출
 |onErrorMap|Mono,Flux|에러 시그널 발생시 주어진 예외로 에러 방출
 |doFinally|Mono,Flux|시퀀스 도중 에러가 발생하든 안하든 시퀀스가 종료되고 항상 수행되는 동작
 |using|Mono,Flux|시퀀스 내에서 특정 자원을 설정하고 값을 방출하고 자원에 대한 후처리 수행
에러 복구|onErrorReturn|Mono,Flux|에러 시그널 발생시 주어진 기본값을 방출
 |onErrorResume|Mono,Flux|에러 시그널 발생시 주어진 시퀀스를 이어서 방출
 |retry|Mono,Flux|에러 시그널 발생시 시퀀스 구독부터 다시 수행
 |retryWhen|Mono,Flux|[Retry](https://projectreactor.io/docs/core/3.4.6/api/reactor/util/retry/Retry.html) 클래스에서 제공하는 기능과 조건을 바탕으로 재시도 수행
`backpressure` 에러<br/>(업스트림에서 요청한 만큼 다운스트림에서 생산하지 못할때)|onBackpressureError|Flux|`IllegalStateException` 에러 방출
 |onBackpressureDrop|Flux|`Backpressure` 에러에 해당하는 요소는 모두 버림(`Drop` 처리)
 |onBackpressureLatest|Flux|`Backpressure` 에러가 발생한 요소중 가장 마지막 값은 정상 방출 처리
 |onBackpressureBuffer|Flux|`Backpressure` 에러가 발생한 요소들을 버퍼를 사용해서 후속처리 ([BufferOverflowStrategy](https://projectreactor.io/docs/core/3.4.6/api/reactor/core/publisher/BufferOverflowStrategy.html))


### retry(), retryWhen()
정해진 횟수, 혹은 `Retry` 클래스에서 제공하는 조건에 따른 시퀀스 구독 재시도를 수행할 수 있다.  

```java

@Test
public void flux_retry() {
	// given
	AtomicBoolean isError = new AtomicBoolean(true);
	Flux<String> source_1 = Flux.<String>create(sink -> {
		if (isError.get()) {
			sink.error(new Exception("my exception"));
			isError.set(false);
		} else {
			sink.next("success");
			sink.complete();
		}
	}).log();
	Flux<String> source = source_1.retry(2);

	StepVerifier
			.create(source)
			.expectNext("success")
			.verifyComplete();
}

/*
[main] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[main] INFO reactor.Flux.Create.1 - request(unbounded)
[main] ERROR reactor.Flux.Create.1 - onError(java.lang.Exception: my exception)
[main] ERROR reactor.Flux.Create.1 - 
java.lang.Exception: my exception
[main] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[main] INFO reactor.Flux.Create.1 - request(unbounded)
[main] INFO reactor.Flux.Create.1 - onNext(success)
[main] INFO reactor.Flux.Create.1 - onComplete()
 */

@Test
public void flux_retryWhen() {
	// given
	StopWatch stopWatch = new StopWatch();
	long timeoutMillis = 1000;
	Flux<String> source_1 = Flux.<String>create(sink -> {
		if (!stopWatch.isRunning()) {
			stopWatch.start();
		}

		stopWatch.stop();
		if (stopWatch.getTotalTimeMillis() < timeoutMillis) {
			stopWatch.start();
			sink.error(new Exception("my exception"));
		} else {
			sink.next("success");
			sink.complete();
		}
	}).log();
	Flux<String> source = source_1
			.retryWhen(Retry.fixedDelay(3, Duration.ofMillis(400)));

	StepVerifier
			.create(source)
			.expectNext("success")
			.verifyComplete();
}

/*
[main] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[main] INFO reactor.Flux.Create.1 - request(unbounded)
[main] ERROR reactor.Flux.Create.1 - onError(java.lang.Exception: my exception)
[main] ERROR reactor.Flux.Create.1 - 
java.lang.Exception: my exception
[parallel-1] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[parallel-1] INFO reactor.Flux.Create.1 - request(unbounded)
[parallel-1] ERROR reactor.Flux.Create.1 - onError(java.lang.Exception: my exception)
[parallel-1] ERROR reactor.Flux.Create.1 - 
java.lang.Exception: my exception
[parallel-2] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[parallel-2] INFO reactor.Flux.Create.1 - request(unbounded)
[parallel-2] ERROR reactor.Flux.Create.1 - onError(java.lang.Exception: my exception)
[parallel-2] ERROR reactor.Flux.Create.1 - 
java.lang.Exception: my exception
[parallel-3] INFO reactor.Flux.Create.1 - onSubscribe(FluxCreate.BufferAsyncSink)
[parallel-3] INFO reactor.Flux.Create.1 - request(unbounded)
[parallel-3] INFO reactor.Flux.Create.1 - onNext(success)
[parallel-3] INFO reactor.Flux.Create.1 - onComplete()
 */
```  

`retry()` 는 2번 횟수만큼 재시도 수행을 통해 정상적으로 시퀀스 방출이 완료 되었고, 
`retryWhen()` 은 `400 millis` 마다 3번 재시도 수행을 통해 정상적으로 시퀀스 방출이 완료된 것을 확인 할 수 있다.  

### onBackPressure*()
`Reactive Streams`(`Reactor`) 의 특징 중 `Backpressure`(배압) 은 생산자와 소비자간에 데이터 처리 속도를 
맞출 수 있는 호율적인 방법이다.
하지만 이러한 메커니즘이 있더라도 생산자, 소비자간의 데이터 처리 혹은 생산 속도 차이에 따른 
데이터 유실은 발생할 수 있다. 
이런 데이터 유실 상황에 대응 방법을 정의할 수 있는 것이 `onBackpressure*()` 메소드 들이다.  

테스트에서는 생산자와 소비자의 데이터 처리성능 차이를 두기 위해 `publishOn()` 을 사용해서, 
기존에 생산, 소비 작업이 비동기적으로 별도의 스레드에서 수행될 수 있도록 진행한다.  

```java
@Test
public void flux_onBackpressureError() throws Exception {
	// given
	List<Integer> actualValues = new LinkedList<>();
	Flux<Integer> source = Flux.range(1, 300)
			.onBackpressureError()
			.publishOn(Schedulers.boundedElastic());

	StepVerifier
			.create(source)
			.thenConsumeWhile(integer -> {
				try {
					Thread.sleep(10);
					actualValues.add(integer);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				return true;
			})
			.expectError(IllegalStateException.class)
			.verify();

	assertThat(actualValues, allOf(
			hasSize(greaterThan(1)),
			hasSize(lessThan(300))
	));
}


@Test
public void flux_onBackpressureDrop() throws Exception {
	// given
	List<Integer> actualValues = new LinkedList<>();
	List<Integer> actualDrops = new LinkedList<>();
	List<Integer> actualAll = new LinkedList<>();
	Flux<Integer> source = Flux.range(1, 300)
			.onBackpressureDrop(integer -> {
				actualDrops.add(integer);
				actualAll.add(integer);
				log.info("drop : {}", integer);
			})
			.publishOn(Schedulers.boundedElastic());

	// when
	StepVerifier
			.create(source)
			.thenConsumeWhile(
					integer -> true,
					integer -> {
						try {
							Thread.sleep(10);
							actualValues.add(integer);
							actualAll.add(integer);
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					})
			.verifyComplete();

	// then
	assertThat(actualValues, allOf(
			hasSize(greaterThan(1)),
			hasSize(lessThan(300))
	));
	assertThat(actualDrops, allOf(
			hasSize(greaterThan(1)),
			hasSize(lessThan(300))
	));
	assertThat(actualAll, hasSize(300));
	assertThat(actualValues, hasSize(actualAll.size() - actualDrops.size()));
	assertThat(actualDrops, hasSize(actualAll.size() - actualValues.size()));
	assertThat(actualValues, everyItem(not(oneOf(actualDrops.toArray()))));
}

/*
[main] INFO com.windowforsun.advanced.FluxTest - drop : 257
[main] INFO com.windowforsun.advanced.FluxTest - drop : 258

.. 생략 ..

[main] INFO com.windowforsun.advanced.FluxTest - drop : 299
[main] INFO com.windowforsun.advanced.FluxTest - drop : 300
 */

@Test
public void flux_onBackpressureLatest() {
	// given
	List<Integer> actualValues = new LinkedList<>();
	Flux<Integer> source = Flux.range(1, 300)
			.onBackpressureLatest()
			.publishOn(Schedulers.boundedElastic());

	StepVerifier
			.create(source)
			.thenConsumeWhile(integer -> true,
					integer -> {
						try {
							Thread.sleep(10);
							actualValues.add(integer);
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					})
			.verifyComplete();


	assertThat(actualValues, allOf(
			hasSize(greaterThan(1)),
			hasSize(lessThan(300))
	));
	assertThat(actualValues, hasItems(1, 300));
	assertThat(actualValues, not(hasItems(260, 270, 280, 290)));
}

@Test
public void flux_onBackpressureBuffer() {
	// given
	List<Integer> actualValues = new LinkedList<>();
	Flux<Integer> source = Flux.range(1, 300)
			.onBackpressureBuffer()
			.publishOn(Schedulers.boundedElastic());

	StepVerifier
			.create(source)
			.thenConsumeWhile(
					integer -> true,
					integer -> {
						try {
							Thread.sleep(10);
							actualValues.add(integer);
							log.info("consume : {}", integer);
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					}
			)
			.verifyComplete();

	assertThat(actualValues, hasSize(300));
}
```  

추가로 `onBackpressureBuffer()` 메소드는 예제에서 소개한 메소드외에도 많은 오버로딩 메소드가 존재한다. 
좀 더 다양하고 커스텀한 동작으로 `backpressure` 에러에 대한 처리를 수행할 수 있다.  


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


