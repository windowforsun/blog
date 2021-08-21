--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Peeking Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 시퀀스를 엿보는 Operator에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Peeking into a Sequence
toc: true 
use_math: true
---  

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Peeking into a Sequence(시퀀스 엿보기)

구분|메소드|타입|설명
---|---|---|---
최종 시퀀스에 영향 없음|doOnNext|Mono,Flux|시퀀스 아이템 방출 시그널 직전에 수행 할 동작
 |doOnComplete|Flux|시퀀스 완료 시그널 직전에 수행 할 동작
 |doOnSuccess|Mono|시퀀스 완료 시그널 직전에 수행 할 동작
 |doOnError|Mono,Flux|시퀀스 에러 시그널 직전에 수행 할 동작
 |doOnCancel|Mono,Flux|시퀀스 취소 시그널 직전에 수행 할 동작
 |doFirst|Mono,Flux|시퀀스 구독 요청 시점에 수행할 동작
 |doOnSubscribe|Mono,Flux|시퀀스 구독 시그널 전에 수행할 동작
 |doOnRequest|Mono,Flux|시퀀스 방출 요청 시그널 전에 수행할 동작
 |doOnTerminate|Mono,Flux|시퀀스 종료 시그널 전에 수행할 동작(완료, 에러 모두)
 |doAfterTerminate|Mono,Flux|시퀀스 종료 시그널 이후에 수행할 동작(완료, 에러 모두)
 |doOnEach|Mono,Flux|시퀀스에서 방출, 완료, 에러 시그널 전에 수행할 동작
 |doFinally|Mono,Flux|시퀀스에서 완료, 에러, 취소 시그널 이후 수행할 동작
 |materialize|Mono,Flux|시퀀스의 시그널이 아이템과 함께 방출
 |dematerialize|Mono,Flux|`materialize` 동작의 역순으로 다시 시퀀스 아이템 방출
 |log|Mono,Flux|시퀀스에서 수행되는 모든 시그널 및 동작에 대한 로그 출력

### log() 
디버깅할때 굉장히 유용하게 사용될 수 있는 `log()` 의 테스트 코드 결과는 아래와 같다.  

```java
@Test
public void flux_log() {
	// given
	Flux<String> source = Flux.just("first", "second", "third");

	source.log().subscribe(s -> log.info("sub : {}", s));
}
```  

```java
[main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO reactor.Flux.Array.1 - | request(unbounded)
[main] INFO reactor.Flux.Array.1 - | onNext(first)
[main] INFO com.windowforsun.advanced.FluxTest - sub : first
[main] INFO reactor.Flux.Array.1 - | onNext(second)
[main] INFO com.windowforsun.advanced.FluxTest - sub : second
[main] INFO reactor.Flux.Array.1 - | onNext(third)
[main] INFO com.windowforsun.advanced.FluxTest - sub : third
[main] INFO reactor.Flux.Array.1 - | onComplete()
```  

### do*() 
`do*()` 로 시작하는 `Listener` 성격을 갖는 메소드의 동작에 대한 예시는 아래와 같다. 
실제로 각 메소드가 어느 시점에 등록되었는지에 따라 메소드 실행 순서는 차이가 있을 수 있다. 
만약 `doOnEach()` 보다 `doOnNext()` 를 먼저 사용하면 `doOnNext()` 가 `doOnEach()` 보다 먼저 실행 되고, 
그 반대라면 반대 순서대로 실행 될 것이다.  

그리고 시퀀스에 구성은 돼있지만 메소드에 해당하는 시그널이 발생하지 않는다면 해당 메소드에 등록된 
동작은 수행되지 않는다. 
`doOnError()` 메소드를 시퀀스에 등록했다고 하더라도 실제 에러 시그널이 발생해야 메소드도 실행될 수 있다.  


```java
@Test
public void flux_do_complete() {
	// given
	Flux<String> source = Flux.just("first", "second", "third");

	source.log()
			.doFirst(() -> log.info("doFirst"))
			.doOnSubscribe(subscription -> log.info("doOnSubscribe"))
			.doOnRequest(value -> log.info("doOnRequest : {}", value))
			.doOnEach(stringSignal -> log.info("doOnEach : {}", stringSignal.get()))
			.doOnNext(s -> log.info("doOnNext : {}", s))
			.doOnComplete(() -> log.info("doOnComplete"))
			.doOnTerminate(() -> log.info("doOnTerminate"))
			.doAfterTerminate(() -> log.info("doOnAfterTerminate"))
			.doFinally(signalType -> log.info("doFinally : {}", signalType.toString()))
			.doOnError(throwable -> log.info("doOnError : {}", throwable.getMessage()))
			.doOnCancel(() -> log.info("doOnCancel"))
			.subscribe(s -> log.info("sub : {}", s), throwable -> log.info("error sub : {}", throwable.getMessage()));
}

/*
[main] INFO com.windowforsun.advanced.FluxTest - doFirst
[main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
[main] INFO com.windowforsun.advanced.FluxTest - doOnSubscribe
[main] INFO com.windowforsun.advanced.FluxTest - doOnRequest : 9223372036854775807
[main] INFO reactor.Flux.Array.1 - | request(unbounded)
[main] INFO reactor.Flux.Array.1 - | onNext(first)
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : first
[main] INFO com.windowforsun.advanced.FluxTest - doOnNext : first
[main] INFO com.windowforsun.advanced.FluxTest - sub : first
[main] INFO reactor.Flux.Array.1 - | onNext(second)
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : second
[main] INFO com.windowforsun.advanced.FluxTest - doOnNext : second
[main] INFO com.windowforsun.advanced.FluxTest - sub : second
[main] INFO reactor.Flux.Array.1 - | onNext(third)
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : third
[main] INFO com.windowforsun.advanced.FluxTest - doOnNext : third
[main] INFO com.windowforsun.advanced.FluxTest - sub : third
[main] INFO reactor.Flux.Array.1 - | onComplete()
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : null
[main] INFO com.windowforsun.advanced.FluxTest - doOnComplete
[main] INFO com.windowforsun.advanced.FluxTest - doOnTerminate
[main] INFO com.windowforsun.advanced.FluxTest - doFinally : onComplete
[main] INFO com.windowforsun.advanced.FluxTest - doOnAfterTerminate
 */


@Test
public void flux_do_error() {
	// given
	Flux<String> source = Flux
			.just("first")
			.mergeWith(Mono.error(new Exception("my exception")))
			.mergeWith(Mono.just("third"))
			;

	source.log()
			.doFirst(() -> log.info("doFirst"))
			.doOnSubscribe(subscription -> log.info("doOnSubscribe"))
			.doOnRequest(value -> log.info("doOnRequest : {}", value))
			.doOnEach(stringSignal -> log.info("doOnEach : {}", stringSignal.get()))
			.doOnNext(s -> log.info("doOnNext : {}", s))
			.doOnComplete(() -> log.info("doOnComplete"))
			.doOnTerminate(() -> log.info("doOnTerminate"))
			.doAfterTerminate(() -> log.info("doOnAfterTerminate"))
			.doFinally(signalType -> log.info("doFinally : {}", signalType.toString()))
			.doOnError(throwable -> log.info("doOnError : {}", throwable.getMessage()))
			.doOnCancel(() -> log.info("doOnCancel"))
			.subscribe(s -> log.info("sub : {}", s), throwable -> log.info("error sub : {}", throwable.getMessage()));
}

/*
[main] INFO com.windowforsun.advanced.FluxTest - doFirst
[main] INFO reactor.Flux.Merge.1 - onSubscribe(FluxFlatMap.FlatMapMain)
[main] INFO com.windowforsun.advanced.FluxTest - doOnSubscribe
[main] INFO com.windowforsun.advanced.FluxTest - doOnRequest : 92233720368547758
[main] INFO reactor.Flux.Merge.1 - request(unbounded)
[main] INFO reactor.Flux.Merge.1 - onNext(first)
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : first
[main] INFO com.windowforsun.advanced.FluxTest - doOnNext : first
[main] INFO com.windowforsun.advanced.FluxTest - sub : first
[main] ERROR reactor.Flux.Merge.1 - onError(java.lang.Exception: my exception)
[main] ERROR reactor.Flux.Merge.1 - 
java.lang.Exception: my exception
[main] INFO com.windowforsun.advanced.FluxTest - doOnEach : null
[main] INFO com.windowforsun.advanced.FluxTest - doOnTerminate
[main] INFO com.windowforsun.advanced.FluxTest - doOnError : my exception
[main] INFO com.windowforsun.advanced.FluxTest - error sub : my exception
[main] INFO com.windowforsun.advanced.FluxTest - doFinally : onError
[main] INFO com.windowforsun.advanced.FluxTest - doOnAfterTerminate
 */

@Test
public void flux_do_cancel() throws Exception{
	// given
	Flux<String> source = Flux.just("first", "second", "third").delayElements(Duration.ofMillis(100));

	Disposable disposable = source.log()
			.doFirst(() -> log.info("doFirst"))
			.doOnSubscribe(subscription -> log.info("doOnSubscribe"))
			.doOnRequest(value -> log.info("doOnRequest : {}", value))
			.doOnEach(stringSignal -> log.info("doOnEach : {}", stringSignal.get()))
			.doOnNext(s -> log.info("doOnNext : {}", s))
			.doOnComplete(() -> log.info("doOnComplete"))
			.doOnTerminate(() -> log.info("doOnTerminate"))
			.doAfterTerminate(() -> log.info("doOnAfterTerminate"))
			.doFinally(signalType -> log.info("doFinally : {}", signalType.toString()))
			.doOnError(throwable -> log.info("doOnError : {}", throwable.getMessage()))
			.doOnCancel(() -> log.info("doOnCancel"))
			.subscribe(s -> log.info("sub : {}", s));

	Thread.sleep(150);
	disposable.dispose();
}

/*
[main] INFO com.windowforsun.advanced.FluxTest - doFirst
[main] INFO reactor.Flux.ConcatMap.1 - onSubscribe(FluxConcatMap.ConcatMapImmediate)
[main] INFO com.windowforsun.advanced.FluxTest - doOnSubscribe
[main] INFO com.windowforsun.advanced.FluxTest - doOnRequest : 9223372036854775807
[main] INFO reactor.Flux.ConcatMap.1 - request(unbounded)
[parallel-1] INFO reactor.Flux.ConcatMap.1 - onNext(first)
[parallel-1] INFO com.windowforsun.advanced.FluxTest - doOnEach : first
[parallel-1] INFO com.windowforsun.advanced.FluxTest - doOnNext : first
[parallel-1] INFO com.windowforsun.advanced.FluxTest - sub : first
[main] INFO com.windowforsun.advanced.FluxTest - doOnCancel
[main] INFO reactor.Flux.ConcatMap.1 - cancel()
[main] INFO com.windowforsun.advanced.FluxTest - doFinally : cancel
 */
```  

### materialize(), dematerialize()

```java
@Test
public void flux_materialize() {
	// given
	Flux<String> source_1 = Flux.just("a", "b");
	Flux<Signal<String>> source = source_1.materialize();

	StepVerifier
			.create(source)
			.expectNext(Signal.next("a"))
			.expectNext(Signal.next("b"))
			.expectNext(Signal.complete())
			.verifyComplete();
}

@Test
public void flux_dematerialize() {
	// given
	Flux<Signal<String>> source_1 = Flux.just("a", "b").materialize();
	Flux<String> source = source_1.dematerialize();
	
	StepVerifier
			.create(source)
			.expectNext("a")
			.expectNext("b")
			.verifyComplete();
}
```  


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


