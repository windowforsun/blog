--- 
layout: single
classes: wide
title: "[Java 실습] "
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
  - Reactive Stream
  - Reactor
  - Reactor Extra
  - BooleanUtils
toc: true 
use_math: true
---  

## Reactor Error Handling
`Reactor` 혹은 `Spring Webflux` 에서 `Mono`, `Flux` 를 통해 `Reactive Stream` 을 
처리하다 보면 `Exception` 발생으로 인한 `Error` 처리가 필요한 상황이 많다. 
발생한 `Exception` 을 그대로 던지는 것이 아니라, `Reactive Stream` 상에서 `Graceful` 하게 
`Error` 를 처리할 수 있는 전략에 대해 알아본다.  

`Reactive Stream` 상에서 예외가 발생하면, 발생한 `Reactive Stream` 위치를 기준으로 가장 가까운 `downstream` 의 
`Error Handler` 가 최종적으로 수행된다. 
즉 에러가 발생한 위치에서 `upstream` 에만 `Error Handler` 가 있다면 `Error Handling` 은 수행되지 않는다.

```java
@Test
public void error_handling() {
    Mono.just("2")
            .onErrorReturn("fallback0")
            .flatMap(this::getMonoResult)
            .onErrorReturn("fallback1")
            .onErrorReturn("fallback2")
            .as(StepVerifier::create)
            .expectNext("fallback1")
            .verifyComplete();
}

@Test
public void not_error_handing() {
    Mono.just("2")
            .onErrorReturn("fallback")
            .flatMap(this::getMonoResult)
            .as(StepVerifier::create)
            .expectError(IllegalArgumentException.class)
            .verify();
}
```  

위 경우를 보면 에러가 발생하는 `flatMap(this::getMonoResult)` 에서 가장 가까우면서 `downstream` 인 `fallback1` 
예외처리가 수행되는 것을 확인 할 수 있다.  

`Error` 처리 시뮬레이션을 위해 사용한 `getMonoResult()` 메서드의 형태와 자세한 내용은 아래와 같다. 
이후 예제에서 발생하는 모든 예외는 `i` 값이 짝수일 때 `RuntimeException` 를 상속하는 `IllagalArgumentException` 임을 기억하자. 

```java
public String getResult(String i) {
    log.info("execute getResult : {}", i);

    if(Integer.parseInt(i) % 2 == 0) {
        throw new IllegalArgumentException("test exception");
    }

    return "result:" + i;
}

public Mono<String> getMonoResult(String i) {
    return Mono.fromSupplier(() -> this.getResult(i));
}
```  

`Error Handling` 과 관련된 `Reactor` 메서드들은 아래와 같이 3가지의 조건이 다른 파라미터를 갖는 형태로 구성되는데, 
가장 먼저 알아볼 `onErrorReturn` 을 예시로 들면 아래와 같다.  

```java
// 발생한 모든 예외 대신 방출할 fallback 값을 설정한다. 
public final Mono<T> onErrorReturn(final T fallback);

// 특정 예외 티입이 발생 했을 떄 방출할 fallback 값 설정한다. 
public final <E extends Throwable> Mono<T> onErrorReturn(Class<E> type, T fallbackValue);

// 예외를 별도 Predicate 구현으로 판별해 fallback 값을 방출할지 결정한다. 
public final Mono<T> onErrorReturn(Predicate<? super Throwable> predicate, T fallbackValue);
```  

### onErrorReturn
`onErrorReturn` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때, 
방출 할 지정된 값인 `Fallback value` 를 사용해서 `Error Handling` 하는 방법이다.  

reactor-error-handling-1.svg

```java
@Test
public void mono_onErrorReturn() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorReturn(RuntimeException.class, "fallback")
            .as(StepVerifier::create)
            .expectNext("fallback")
            .verifyComplete();
}

@Test
public void flux_onErrorReturn() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorReturn("fallback")
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("fallback")
            .verifyComplete();
}
```  

`Mono` 예제에서는 `RuntimeException` 에 해당하는 예외에 대해서 `fallback` 이라는 값을 방출하도록
선언 했기 때문에 예외 대신 `fallback` 이라는 문자열이 방출되는 것을 확인 할 수 있다.  

`Flux` 예제를 보면 원본 `source` 에서는 3개의 아이템을 방출하지만, 
첫번째 아이템은 정상적으로 결과를 확인 할 수 있지만 두번째 아이템에서 예외 발생으로 `fallback` 이라는 
문자열이 방출되고 스트림은 종료된다.  

### onErrorResume
`onErrorResume` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때, 
이어서 방출할 `Fallback publisher` 를 사용해서 `Error Handling` 을 하는 방법이다. 

reactor-error-handling-2.svg

```java
@Test
public void mono_onErrorResume() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorResume(throwable -> throwable.getMessage().equals("test exception"),
                    e -> Mono.just("fallback"))
            .doOnNext(s -> log.info("{}", s))
            .as(StepVerifier::create)
            .expectNext("fallback")
            .verifyComplete();
}

@Test
public void flux_onErrorResume() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorResume(throwable -> Flux.fromIterable(List.of("3", "5"))
                    .flatMap(this::getMonoResult))
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:3")
            .expectNext("result:5")
            .verifyComplete();
}
```  

`Mono` 예제에서는 `Predicate` 구현을 통해 `Error message` 가 `test exception` 인 경우 
`fallback` 문자열을 방출하는 스트림을 구독해서 이어 방출하도록 `Error Handling` 을 수행했다.  

`Flux` 예제는 초기 `Source` 는 `1, 2, 3` 3개의 아이템을 방출하도록 하지만
두 번째 아이템인 `2`에서 예외가 발생하게 된다. 
그러면 `3, 5` 를 사용해서 결과를 만드는 스트림을 구독해서 이어 방출하도록 `Error Handling` 을 수행했다.  

### onErrorMap
`onErrorMap` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때,
던질 예외를 `Mapper` 를 사용해서 필요한 예외 타입으로 변경해서 `Error Handling` 을 하는 방법이다.

reactor-error-handling-3.svg

```java
@Test
public void mono_onErrorMap() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorMap(throwable -> new ClassNotFoundException())
            .as(StepVerifier::create)
            .expectError(ClassNotFoundException.class)
            .verify();
}

@Test
public void flux_onErrorMap() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorMap(throwable -> new ClassNotFoundException())
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectError(ClassNotFoundException.class)
            .verify();
}
```  
`Mono` 예제에서는 `upstream` 에서 발생하는 모든 예외에 대해서 `ClassNotFoundException` 으로 
변환해 `downstream` 으로 예외를 전달하는 방식으로 `Error Handling` 을 수행하였다.  

`Flux` 또한 `upstream` 에서 발생하는 모든 예외를 `ClassNotFoundException` 으로
치환해 `donwstream` 으로 전달하기 때문에 두 번째 아이템에서 치환된 예외가 전달된 것을 확인 할 수 있다.  


### onErrorContinue
`onErrorContinue` 는 `Reactive Stream` 중 `Error` 가 발생 했을 때,
`Error consumer` 를 사용해서 예외를 처리(로깅,..)하고 남은 아이템들을 이어서 처리하는 `Error Handling` 방식이다.  

reactor-error-handling-3.svg

```java
@Test
public void mono_onErrorContinue() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .onErrorContinue((throwable, o) -> log.info("error at : {}, {}", o, throwable.getMessage()))
            .as(StepVerifier::create)
            .expectError(IllegalArgumentException.class)
            .verify();
}

@Test
public void flux_onErrorContinue() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .onErrorContinue(RuntimeException.class,
                    (throwable, o) -> log.info("error at : {}, exception : {}", o, throwable.getMessage()))
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:3")
            .verifyComplete();
}
```  

`Mono` 예제는 스트림상 단일 아이템만 존재하기 때문에 `onErrorContinue()` 에 로깅 처리를 해주었지만, 
이후 이어 진행할 스트림이 없기 때문에 `Error Handling` 이 정상적으로 수행되지 않고 `upstream` 에서 발생한 
`IllegalArgumentException` 이 그대로 `downstream` 까지 전달된 것을 확인 할 수 있다.  

반면 `Flux` 의 경우 두 번째 아이템인 `2` 에서 에러가 발생하므로 해당 경우는 `onErrorContinue()` 에 정의한 로깅이 수행되고, 
결과는 `1, 3` 의 결과가 정상적으로 전달된 것을 확인 할 수 있다.  


## Retry Strategy
로직 수행이 외부 네트워크를 사용하는 경우에는 네트워크 상태에 따라 `Reactive Stream` 에서 `Error` 가 발생할 수 있다. 
이는 장비의 네트워크에 문제가 있거나 타겟이 되는 서버의 네트워크에 문제가 있을 수도 있고, 
타겟 서버가 배포 중이거나 하는 등의 클라이언트 측에서는 예측 불가능한 경우가 무수히 많다.  

일반적으로 이런 경우 앞서 알아본 `Error Handling` 처럼 로그를 남기거나, 기본값 처리를 하는 등의 처리도 방법이 될 수 있다. 
하지만 좀 더 추전 되는 방안은 재시도 동작이다. 
네트워크 이슈처럼 원인이 불명확한 상태인 경우에는 재시도 동작으로 에러가 발생하는 경우를 줄여 볼 수 있다. 
하지만 재시도 동작이 과하게 설정된다면, 타겟 서버에 큰 부하를 줄 수 있으므로 적절한 수치값 지정이 필요하다.  

### retry
`retry` 는 재시도를 수행할 횟수를 파라미터로 넘겨주면 `upstream` 에서 
`Error` 가 발생하면 `upstream` 을 재수도 횟수만큼 다시 구독한다. 
만약 `retry()` 와 같이 파라미터에 아무것도 넘겨 주지 않으면 무한정 재구독이 수행 될 수 있으므로 주의가 필요하다.  

reactor-error-handling-retry-1.svg

```java
@Test
public void mono_retry() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .retry(2)
            .as(StepVerifier::create)
            .expectError(IllegalArgumentException.class)
            .verify();
}
/**
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */

@Test
public void flux_retry() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .retry(2)
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:1")
            .expectNext("result:1")
            .expectError(IllegalArgumentException.class)
            .verify();
}
/**
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */
```  

`Mono` 의 경우 `source stream` 이 에러가 발생되는 값만 방출하기 때문에 
초기 구독과 재구독를 포함해서 총 3번의 `getResult()` 메소드 호출이 수행된 것을 확인 할 수 있다. 
그리고 2번의 재시도 이후에도 실패하기 때문에 최종적으로 `IllelgalArgumentException` 이 발생했다.  

`Flux` 는 `source stream` 이 `1, 2, 3` 을 방출하므로 `2` 에서 에러가 발생한다. 
그러므로 총 3번의 `1` 에 대한 결과가 수행되고 `2` 로 인한 3번의 에러가 발생하게 된다.  
재시도 이후에도 에러는 동일하게 발생하는 코드로 돼 있어서 최종적으로 예외 결과가 나온 것을 확인 할 수 있다.  

### retryWhen(Retry.max)
`retryWhen` 은 [Retry](https://projectreactor.io/docs/core/3.5.4/api/reactor/util/retry/Retry.html) 
를 통해 재시작에 대한 조건을 다양하게 주고 싶을 때 `RetrySpec` 을 통해 원하는 조건으로 재시도를 가능하게 할 수 있다.  

reactor-error-handling-retry-2.svg

`RetrySpec` 의 다양한 조건 중 먼저 얼아 볼 것은 `Retry.max` 이다. 
`Retry.max` 는 앞서 알아본 `retry(maxAttempts)` 와 동일한 기능으로, 
재시도를 수행할 최대 횟수를 설정할 수 있다. 
`retry()` 와 치아점이 있다면, 재시도에도 성공하지 못했을 때 발생하는 예외가 
`retry()` 는 `upstream` 에서 발생한 예외롤 그대로 던진다면, 
`retryWhen()` 은 아래와 같이 `RetryExhaustedException` 에 감싸진 예외가 
어떤 재시도 조건에 실패해서 발생한 예외인지 설명과 함께 발생한다. 

```
Suppressed: reactor.core.Exceptions$RetryExhaustedException: Retries exhausted: 2/2
    at reactor.core.Exceptions.retryExhausted(Exceptions.java:290)
    at reactor.util.retry.RetrySpec.lambda$static$2(RetrySpec.java:61)
    at reactor.util.retry.RetrySpec.lambda$generateCompanion$5(RetrySpec.java:369)
    at reactor.core.publisher.FluxConcatMap$ConcatMapImmediate.drain(FluxConcatMap.java:374)
    ... 85 more
Caused by: java.lang.IllegalArgumentException: test exception
    at com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest.getResult(ReactorErrorRetryTest.java:23)
    at com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest.lambda$getMonoResult$0(ReactorErrorRetryTest.java:31)
    ... 85 more
```  

reactor-error-handling-retry-3.svg

```java
@Test
public void mono_retryWhen_max() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .retryWhen(Retry.max(2))
            .as(StepVerifier::create)
            .expectErrorMessage("Retries exhausted: 2/2")
            .verify();
}

/**
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */


@Test
public void flux_retryWhen_max() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .retryWhen(Retry.max(2))
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:1")
            .expectNext("result:1")
            .expectErrorMessage("Retries exhausted: 2/2")
            .verify();
}
/**
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */
```  

`Mono` 의 경우 `source stream` 이 에러가 발생되는 값만 방출하기 때문에
초기 구독과 재구독를 포함해서 총 3번의 `getResult()` 메소드 호출이 수행된 것을 확인 할 수 있다.
그리고 2번의 재시도 이후에도 실패하기 때문에 최종적으로 `Retries exhausted: 2/2` 라는 재시도 횟수 초과라는 메세지가 발생한다. 

`Flux` 는 `source stream` 이 `1, 2, 3` 을 방출하므로 `2` 에서 에러가 발생한다.
그러므로 총 3번의 `1` 에 대한 결과가 수행되고 `2` 로 인한 3번의 에러가 발생하게 된다.  
재시도 이후에도 에러는 동일하게 발생하는 코드로 돼 있어서 `Retries exhausted: 2/2` 라는 재시도 횟수 초과라는 메시지가 발생한다.  


### retryWhen(Retry.fixedDelay)
`retryWhen` 에 사용할 수 있는 `Retry.fixedDelay` 는 정해진 시간 동안 `delay` 를 
가진 뒤 재시도 횟수만큼 재시도를 수행하는 조건이다.  

reactor-error-handling-retry-4.svg

```java
@Test
public void mono_retryWhen_fixedDelay() {
    Mono.just("2")
            .flatMap(this::getMonoResult)
            .retryWhen(Retry.fixedDelay(2, Duration.ofSeconds(1)))
            .as(StepVerifier::create)
            .expectErrorMessage("Retries exhausted: 2/2")
            .verify();
}
/**
 * 18:45:56.715 [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * 18:45:57.730 [parallel-1] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * 18:45:58.735 [parallel-2] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */

@Test
public void flux_retryWhen_fixedDelay() {
    Mono.just(List.of("1", "2", "3"))
            .flatMapMany(Flux::fromIterable)
            .flatMap(this::getMonoResult)
            .retryWhen(Retry.fixedDelay(2, Duration.ofSeconds(1)))
            .as(StepVerifier::create)
            .expectNext("result:1")
            .expectNext("result:1")
            .expectNext("result:1")
            .expectErrorMessage("Retries exhausted: 2/2")
            .verify();
}
/**
 * 18:46:15.216 [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * 18:46:15.219 [main] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * 18:46:16.237 [parallel-1] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * 18:46:16.238 [parallel-1] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 * 18:46:17.243 [parallel-2] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 1
 * 18:46:17.243 [parallel-2] INFO com.windowforsun.reactor.errorhanding.ReactorErrorRetryTest - execute getResult : 2
 */
```  
