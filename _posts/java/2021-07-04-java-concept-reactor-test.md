--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Testing"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 의 시퀀스를 테스트할 수 있는 reactor-test 로 검증하는 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - StepVerifier
  - Testing
  - TestPublisher
  - PublisherProbe
  - reactor-test
toc: true 
use_math: true

---  

## Reactor Testing
`Reactor` 에서 제공하는 다양한 연산자를 사용해서 시퀀스를 구성하고, 그 시퀀스가 체인을 바탕으로 여러 작업을 수행된다면 그에 따른 테스트가 필요하다.

테스트를 수행할 수 있는 방법은 시퀀스가 이벤트에 따라 실제 동작이 잘 이뤄지는지 구독해 보는 것이다. 
간단한 방법으로 아래 처럼 `block()` 메소드를 사용해서 시퀀스에서 생산한 아이템을
바탕으로 테스트를 진행할 수 있을 것이다.

```java
@Test
public void mono_testing_block(){
	// given
	Mono<String> mono = Mono.just("item");

	// when
	String actual = mono.block();

	// then
	assertThat(actual,notNullValue());
	assertThat(actual,is("item"));
}
```  

하지만 `block()` 을 사용한 테스트는 단순히 시퀀스가 종료되고 생산된 아이템에 대해서만 테스트를 할 수 있다. 
그리고 `block()` 메소드 뿐만 아니라 `Mono`, `Flux` 에서 제공하는 다앙한
구독관련 메소드를 사용해서 테스트를 수행하더라도, 테스트 코드가 오히려 더 복잡해 질 수 있고 많은 제약 상황이 발생 할 수 있다.

이러한 이유로 `Reactor` 에서는 테스트에서 필요한 몇 가지 기능을 사용할 수
있는 [reactor-test](https://github.com/reactor/reactor-core/tree/main/reactor-test)
라는 의존성을 제공한다.

- `Maven`

```xml
<dependency>
	<groupId>io.projectreactor</groupId>
	<artifactId>reactor-test</artifactId>
	<scope>test</scope>
</dependency>
```  

- `Gradle`

```groovy
dependencies {
	testCompile 'io.projectreactor:reactor-test'
}
```  

`reactor-test` 의 주요 특징은 아래와 같다.

- `StepVerifier` 을 통해 시퀀스가 주어진 시나리오에 따라 수행되는지, 단계별로 테스트를 수행할 수 있다.
- `TestPublisher` 를 사용해서 다운스트림 연산자를 테스트할 수 있는 데이터를 생산한다.
- 여러 `Publisher` 로 구성된 시퀀스에서 해당 `Publisher` 가 다양한 조건 분기 상황에서 어떻게 사용되고 있는지 테스트를 수행할 수 있다.

### StepVerifier

가장 간단한 `Reactor` 관련 테스트는 하나의 `Mono`, `Flux` 시퀀스를 구독했을 떄, 실제로 어떻게 동작하는지에 대한 테스트 일 것이다. 시퀀스는 여러 이벤트를 바탕으로 구성되기 때문에 아래와 같은
테스트가 필요할 것이다.

- 현재 혹은 다음에 기대되는 이벤트에 대한 테스트
- 시퀀스에서 실제로 생산되는 아이템에 대한 테스트
- 시퀀스의 이벤트가 특정 주기 혹은 잠시 슬립이 있다면 실제로 정해진 시간에 따라 수행되는지에 대한 테스트

위와 같이 하나의 시퀀스를 구성하고 이를 구독했을 때 고려가 필요한 테스트를 `StepVerifier` 를 통해 할 수 있다.
`StepVerifier` 를 사용한 테스트는 아래와 같다.

```java
@Test
public void flux_stepVerifier() {
	// given
	Flux<String> source = Flux.just("first", "second");

	// then
	StepVerifier
			.create(source)
			.expectNext("first")
			.expectNextMatches(s -> s.startsWith("sec"))
			.expectComplete()
			.verify()
	;
}
```  

1. `create()` 메소드로 테스트를 수행할 시퀀스를 인자로 전달한다.
1. `expectNext()`, `expectMatches()` 등 시퀀스에서 생산하는 아이템을 검증하는 메소드를 사용해서 아이템을 순서대로 검증한다.
1. `expectComplete()` 메소드를 통해 시퀀스가 정상적으로 종료 되었는지 검증한다.
1. `verify()` 를 호출해서 작성한 검증 시나리오를 수행한다. (`verifyComplete()` 메소드를 사용해서 `expectComplete()` 와 `verifiy()` 를 한번에 수행할 수도 있다.)

`StepVerifier` 를 사용한 테스트 과정을 일반화 시키면 아래와 같다.

1. [StepVerifier](https://projectreactor.io/docs/test/release/api/index.html?reactor/test/StepVerifier.html)
   의 메소드를 사용해서 테스트 할 시퀀스 등록
1. [StepVerifier.FirstStep](https://projectreactor.io/docs/test/release/api/index.html?reactor/test/StepVerifier.FirstStep.html)
   초기 구독 이벤트 관련 검증 스텝
1. [StepVerifier.Step](https://projectreactor.io/docs/test/release/api/index.html?reactor/test/StepVerifier.Step.html)
   구독 과정에서 이뤄지는 모든 이벤트 검증 스텝
1. [StepVerifier.LastStep`](https://projectreactor.io/docs/test/release/api/index.html?reactor/test/StepVerifier.LastStep.html)
   구독 종료 이벤트 관련 검증 스텝
1. `StepVerifier` 을 통해 구성된 스텝 검증 메소드 호출
1. [StepVerifier.Assertions](https://projectreactor.io/docs/test/release/api/index.html?reactor/test/StepVerifier.Assertions.html)
   시퀀스 검증이 완료된 이후 시퀀스 상태에 대해서도 검증 검증

`StepVerifier` 에서 제공하는 다양한 메소드를 사용하면 아래와 같이 테스트를 구성할 수 있다.

```java
@Test
public void flux_stepVerifier_count() {
	// given
	Flux<String> source = Flux.just("first", "second");

	// then
	StepVerifier
			.create(source)
			.expectNextCount(2)
			.expectComplete()
			.verify()
	;
}

@Test
public void flux_stepVerifier_log() {
	// given
	Flux<String> source = Flux.just("first", "second");

	// then
	StepVerifier
			.create(source)
			.expectNextCount(1)
			.expectNext("second")
			.expectComplete()
			.log()
			.verify()
	;
}

/*
[main] DEBUG reactor.test.StepVerifier - Scenario:
[main] DEBUG reactor.test.StepVerifier - 	<defaultOnSubscribe>
[main] DEBUG reactor.test.StepVerifier - 	<expectNextCount(1)>
[main] DEBUG reactor.test.StepVerifier - 	<expectNext(second)>
[main] DEBUG reactor.test.StepVerifier - 	<expectComplete>
 */

@Test
public void flux_stepVerifier_consume() {
	// given
	Flux<String> source = Flux.just("first", "second");

	// then
	StepVerifier
			.create(source)
			.consumeNextWith(s -> assertThat(s, is("first")))
			.expectNext("second")
			.verifyComplete();
}

@Test
public void flux_stepVerifier_then() {
	// given
	Flux<String> source = Flux.just("first", "second");

	// then
	StepVerifier
			.create(source)
			.then(() -> log.info("before first"))
			.expectNext("first")
			.then(() -> log.info("after first"))
			.then(() -> log.info("before second"))
			.expectNext("second")
			.then(() -> log.info("after second"))
			.verifyComplete();
}

/*
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - before first
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - after first
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - before second
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - after second
 */
```  

### Exception Testing

시퀀스에서 발생할 수 있는 예외에 대한 검증도 `StepVerifier` 를 통해 가능하다. 아래는 간단한 몇가지 예시이다.

```java
public static class MyException extends Exception {
	public MyException(String message) {
		super(message);
	}
}

@Test
public void flux_error_expectErrorMatches() {
	// given
	Flux<String> source = Flux
		.just("first", "second")
		.concatWith(Mono.error(new MyException("flux error")));

	// then
	StepVerifier
		.create(source)
		.expectNextCount(2)
		.expectErrorMatches(throwable ->
			throwable instanceof MyException
				&& throwable.getMessage().equals("flux error"))
		.verify()
	;
}

@Test
public void flux_error_expectErrorSatisfies() {
	// given
	Flux<String> source = Flux
		.just("first", "second")
		.concatWith(Mono.error(new MyException("flux error")));

	// then
	StepVerifier
		.create(source)
		.expectNextCount(2)
		.expectErrorSatisfies(throwable ->
			assertThat(throwable, instanceOf(MyException.class)))
		.verify()
	;
}

@Test
public void flux_error_expectErrorMessage() {
	// given
	Flux<String> source = Flux
		.just("first", "second")
		.concatWith(Mono.error(new MyException("flux error")));

	// then
	StepVerifier
		.create(source)
		.expectNextCount(2)
		.expectErrorMessage("flux error")
		.verify()
	;
}
```  

### 시간기반 테스트

시퀀스가 시작되고 완료될 때까지 오랜시간이 걸린다면, 테스트 한번을 위해 오랜 시간이 필요할 것이다.  
`Stepverifier` 의 시간 기반 연산자를 사용하면 오랜 시간이 걸리는 코드를 실제로 기다리지 않고 테스트를 수행할 수 있다. 지금까지는 테스트를 수행할 시퀀스를 등록할 때 `create()` 메소드를 사용
했다면, 가상 시간을 사용해서 테스트를 수행하고 싶을 때는 `withVirtualTime()` 메소드를 사용한다.

`withVirtualTime()` 메소드는 내부적으로 `VirtualTimeScheduler` 를 사용하기 때문에 가상 시간 스케쥴러를 활성화 한 이후에 연산자 초기화가 필요하다.
이는 `withVirtualTime()` 메소드가 인자로 받는 `Supplier<Publisher<T>>` 는 `Lazy` 방식으로 사용해야 정상동작이 가능하다. 만약 그렇지 않으면 가상시간 동작을 보장할 수 없게
된다. 또한 테스트 코드상에서 시퀀스를 초기화한 후 `Supplier` 에서 리턴해도 않되고,
`Supplier` 람다 내부에서 시퀀스를 초기화하고 리턴해야 한다.

이후 테스트에서는 `verify()` 메소드가 리턴하는 검증 소요시간 `Duration` 을 기반으로 시퀀스 검증이 실제 시간으로 수행됐는지, 가상 시간을 기반으로 수행됐는지 살펴 볼 것이다.

가상시간을 사용하는 테스트에서 주로 사용되는 검증용 메소드는 아래와 같다.

- `thenAwait(Duration)` : 인자로 전달된 시간 만큼 스텝 검증을 멈춘다. (검증만 멈추는 것이기 때문에 시퀀스에서 신호는 그대로 수행된다.)
- `expectNoEvent(Duration)` : 인자로 전달된 시간 만큼 시퀀스를 그대로 수행하지만, 시간내에 시퀀스에서 신호가 하나라도 발새앟면 테스트는 실패한다.

위 메소드는 일반적인 시퀀스 테스트에서는 실제로 주어진 시간만큼 스레드를 중지하고, 가상 시간인 경우에만 가상 시계를 사용해서 실제 시간을 기다리지 않으면서 테스트를 수행할 수 있다.

시퀀스에 구독이 시작되면 가장 먼저 `subscription` 이벤트가 전달된다. 이때 `expectNoEvent` 는 `subscription` 도 하나의 이벤트로 간주하기 때문에,
`expectNoEvent` 사용전에 아래와 같이 `expectSubscription()` 을 사용해 줘야한다.

```java
StepVerifier
	.withVirtualTime(...)
	.expectSubscription()
	.expectNoEvent(...)
```  

이제 테스트 코드를 하나씩 보며 실제 사용방법에 대해 살펴본다. 먼저 가상 시간을 사용하는 기본적인 케이스는 아래와 같다.

```java
@Test
public void flux_longtime_virtualtime() {
	Duration actual = StepVerifier
			.withVirtualTime(() -> Flux
					.just("first", "second", "third")
					.delayElements(Duration.ofDays(1)))
			.expectSubscription()
			.expectNoEvent(Duration.ofDays(1))
			.expectNext("first")
			.expectNoEvent(Duration.ofDays(1))
			.expectNext("second")
			.thenAwait(Duration.ofDays(1))
			.expectNext("third")
			.verifyComplete();

	assertThat(actual, lessThanOrEqualTo(Duration.ofSeconds(1)));
}
```

위 테스트 코드에서 시퀀스는 하루에 한번씩 `first`, `second`, `third` 문자열을 생산한다. 실제 구독해서 테스트를 수행한다면 3일이라는 시간이 걸려야 검증을 완료할 수 있다.
`subscription` 이벤트를 시작으로 `expectNoEvent()`, `thenAwait()` 메소드에 하루 기간을 인자로 전달해 테스트를 수행했다. 그리고 `verifyComplete()` 에서 리턴되는
검증 소요 시간을 살펴보면 1초 미만으로 가상 시간 기반으로 테스트가 수행된 것을 확인 할 수 있다.

다음으로 시퀀스 등록이 `Lazy` 하게 수행되지 못해 실제 시간으로 검증이 수행되는 케이스로 주의가 필요한 경우이다.

```java
@Test
public void flux_longtime_realtime() {
	// given
	Flux flux = Flux.just("first", "second").delayElements(Duration.ofSeconds(2));

	Duration actual = StepVerifier
			.withVirtualTime(() -> flux)
			.expectSubscription()
			.expectNoEvent(Duration.ofSeconds(2))
			.expectNext("first")
			.thenAwait(Duration.ofSeconds(2))
			.expectNext("second")
			.verifyComplete();

	assertThat(actual, greaterThanOrEqualTo(Duration.ofSeconds(4)));
}
```  

위 테스트 코드는 시퀀스 초기화가 `withVirtualTime()` 메소드 람다식 외부에서 수행 되고 객체만 내부에서 리턴하고 있다. 시퀀스는 하루마다 아이템을 생성하는게 아니라, 2초마다 아이템을 생성하기
때문에 `expectNoEvent()`, `thenAwait()` 메소드도 모두 2초로 검증을 수행한다. 그리고 검증에 소요된 시간을 살펴보면 4초 이상으로 실제 시퀀스에 소요된 시간과 비슷한 시간이므로 가상
시간모드로 수행되지 못한 것을 확인 할 수 있다.

`expectNoEvent(Duration)` 메소드의 경우 인자로 전달된 시간 내에 시그널이 발생하면 검증은 실패하므로 아래와 같이 케이스를 나눌 수 있다.

- `Duration < 실제 시그널이 발생하는 시간` : `Blocking` 이 걸려 이후 검증이 수행되지 못함
- `Duration > 실제 시그널이 발생하는 시간` : 검증 실패

```java
@Test
public void flux_expectNoEvent_too_long() {
	Throwable actual = assertThrows(AssertionError.class,
			() -> StepVerifier
					.withVirtualTime(() -> Flux
							.just("first")
							.delayElements(Duration.ofDays(1))
					)
					.expectSubscription()
					.expectNoEvent(Duration.ofDays(2))
					.expectNext("first")
					.verifyComplete()
	);

	assertThat(actual.getMessage(), allOf(
			containsString("expected no event: onComplete()"),
			containsString("expected no event: onNext(first)")
	));
}
```  

`thenAwait(Duration)` 메소드는 테스트 검증 수행만 주어진 시간만큼 딜레이 시키는 것이기 때문에 아래와 같은 경우가 있다.

- `Duration < 실제 시그널이 발생하는 시간` : `Blocking` 이 걸려 이후 검증이 수행되지 못함
- `Duration > 실제 시그널이 발생하는 시간` : 정상적으로 이후 테스트 검증 수행

```java
@Test
public void flux_thenAwait_too_long() {
	Duration actual = StepVerifier
			.withVirtualTime(() -> Flux
					.just("first")
					.delayElements(Duration.ofDays(1))
			)
			.expectSubscription()
			.thenAwait(Duration.ofDays(2))
			.expectNext("first")
			.verifyComplete();

	assertThat(actual, lessThanOrEqualTo(Duration.ofSeconds(1)));
}
```  

### 시퀀스 검증 이후 검증 수행

시퀀스 검증 마지막에 `verify()` 대신 `verifyThenAssertThat()` 를 사용하면,
`StepVerifier.Assertions` 객체를 리턴하기 때문에 검증 시나리오가 끝난 이후 몇가지 상태를 검증할 수 있다. 주로 `onComplete` 으로 시퀀스가 모두 종료된 상태에서 호출
되는 `onNext`, `onError` 시그널 검증을 할 수 있다.

```java
@Test
public void flux_verifyThenAssertThat() {
	// given
	Flux<String> source = Flux.create(fluxSink -> {
		fluxSink.next("first").next("second");
		fluxSink.complete();

		try {
			Thread.sleep(100L);
		} catch (Exception e) {
			e.printStackTrace();
		}

		fluxSink.next("third");
	});

	StepVerifier
			.create(source)
			.expectNext("first")
			.expectNext("second")
			.expectComplete()
			.verifyThenAssertThat()
			.hasDropped("third")
			.tookMoreThan(Duration.ofMillis(100))
			.tookLessThan(Duration.ofMillis(150))
	;
}
```  

위 테스트는 `Flux.create()` 메소드를 사용해서 `first`, `second` 를 생산하고 `onComplete` 시그널이 수행된 후,
`100 millis` 대기 후 `third` 를 생상하는 시퀀스를 만들었다. 그리고 이를 검증하기 위해 `verifyThenAssertThat()` 을 사용해서 `onComplete` 이후 `third` 아이템이
실제로 생산 되었느지와 전체 소요시간 이 `100 millis < 소요시간 < 150 millis` 인지 검증한 것을 확인 할 수 있다.

### TestPublisher

소스가 되는 시퀀스의 시그널을 좀더 디테일하게 조작하면서 실제 발생될 수 있는 상황에 대한 테스트를 수행하야 할 수도 있다. 이때 `TestPublisher` 를 사용하면 테스트 코드에서 시퀀스의 이벤트를 좀더
자유롭게 조작하면서 시퀀스가 정상적으로 동작하는지 검증 할 수 있다.
`TestPublisher` 에서 제공하는 메소드는 아래와 같다.

- `next(T)`, `next(T, T, ...)` : `onNext` 신호를 트리거 한다.
- `emit(T, T, ...)` : `onNext` 신호를 트리거 한 이후 `complete()` 을 호출한다.
- `complete()` : `onComplete` 신호를 트리거하고 시퀀스를 종료한다.
- `error(Throwable)` : `onError` 신호를 트리거하고 시퀀스를 종료한다.

`TestPublisher` 를 사용한 간단한 테스트 예시는 아래와 같다.

```java
public static Flux<String> myUpperCase(Flux<String> source){
	return source.map(String::toUpperCase);
}

@Test
public void flux_testPublisher() {
    // given
	TestPublisher<String> testPublisher = TestPublisher.create();
	Flux<String> source = testPublisher.flux();

	StepVerifier
			.create(myUpperCase(source))
			.then(() -> testPublisher
				.next("first")
				.emit("second", "third")
			)
			.expectNext("FIRST", "SECOND", "THIRD")
			.verifyComplete()
	;
}
```  

위 테스트는 `myUpperCase()` 메소드가 정상적으로 모든 시그널을 처리해서 기대하는 값을 도출해 내는지에 대한 검증이다. 
`TestPublisher` 를 사용해서 데이터를 생산할 시퀀스를 만들어 둔다. 
그리고 `TestPublisher` 객체를 통해 데이터 소스 시퀀스를 만든다. 
`then()` 메소드에서 `TestPubliher` 를 사용해서 테스트에서 필요한 신호를 구성하게 되면 `TestPublisher` 를 바탕으로 다양한 로직에 따른 신호에 대한 검증을 수행할 수 있다.  

`TestPublisher` 에서 `createNonCompliant()` 메소드를 사용하면 정상적으로 동작하지 않는 `TestPublisher` 를 생성할 수 있다. 
생성할 수 있는 종류는 `TestPublisher.Violation` 의 `enum` 값으로 구성된다. 
- `REQUEST_OVERFLOW` : 요청이 충분하지 않을 때도 `IllegalStationException` 을 트리거하는 대신 `next` 호출을 허용한다. 
- `ALLOW_NUL` : `null` 값이 들어오면 `NullPointException` 을 트리거 하는 대신 `next` 호출을 허용한다. 
- `CLEANUP_ON_TERMINATE` : `row` 하나에서 종료 신호를 여러 번 허용한다. (`complete()`, `error()`, `emit()`)
- `DEFER_CANCELLATION` : `TestPublisher` 가 취소 신호를 무시하고, 마치 신호가 밀린 것처럼 계속해서 신호를 방출할 수 있다. 

아래는 정상적으로 동작하지 않는 `TestPublisher` 를 생성하고 검증을 수행한 테스트 코드 예시이다. 

```java
@Test
public void flux_testPublisher_allow_null() {
    // given
	TestPublisher<String> testPublisher = TestPublisher
			.createNoncompliant(TestPublisher.Violation.ALLOW_NULL);
	Flux<String> source = testPublisher.flux();

	StepVerifier
			.create(myUpperCase(source))
			.then(() -> testPublisher
					.next("first")
					.emit("second", null)
			)
			.expectNext("FIRST", "SECOND")
			.expectError(NullPointerException.class)
			.verify();
	;
}

@Test
public void flux_testPublisher_cleanup_on_terminate() {
    // given
	TestPublisher<String> testPublisher = TestPublisher
			.createNoncompliant(TestPublisher.Violation.CLEANUP_ON_TERMINATE);
	Flux<String> source = testPublisher.flux();

	StepVerifier
			.create(myUpperCase(source))
			.then(() -> testPublisher
					.next("first")
					.emit("second", "third")
					.complete()
					.error(new Exception("myException"))
			)
			.expectNext("FIRST", "SECOND", "THIRD")
			.expectComplete()
			.verifyThenAssertThat()
			.hasDroppedErrorWithMessage("myException");
}
```  

`emit()` 메소드는 인자값으로 `null` 값 전달이 불가능하다. 하지만 `ALLOW_NULL` 옵셥으로 `null` 값을 시퀀스의 아이템으로 생산할 수 있지만, 
예외가 발생하는 것을 확인 할 수 있다. 
다음으로 하나의 시퀀스에서 `onComplete()` 와 `onError()` 는 둘중 하나만 한번만 호출 될 수 있지만, 
`CLEANUP_ON_TERMINATE` 옵션을 사용해서 여러번 가능하도록 설정해서 검증을 수행한 것을 확인 할 수 있다.    


### PublisherProbe
시퀀스가 복잡한 체인으로 구성돼 있는 상태라면 조건에 따라 서로 다른 하위 시퀀스로 분기되는 상황이 있을 수 있다. 
간단한 예시로 아래와 같이 `source` 가 비어있는 경우 `swithIfEmpty()` 를 사용해서 `Fallback` 을 수행하는 경우를 가정해 본다.  

```java
public Flux<String> doFallback(Flux<String> source, Publisher<String> fallback) {
	return source
			.flatMap(s -> Flux.just(s.toUpperCase()))
			.switchIfEmpty(fallback);
}

@Test
public void doFallback_notEmpty() {
    // given
	Flux<String> source = Flux.just("first", "second", "third");
	Mono<String> fallback = Mono.just("empty");

	StepVerifier
			.create(doFallback(source, fallback))
			.expectNext("FIRST", "SECOND", "THIRD")
			.verifyComplete();
}

@Test
public void doFallback_empty() {
    // given
	Flux<String> source = Flux.empty();
	Mono<String> fallback = Mono.just("empty");

	StepVerifier
			.create(doFallback(source, fallback))
			.expectNext("empty")
			.verifyComplete();
}
```  

위 테스트 처럼 시퀀스가 조건에 따라 서로다른 하위 시퀀스로 구성될 때, 
시퀀스에 결과값이 존재하는 경우 모든 경우에 따라 하위 시퀀스들이 정상적으로 동작하는지는 결과값을 바탕으로 검증을 수행할 수 있다.  

하지만 아래와 같이 조건에 따라 하위 시퀀스로 분기되지만 시퀀스에 결과 값이 존재하지 않는 경우를 생각해보자. 

```java

public Mono<Void> doFallbackVoid(Flux<String> source, Publisher<String> fallback) {
	return source
			.flatMap(s -> Flux.just(s.toUpperCase()))
			.switchIfEmpty(fallback).then();
}
```  

`source` 시퀀스가 비었는지 아닌지에 따라 하위 시퀀스가 분기되는 상태이다. 
코드에서는 정확하게 표현되지 못했지만, 
이렇게 시퀀스에 결과가 존재하지 않는 경우는 결과를 조건에 따란 서로 다른 외부 저장소에 저장한다는 등의 상황이 있을 수 있다. 
위와 같은 상태에서는 외부 저장소를 통해서만 실제 하위 시퀀스가 정상적으로 수행되었는지 검증을 수행할 수 있다.  

이때 `PublisherProbe` 로 시퀀스를 만들어 전달하게 되면 해당 시퀀스가 어떤 이벤트를 받았고 처리 됐는지 검증을 수행 할 수 있다. 
`PublisherProbe` 를 사용해서 조건에 따라 하위 시퀀스를 분기해 처리하는 `doFallbackVoid()` 메소드를 테스트하면 아래와 같다.  

```java
@Test
public void doFallbackVoid_notEmpty() {
	// given
	Flux<String> source = Flux.just("first", "second", "third");
	PublisherProbe<String> probeSource = PublisherProbe.of(source);
	PublisherProbe<String> probeFallback = PublisherProbe.of(Mono.just("empty"));

	// when
	StepVerifier
			.create(doFallbackVoid(probeSource.flux(), probeFallback.mono()))
			.verifyComplete();

	// then
	probeSource.assertWasSubscribed();
	probeSource.assertWasRequested();
	probeSource.assertWasNotCancelled();
	probeFallback.assertWasNotSubscribed();
	probeFallback.assertWasNotRequested();
	probeSource.assertWasNotCancelled();
}

@Test
public void doFallbackVoid_empty() {
	// given
	Flux<String> source = Flux.empty();
	PublisherProbe<String> probeSource = PublisherProbe.of(source);
	PublisherProbe<String> probeFallback = PublisherProbe.of(Mono.just("empty"));

	// when
	StepVerifier
			.create(doFallbackVoid(probeSource.flux(), probeFallback.mono()))
			.verifyComplete();

	// then
	probeSource.assertWasSubscribed();
	probeSource.assertWasRequested();
	probeSource.assertWasNotCancelled();
	probeFallback.assertWasSubscribed();
	probeFallback.assertWasRequested();
	probeSource.assertWasNotCancelled();
}
```  

테스트 코드를 보면 `PublisherProbe` 로 `doFallbackVoid()` 메소드에 전달할 시퀀스를 생성했다. 
그리고 `source` 가 비어있지 않은 경우의 테스트를 보면 `probeSource` 는 `subscription`, `request` 등의 시그널이 수행되고, 
`cancel` 은 이뤄지지 않았다. `probeFallback` 는 `subscription`, `request`, `cancel` 시그널이 모두 이뤄지지 않은 것을 확인 할 수 있다.  

다음으로 `source` 가 비어있는 경우를 보면 `probeSource` 는 비어있지 않은 경우와 동일하게 `subscription`, `reuquest` 시그널만 이뤄졌지만, 
`probeFallback` 은 `source` 가 비었기 때문에 `subscription`, `request` 시그널이 모두 수행된 것을 확인 할 수 있다.


### 테스트 실패한 스텝 찾기
`StepVierfier` 를 사용해서 복잡한 시퀀스를 검증하는 과정에서 실패를 하게 될때, 검증 실패가 정확하게 어느 스텝에서 발생했는지 찾기 어려울 수 있다. 이러한 상황에서 `as(desc)`
를 `expect*()` 메소드 뒤에 사용해서 검증문에 대한 설명을 추가할 수 있다. 해당 검증문이 실패하면 `as()` 메소드에 작성한 설명을 포함한 에러 메시지와 함께 출력된다. 마지막 종료
검증문과 `verify()` 에는 사용할 수 없다.

```java
@Test
public void flux_test_failures_as(){
	// given
	Flux<String> source=Flux.just("first","second");

	// then
	StepVerifier
			.create(source)
			.expectNext("first")
			.as("first is not first")
			.expectNext("third")
			.as("second is not third")
			.verifyComplete();
}
```  

```
java.lang.AssertionError: expectation "second is not third" failed (expected value: third; actual value: second)
```  

위 테스트 코드는 시퀀스가 생성하는 2번째 아이템이 `third` 가 아니기 때문에 `expectNext("third")` 부분에서 실패하게 된다. 
테스트가 실패하면 위처럼 검증문 아래 작성한 `second is not third` 가 출력되는 것을 확인 할 수 있다.  

---
## Reference
[6. Testing](https://projectreactor.io/docs/core/release/reference/#testing)  
[Testing Reactive Streams Using StepVerifier and TestPublisher](https://www.baeldung.com/reactive-streams-step-verifier-test-publisher)  
[reactor-test](https://projectreactor.io/docs/test/release/api/index.html)  

