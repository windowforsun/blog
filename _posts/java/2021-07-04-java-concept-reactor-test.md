--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Testing"
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
  - Reactor
  - StepVerifier
  - Testing
toc: true
use_math: true
---  

## Reactor Testing
`Reactor` 에서 제공하는 다양한 연산자를 사용해서 시퀀스를 구성하고, 
그 시퀀스가 체인을 바탕으로 여러 작업을 수행된다면 그에 따른 테스트가 필요하다.  

실제로 테스트를 수행할 수 있는 방법은 시퀀스가 이벤트에 따라 실제 동작이 잘 이뤄지는지 실제로 구독해 보는 것이다. 
간단한 방법으로 아래 처럼 `block()` 메소드를 사용해서 시퀀스에서 실제로 생산한 아이템을 바탕으로 테스트를 진행할 수 있을 것이다. 

```java
@Test
public void mono_testing_block() {
	// given
	Mono<String> mono = Mono.just("item");

	// when
	String actual = mono.block();

	// then
	assertThat(actual, notNullValue());
	assertThat(actual, is("item"));
}
```  

하지만 `block()` 을 사용한 테스트는 단순히 시퀀스가 종료되고 생산된 아이템에 대해서만 테스트를 할 수 있다. 
그리고 `block()` 메소드 뿐만 아니라 `Mono`, `Flux` 에서 제공하는 다앙흔 구독관련 메소드를 사용해서 테스트를 수행하더라도, 
테스트 코드가 오히려 더 복잡해 질 수 있고 다양한 제약 상황이 발생 할 수 있다.  

이러한 이유로 `Reactor` 에서는 테스트에서 필요한 몇 가지 기능을 사용할 수 있는 [reactor-test](https://github.com/reactor/reactor-core/tree/main/reactor-test)
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
- `StepVerifier` 을 통해 시퀀스가 주어진 시나리오에 따라 수행되는지, 
단계별로 테스트를 수행할 수 있다.
- `TestPublisher` 를 사용해서 다운스트림 연산자를 테스트할 수 있는 데이터를 생산한다. 
- 여러 `Publisher` 로 구성된 시퀀스에서 해당 `Publisher` 가 다양한 조건 분기 상황에서 
어떻게 사용되고 있는지 테스트를 수행할 수 있다. 
  
### StepVerifier
가장 간단한 `Reactor` 관련 테스트는 하나의 `Mono`, `Flux` 시퀀스를 구독했을 떄, 
실제로 어떻게 동작하는지에 대한 테스트 일 것이다. 
시퀀스는 여러 이벤트를 바탕으로 구성되기 때문에 아래와 같은 테스트가 필요할 것이다. 
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
시퀀스 검증 후 상태 확인

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
	.		verifyComplete();
}

/*
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - before first
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - after first
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - before second
[main] INFO com.windowforsun.reactor.testing.StepVerifierTest - after second
 */
```  

### Exception Testing
시퀀스에서 발생할 수 있는 예외에 대한 검증도 `StepVerifier` 를 통해 가능하다. 
아래는 간단한 몇가지 예시이다.  

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




### 테스트 실패한 스텝 찾기
`StepVierfier` 를 사용해서 복잡한 시퀀스를 검증하는 과정에서 실패를 하게 될때, 
검증 실패가 정확하게 어느 스텝에서 발생했는지 찾기 어려울 수 있다. 
이러한 상황에서 `as(desc)` 를 `expect*()` 메소드 뒤에 사용해서 검증문에 대한 설명을 추가할 수 있다. 
해당 검증문이 실패하면 `as()` 메소드에 작성한 설명을 포함한 에러 메시지와 함께 출력된다. 
마지막 종료 검증문과 `verify()` 에는 사용할 수 없다.  

```java
@Test
public void flux_test_failures_as() {
	// given
	Flux<String> source = Flux.just("first", "second");

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





---
## Reference
[6. Testing](https://projectreactor.io/docs/core/release/reference/#testing)  
[Testing Reactive Streams Using StepVerifier and TestPublisher](https://www.baeldung.com/reactive-streams-step-verifier-test-publisher)  

