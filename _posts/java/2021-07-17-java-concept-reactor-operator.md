--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 의 종류와 특징, 사용방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
toc: true 
use_math: true
---  

## Reactor Operator
[Reactive Streams 활용](https://windowforsun.github.io/blog/java/java-concept-ractivestreams-advanced/#operator) 
에서 설명한것 처럼 `Operator` 는 `Reactive Streams` 흐름에 어떠한 동작을 추가하는 것을 의미한다. 
`Reactor` 에서는 아주 많은 `Operator` 제공을 해주기 때문에 이를 활용해서, 
필터링, 예외처리, 데이터 조작 등 다양한 동작을 수행할 수 있다. 
`Operator` 를 역할에 따라 잘 활용하면 `Reactive Streams` 의 시퀀스 내에서 
애플리케이션에서 수행하고자 하는 대부분의 동작을 구성할 수 있을 것이다. 
`Operator` 의 종류를 역할별로 나누면 아래와 같다.  
- 생성 `Creating a New Sequence`
- 변경 `Transforming an Existing Sequence`
- 엿보기 `Peeking into a Sequence`
- 필터링 `Filtering a Sequence`
- 에러 처리 `Handling Errors`
- 시간 작업 `Working with Time`
- `Flux` 나누기 `Splitting a Flux`
- 비동기에서 동기로 변경 `Going Back to the Synchronous World`
- `Flux` 멀티캐스킹 `Multicasting a Flux to several Subscribers`

이렇게 다양한 `Operator` 는 내부적으로 시퀀스를 구독(`Subscripber`)하고, 생산(`Publisher`)하는 동작을 적절히 활용해서 구현되어 있다. 
수행하는 역할은 동일할지라도 내부적으로 구독, 생산을 어떻게 하는지 시점이 어떻게 되는지에 따라 내부적으로 동작이 상이할 수 있고, 
결과에도 차이가 있을 수 있다.  

이후 설명에서는 전체 `Operator` 에 대해서 모두 다루지는 못하고, 
몇개에 대해서만 예제를 진행한다. 
`Operator` 에 대해서 궁금한 점이 있다면 아래 링크에서 마블 다이어그램과 설명을 통해 보다 자세한 설명을 참고할 수 있다. 
마블 다이어그램은 실제 메소드의 특징이나 전체적인 흐름을 구체적으로 한번에 확인할 수 있어서, 효율적으로 메소드에 대해 이해할 수 있다.  

- [Mono](https://projectreactor.io/docs/core/3.4.6/api/index.html?reactor/core/publisher/Mono.html)
- [Flux](https://projectreactor.io/docs/core/3.4.6/api/index.html?reactor/core/publisher/Flux.html)

### Creating a New Sequence(시퀀스 생성)

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

[Reactor Create Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-create-operator)


### Transforming an Existing Sequence(시퀀스 변경)

구분|메소드|타입|설명
---|---|---|---
기존 데이터 변형|map|Mono,Flux|시퀀스내 아이템 하나씩 변환
 |cast|Mono,Flux|시퀀스 아이템 하나씩 타입 변환(에러)
 |ofType|Mono,Flux|시퀀스 아이템 하나씩 타입 변환(스킵)
 |index|Mono,Flux|시퀀스내 아이템 하나씩 색인 추가
 |concatMap|Mono,Flux|시퀀스 아이템을 시퀀스로 반환하면 이를 하나의 시퀀스로 합침(순서 보장)
 |flatMap|Mono,Flux|시퀀스 아이템을 시퀀스로 반환하면 이를 하나의 시퀀스로 합침
 |handle|Mono,Flux|시퀀스를 시그널을 사용해서 변경
 |flatMapSequential|Flux|시퀀스 아이템을 시퀀스로 반환하면 이를 하나의 시퀀스로 합침(순서 보장)
 |flatMapMany|Mono|`flatMap` 동작에서 `Mono` 를 `Flux` 로 전환
기존 시퀀스에 데이터 추가|startWith|Flux|시퀀스 시작 부분에 데이터 추가
 |concatWith|Flux|시퀀스 끝 부분에 데이터 추가
`Flux` 하나로(`Mono`) 합치기|collectList|Flux|`Flux` 의 데이터를 `Mono<List>` 로 변환
 |collectSortedList|Flux|`Flux` 의 데이터를 정렬된 `Mono<List>` 로 변환
 |collectMap|Flux|`Flux` 의 데이터를 `Mono<Map<K,V>` 로 변환
 |collectMultiMap|Flux|`Flux` 의 데이터를 `Mono<Map<K,Collection>` 로 변환
 |collect|Flux|`Flux` 의 데이터를 사용자가 정의한 `Mono<V>` 로 변환
 |count|Flux|`Flux` 의 데이터 수 반환
 |reduce|Flux|`Flux` 전체 시퀀스에 `Function` 적용(합계)
 |scan|Flux|`Flux` 시퀀스 데이터 하나 마다 `Function` 적용(합계)
 |all|Flux|`Flux` 전체 시퀀스에 대해 `AND` 조건 결과
 |any|Flux|`Flux` 전체 시퀀스에 대해 `OR` 조건 결과
 |hasElements|Flux|`Flux` 전체 시퀀스에 데이터 여부
 |hasElement|Flux|`Flux` 전체 시퀀스에 해당 값 존재 여부
`Publisher` 결합|concat(concatWith)|Flux|순서대로 인자값의 시퀀스를 하나로 합침
 |concatDelayError|Flux|`concat` 동작에서 에러 지연 방출
 |merge(mergeWith)|Flux|인자값의 시퀀스의 **방출** 순서대로 시퀀스를 하나로 합침 
 |mergeSequential|Flux|`merge` 동작에서 방출 순서가 아닌 `concat` 처럼 기존 순서
 |zip(zipWith)|Mono,Flux|인자값의 여러 시퀀스를 하나씩의 데이터 쌍(`Tuple`)으로 합침
 |and|Mono|인자 값과 하나의 시퀀스로 합치고 완료 처리 `Mono<Void>` 리턴
 |when|Mono|여러 인자 값과 하나의 시퀀스로 합치고 완료 처리 `Mono<Void>` 리턴
 |combineLatest|Flux|각 시퀀스 소스에서 가장 최근에 방출된 값으로 데이터 조합
 |firstWithSignal|Mono,Flux|여러 시퀀스에서 가장 처음으로(빨리) 방출된 시퀀스 리턴
 |or|Mono,Flux|기존 시퀀스와 인자값 중 가장 처음으로(빨리) 방출된 시퀀스 리턴
 |switchMap|Flux|`flatMap` 비슷하지만 시퀀스 요소에 의해 새로운 시퀀스로 전환(구독 시점에 따라 무시)
 |switchOnNext|Flux|`flatMap` 과 비슷하지만 주어진 여러 시퀀스를 바탕으로 새로운 시퀀스로 전환(구독 시점에 따라 무시)
 |switchOnFirst|Flux|시퀀스의 첫 번째 요소를 바탕으로 시퀀스 전환
기존 시퀀스 반복|repeat|Flux,Mono|시퀀스를 반복해서 구독
 |interval|Flux|특정 주기에 따라 증가값 방출 하는 시퀀스 생성
비어있는 시퀀스|defaultIfEmpty|Flux,Mono|시퀀스 비어있는 경우 기본값이 필요 할때
 |switchIfEmpty|Flux,Mono|시퀀스 비어있는 경우 다른 시퀀스로 전환 할때
시퀀스 값에 관심 없을 때|ignoreElements|Flux,Mono|시퀀스에서 값은 무시하고 완료 시그널만 필요할 때
 |then|Flux,Mono|시퀀스를 그냥 완료 시키거나 다른 시퀀스가 필요할 때 
 |thenEmpty|Flux,Mono|시퀀스를 그냥 완료 시키고 다른 작업을 수행
 |thenMany|Flux,Mono|시퀀스를 그냥 완료 시키고 다른 시퀀스로 데이터 방출
 |thenReturn|Mono|시퀀스를 그냥 완료 시키고 주어진 데이터 방출
`Mono` 시퀀스 완료 연기 처리|delayUntil|Mono|주어진 시퀀스가 완료될때까지 기존 시퀀스 완료 지연
시퀀스 값에 대해 재귀처리|expand|Mono,Flux|시퀀스 값에 대해 재귀 처리(너비 우선)
 |expandDeep|Mono,Flux|시퀀스 값에 대해 재귀 처리(깊이 우선)

[Reactor Transforming Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-transforming-operator)


### Peeking into a Sequence(시퀀스 엿보기)

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


### Filtering a Sequence(필터링)

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

[Reactor Filtering Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-filtering-operator)


### Handling Errors(에러 처리)

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

[Reactor Error Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-error-operator)


### Working with Time(시간 작업)

구분|메소드|타입|설명
---|---|---|---
측정 시간 함께 방출|timed|Mono,Flux|시퀀스에서 요소를 방출할때 측정시간과 함께 방출
방출 지연 타임아웃 설정|timeout|Mono,Flux|방출 지연시간 제한을 설정 및 타임아웃시 `Fallback` 처리
일정한 간격으로 시간 값 방출|interval|Flux|정해진 지연시간마다 `Long` 값을 0부터 차례대로 방출
방출 지연시간 추가|delayElements|Mono,Flux|주어진 시퀀스의 요소 방출 지연시간 설정
 |delaySubscription|Mono,Flux|주어진 시퀀스에 `subscribe()` 이벤트까지 지연시간 설정

[Reactor Time Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-time-operator)


### Splitting a Flux(`Flux` 나누기)

구분|메소드|타입|설명
---|---|---|---
`Flux<T>`를 `Flux<Flux<T>>` 로 분리|window|Flux|방출되는 요소 크기 혹은 시간을 기준으로 분리
 |windowTimeout|Flux|방출되는 요소 크기와 시간을 기준으로 분리
 |windowUntil|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소 다음 부터 분리
 |windowWhile|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소는 제외하고 다음 부터 분리
 |windowWhen|Flux|`Publisher` 의 신호의 경계 값을 기준으로 분리
`Flux<T>`를 `Flux<List<T>` 로 분리|buffer|Flux|방출되는 요소 크기 혹은 시간을 기준으로 분리
 |bufferTimeout|Flux|방출되는 요소 크기와 시간을 기준으로 분리
 |bufferUntil|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소 다음 부터 분리
 |bufferWhile|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소는 제외하고 다음 부터 분리
 |bufferWhen|Flux|`Publisher` 의 신호의 경계 값을 기준으로 분리
`Flux<T>`를 `Flux<GroupedFlux<K, T>>` 로 같은 특정을 가진 것들로 분리|groupBy|Flux|`Function` 로직에서 같은 특정을 갖는 요소들 끼리 그룹지어 분리

[Reactor Splitting Operator 추가 설명](https://windowforsun.github.io/blog/java/java-concept-reactor-splitting-operator)


### Going Back to the Synchronous World(비동기에서 동기로 변경)


>`Non-Blocking Only` 로 표시된 `Subscriber` 에서 동기로 변환 메소드를 호출하면 `UnsupportedOperatorException` 예외를 던진다. 



|구분|메소드|타입|설명
|---|---|---|---
|`Flux<T>` 에 대한 블로킹|blockFirst|Flux|첫번째 요소를 얻을 때까지 블로킹(타입아웃 설정 가능)
| |blockLast|Flux|마지막 요소를 얻을 때까지 블로킹(타입아웃 설정 가능)
| |toIterable|Flux|시퀀스를 `Iterable<T>` 로 전환할때 까지 블로킹
| |toStream|Flux|시퀀스를 `Stream<T>` 로 전환할때 까지 블로킹
|Mono<T> 에 대한 블로킹|block|Mono|요소를 얻을 때까지 블로킹(타임아웃 설정 가능)
| |toFuture|Mono|시퀀스를 `CompletableFuture<T>` 로 전환 할떄까지 블로킹


### Multicasting a Flux to several Subscribers(`Flux` 멀티캐스킹)

구분|메소드|타입|설명
---|---|---|---
시퀀스 하나에 여러 `Subscriber` 연결|publish().connect()|Flux|리턴된 `ConnectableFlux` 를 사용해서 여러 `Subscriber` 를 연결하고 `connect()` 호출
 |share|Mono,Flux|시퀀스가 현재 `Subscriber` 에 의해 요소을 방출 중일때 다른 `Subscriber` 가 구독하면 동일한 요소를 방출 
 |publish().autoConnect(n)|Flux|`publish.connect` 동작에서 `autoConnect` 에 설정한 숫자만큼 구독되면 자동으로 `connect()` 호출
 |publish().refCount(n)|Flux|`publish.autuConnect` 동작에서 `refCount` 에 설정된 숫자만큼 구독자가 발생하지 않으면 구독을 취소 
시퀀스의 요소를 캐싱해두고 모든 구독자에게 방출|cache|Mono,Flux|`cache` 에 설정한 갯수 혹은 `Duration` 동안 요소를 캐싱
 |replay|Flux|`replay` 에 설정한 갯수 혹은 `Duration` 동안 가장 최근에 구독된 시퀀스의 방출 및 시그널을 그대로 수행


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  
[사용하면서 알게 된 Reactor, 예제 코드로 살펴보기](https://tech.kakao.com/2018/05/29/reactor-programming/)  


