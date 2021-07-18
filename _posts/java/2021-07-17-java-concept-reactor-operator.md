--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 와 특징에 대해 알아보자'
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
`Operator` 의 종류를 역할별로 나누면 아래와 같다. 
- 생성 `Creating a New Sequence`
- 변경 `Transforming an Existing Sequence`
- 엿보기 `Peeking into a Sequence`
- 필터링 `Filtering a Sequence`
- 에러 처리 `Handling Errors`
- 시간 작업 `Working with Time`
- `Flux` 나누기 `Splitting a Flux`
- 동기로 변경 `Going Back to the Synchronous World`
- `Flux` 멀티캐스킹 `Multicasting a Flux to several Subscribers`

이렇게 다양한 `Operator` 는 내부적으로 시퀀스를 구독(`Subscripber`)하고, 생산(`Publisher`)하는 동작을 적절히 활용해서 구현되어 있다. 
수행하는 역할은 동일할지라도 내부적으로 구독, 생산을 어떻게 하는지 시점이 어떻게 되는지에 따라 내부적으로 동작이 상이할 수 있고, 
결과에도 차이가 있을 수 있다.  

이후 설명에서는 전체 `Operator` 에 대해서 모두 다루지는 못하고, 
몇개에 대해서만 예제를 진행한다. 
`Operator` 에 대해서 궁금한 점이 있다면 아래 링크에서 마블 다이어그램과 설명을 통해 보다 자세한 설명을 참고할 수 있다.  
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

#### defer(), fromSupplier()
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

### Transforming an Existing Sequence(시퀀스 변경)

구분|메소드|타입|설명
---|---|---|---
기존 데이터 변형|map|Mono, Flux|시퀀스내 아이템 하나씩 변환
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

#### cast(), ofType()
두 메소드 모드 시퀀스 아이템의 타입을 변환하는 용도로 사용할 수 있다. 
차이점은 아래와 같다. 
- `cast()` : 타입 캐스팅이 실패하면 `ClassCastException` 예외 발생
- `ofTyoe()` : 타입 캐스팅 실패한 요소는 스킵

```java
@Test
public void flux_cast() {
	// given
	Flux<Object> source_1 = Flux.just(1, 2, 3);
	Flux<Integer> source = source_1.cast(Integer.class);

	StepVerifier
			.create(source)
			.expectNext(1, 2, 3)
			.verifyComplete();
}

@Test
public void flux_cast_error() {
	// given
	Flux<Object> source_1 = Flux.just(1, "2", 3);
	Flux<Integer> source = source_1.cast(Integer.class);

	StepVerifier
			.create(source)
			.expectNext(1)
			.expectError(ClassCastException.class)
			.verify();
}

@Test
public void flux_ofType() {
	// given
	Flux<Object> source_1 = Flux.just(1, 2, 3);
	Flux<Integer> source = source_1.ofType(Integer.class);

	StepVerifier
			.create(source)
			.expectNext(1, 2, 3)
			.verifyComplete();
}

@Test
public void flux_ofType_error() {
	// given
	Flux<Object> source_1 = Flux.just(1, "2", 3);
	Flux<Integer> source = source_1.ofType(Integer.class);

	StepVerifier
			.create(source)
			.expectNext(1)
			.expectNext(3)
			.verifyComplete();
}
```  

#### handle()
시퀀스 요소를 바탕으로 시그널을 사용해 새로운 시퀀스로 변환 할 수 있다. 

```java
@Test
public void flux_handle_complete() {
	// given
	Flux<String> source_1 = Flux.just("val-1", "val-2", "complete", "dummy");
	Flux<String> source = source_1.handle((s, sink) -> {
		if (s.equals("complete")) {
			sink.complete();
		} else {
			sink.next(s);
		}
	});

	StepVerifier
			.create(source)
			.expectNext("val-1")
			.expectNext("val-2")
			.verifyComplete();
}

@Test
public void flux_handle_error() {
	// given
	Flux<String> source_1 = Flux.just("val-1", "val-2", "error", "dummy");
	Flux<String> source = source_1.handle((s, sink) -> {
		if (s.equals("error")) {
			sink.error(new Exception("my exception"));
		} else {
			sink.next(s);
		}
	});

	StepVerifier
			.create(source)
			.expectNext("val-1")
			.expectNext("val-2")
			.verifyErrorMessage("my exception");
}
```  

#### index()
시퀀스 요소들에 `Tuple` 타입으로 순서대로 색인을 추가할 수 있다.  

```java
@Test
public void flux_index() {
	// given
	Flux<Tuple2<Long, String>> source = Flux
			.just("a", "b", "c")
			.index((aLong, s) -> Tuples.of(aLong, s));

	StepVerifier
			.create(source)
			.expectNext(Tuples.of(0L, "a"))
			.expectNext(Tuples.of(1L, "b"))
			.expectNext(Tuples.of(2L, "c"))
			.verifyComplete();
}
```  

#### merge(), concat()
두 메소드 모두 `N` 개의 시퀀스를 하나의 시퀀스로 합치는 역할을 수행하는 차이점은 아래와 같다. 
- `concat()` : `N` 개의 시퀀스를 1개의 구독으로 차례대로 구독해서 하나의 시퀀스를 구성(시퀀스 순서에 따름)
- `merge()` : `N` 개의 시퀀스를 합치기 위해 `N` 개의 구독을 함께 수행하고 방출되는 순서대로 하나의 시퀀스를 구성(방출된 순서에 따름)

```java
@Test
public void flux_concat() {
	// given
	Flux<String> source_1 = Flux
			.just("1", "2", "3")
			.delayElements(Duration.ofMillis(10));
	Flux<String> source_2 = Flux
			.just("a", "b", "c")
			.delayElements(Duration.ofMillis(2));
	Flux<String> source = Flux.concat(source_1, source_2);

	StepVerifier
			.create(source)
			.expectNext("1", "2", "3")
			.expectNext("a", "b", "c")
			.verifyComplete();
}

@Test
public void flux_merge() throws Exception {
	// given
	Flux<String> source_1 = Flux
			.just("1", "2", "3")
			.delayElements(Duration.ofMillis(10));
	Flux<String> source_2 = Flux
			.just("a", "b", "c")
			.delayElements(Duration.ofMillis(2));
	Flux<String> source = Flux.merge(source_1, source_2);
	List<String> recordItems = new ArrayList<>();

	StepVerifier
			.create(source)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.consumeRecordedWith(actual -> {
				assertThat(actual, not(contains("1", "2", "3", "a", "b", "c")));
				assertThat(actual, hasSize(6));
				assertThat(actual, everyItem(oneOf("1", "2", "3", "a", "b", "c")));
			})
			.verifyComplete();
}
```  

#### concatMap(), flatMap()
시퀀스 아이템을 바탕으로 시퀀스를 반환하면 이를 하나의 시퀀스로 구성하는 역할은 동일 하지만 차이점은 아래와 같다. 

> `conat()`, `merge()` 의 차이와 비슷하다. 

- `concatMap()` : 시퀀스 아이템에서 반환하는 시퀀스를 순서대로 구독하면서 하나의 시퀀스로 구성(리턴되는 시퀀스 순서에 따름)
- `flatMap()` : 시퀀스 아이템에서 반환하는 시퀀스를 `N`개 구독으로 구독하면서 하나의 시퀀스로 구성(리턴되 시퀀스에서 방출하는 순서에 따름)

```java
@Test
public void flux_concatMap() {
	// given
	Flux<Integer> source = Flux
			.just(1, 2, 3)
			.concatMap(integer -> {
				Stream stream = IntStream
						.iterate(integer, n -> n)
						.limit(integer)
						.boxed();

				if (integer % 2 == 0) {
					return Flux.fromStream(stream).delayElements(Duration.ofMillis(10));
				} else {
					return Flux.fromStream(stream).delayElements(Duration.ofMillis(2));
				}
			});

	StepVerifier
			.create(source)
			.expectNext(1)
			.expectNext(2, 2)
			.expectNext(3, 3, 3)
			.verifyComplete();
}

@Test
public void flux_flatMap() {
	// given
	Flux<Integer> source = Flux
			.just(1, 2, 3)
			.flatMap(integer -> {
				Stream stream = IntStream
						.iterate(integer, n -> n)
						.limit(integer)
						.boxed();

				if (integer % 2 == 0) {
					return Flux.fromStream(stream).delayElements(Duration.ofMillis(10));
				} else {
					return Flux.fromStream(stream).delayElements(Duration.ofMillis(2));
				}
			});
	ArrayList<Integer> recordItems = new ArrayList<>();

	StepVerifier
			.create(source)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.recordWith(() -> recordItems)
			.consumeRecordedWith(actual -> {
				assertThat(actual, not(contains(1, 2, 2, 3, 3, 3)));
				assertThat(actual, hasSize(6));
				assertThat(actual, everyItem(oneOf(1, 2, 3)));
			})
			.verifyComplete();
}
```  

#### reduce(), scan()
시퀀스 아이템을 조합하는 동작을 수행하는 것은 동일하지만 차이점은 아래와 같다. 
- `reduce()` : 시퀀스 아이템의 최종 조합 결과 값만 리턴(`Mono<T>`)
- `scan()` : 시퀀스 아이템이 조합되는 순서대로 `N` 개 값 리턴(`Flux<t>`)

```java
@Test
public void flux_reduce() {
	// given
	Mono<String> source = Flux
			.just("1", "2", "3")
			.reduce((s, s2) -> s.concat(s2));

	StepVerifier
			.create(source)
			.expectNext("123")
			.verifyComplete();
}

@Test
public void flux_scan() {
	// given
	Flux<String> source = Flux
			.just("1", "2", "3")
			.scan((s, s2) -> s.concat(s2));

	StepVerifier
			.create(source)
			.expectNext("1", "12", "123")
			.verifyComplete();
}
```  

#### zip()
`N` 개의 시퀀스를 하나의 시퀀스로 합치는데 
`concat()`, `merge()` 처럼 아이템 수를 늘리며 합치는게 아니라 
각 아이템을 조합하는 방식으로 하나의 시퀀스를 구성한다. 
최대 8개 까지 한번에 조합 할 수 있다.  

```java
@Test
public void flux_zip_Tuple() {
	// given
	Flux<String> source_1 = Flux.just("1", "2", "3");
	Flux<String> source_2 = Flux.just("first", "second", "third");
	Flux<Tuple2<String, String>> source = Flux.zip(source_1, source_2);

	StepVerifier
			.create(source)
			.expectNext(Tuples.of("1", "first"))
			.expectNext(Tuples.of("2", "second"))
			.expectNext(Tuples.of("3", "third"))
			.verifyComplete();
}

@Test
public void flux_zip_BiFunction() {
	// given
	Flux<String> source_1 = Flux.just("1", "2", "3");
	Flux<String> source_2 = Flux.just("first", "second", "third");
	Flux<String> source = Flux.zip(source_1, source_2, (s, s2) -> s + "-" + s2);

	StepVerifier
			.create(source)
			.expectNext("1-first", "2-second", "3-third")
			.verifyComplete();
}

@Test
public void flux_zip_Iterable() {
	// given
	Flux<String> source_1 = Flux.just("1", "2");
	Flux<String> source_2 = Flux.just("a", "b");
	Flux<String> source_3 = Flux.just("first", "second");
	Iterable<Flux<String>> iter = Stream.of(source_1, source_2, source_3).collect(Collectors.toList());
	Flux<String> source = Flux.zip(iter, objects -> {
		StringBuilder builder = new StringBuilder();

		for (Object obj : objects) {
			builder
					.append("-")
					.append(obj);
		}

		return builder.toString();
	});

	StepVerifier
			.create(source)
			.expectNext("-1-a-first")
			.expectNext("-2-b-second")
			.verifyComplete();
}

@Test
public void flux_zip_Iterable_prefetch() {
	// given
	Flux<String> source_1 = Flux.just("1", "2").log();
	Flux<String> source_2 = Flux.just("a", "b").log();
	Flux<String> source_3 = Flux.just("first", "second").log();
	Iterable<Flux<String>> iter = Stream.of(source_1, source_2, source_3).collect(Collectors.toList());
	Flux<String> source = Flux.zip(iter, 2, objects -> {
		StringBuilder builder = new StringBuilder();

		for (Object obj : objects) {
			builder
					.append("-")
					.append(obj);
		}

		return builder.toString();
	});

	StepVerifier
			.create(source)
			.expectNext("-1-a-first")
			.expectNext("-2-b-second")
			.verifyComplete();
}
```  

#### then(), thenEmpty()
시퀀스를 바로 완료시킨다. 에러는 그대로 전달 된다.  

```java
@Test
public void flux_then() {
	// given
	Mono<Void> source = Flux.just("a", "b").then();

	StepVerifier
			.create(source)
			.expectComplete()
			.verify();
}

@Test
public void flux_then_error() {
	// given
	Mono<Void> source = Flux
			.merge(Flux.just("a", "b"), Flux.error(new Exception("my exception")))
			.then();

	StepVerifier
			.create(source)
			.expectErrorMessage("my exception")
			.verify();
}

@Test
public void flux_then_2() {
	// given
	Mono<String> source = Flux
			.just("a", "b")
			.then(Mono.just("mono val"));

	StepVerifier
			.create(source)
			.expectNext("mono val")
			.verifyComplete();
}

@Test
public void flux_then_2_error() {
	// given
	Mono<String> source = Flux
			.merge(Flux.just("a", "b"), Flux.error(new Exception("my exception")))
			.then(Mono.just("mono val"));

	StepVerifier
			.create(source)
			.expectErrorMessage("my exception")
			.verify();
}

@Test
public void flux_thenEmpty() {
	// given
	Mono<Void> source = Flux.just("a", "b").thenEmpty(Mono.empty());

	StepVerifier
			.create(source)
			.verifyComplete();
}

@Test
public void flux_thenEmpty_error() {
	// given
	Mono<Void> source = Flux
			.merge(Flux.just("a", "b"), Flux.error(new Exception("my exception")))
			.thenEmpty(Mono.empty());

	StepVerifier
			.create(source)
			.expectErrorMessage("my exception")
			.verify();
}
```  

#### switchMap(), switchOnFirst(), switchOnNext()
이름에 `switch` 가 붙은 메소들은 모두 시퀀스를 전환하는 역할을 한다. 
- `switchMap()` : 시퀀스 요소들을 바탕으로 새로운 시퀀스로 전환 가능하고, 다음 요소가 방출되면 이전 요소 시퀀스는 종료된다. 
- `switchOnNext()` : 주어진 여러 시퀀스를 방출하는 시퀀스를 바탕으로 `switchMap` 동작을 수행한다.
- `switchOnFirst()` : 시퀀스의 첫 번째 요소를 바탕으로 방출될 시퀀스를 결정 할 수 있다. 

>`switchMap()`, `switchOnNext()` 의 동작에 대해 의문을 가질 수 있는데, 
> 검색 자동완성과 같은 경우를 생걱하면 쉽다. 
> `가`를 검색어로 입력해서 자동완성 결과가 완료되기 전에 `가나` 가 입력되면, 
> `가`에 대한 자동완성 결과를 필요없고 `가나`에 대한 자동완성 결과만 있으면 된다. 
> 이런 상황에서 시퀀스를 새로운 구독 시점에 따라 전환 시킬 수 있다.  

```java
@Test
public void flux_switchMap() {
	// given
	Flux<String> source = Flux.just("1", "delay-2", "3", "delay-4", "delay-last")
			.switchMap(s -> {
				Flux<String> flux = Flux.just("switchMap-" + s);

				if (s.startsWith("delay")) {
					flux = flux.delayElements(Duration.ofMillis(100));
				}

				return flux;
			});

	StepVerifier
			.create(source)
			.expectNext("switchMap-1", "switchMap-3", "switchMap-delay-last")
			.verifyComplete();
}

@Test
public void flux_switchOnNext() {
	// given
	Flux<Flux<Integer>> source_1 = Flux
			.just(
					Flux.just(1, 2),
					Flux.just(3, 4).delayElements(Duration.ofMillis(200)),
					Flux.just(5, 6).delayElements(Duration.ofMillis(100))
			);
	Flux<Integer> source = Flux.switchOnNext(source_1);

	StepVerifier
			.create(source)
			.expectNext(1, 2, 5, 6)
			.verifyComplete();
}

@Test
public void flux_switchOnFirst_odd() {
	// given
	Flux<Integer> source_1 = Flux.just(1, 2, 3, 4, 5);
	Flux<Integer> source = source_1
			.switchOnFirst((signal, integerFlux) -> {
				if (signal.hasValue()) {
					int mod = signal.get() % 2;
					return integerFlux
							.filter(integer -> integer % 2 == mod);
				}

				return integerFlux;
			});

	StepVerifier
			.create(source)
			.expectNext(1, 3, 5)
			.verifyComplete();
}

@Test
public void flux_switchOnFirst_even() {
	// given
	Flux<Integer> source_1 = Flux.just(2, 3, 4, 5, 6);
	Flux<Integer> source = source_1
			.switchOnFirst((signal, integerFlux) -> {
				if (signal.hasValue()) {
					int mod = signal.get() % 2;
					return integerFlux
							.filter(integer -> integer % 2 == mod);
				}

				return integerFlux;
			});

	StepVerifier
			.create(source)
			.expectNext(2, 4, 6)
			.verifyComplete();
}
```  

#### expand(), expandDeep()
시퀀스 요소에 대해서 재귀적인 처리를 수행할 수 있다. 
재귀 방식은 `Graph` 탐색을 바탕으로 한다. 
- `expand()` : `BFS` 방식으로 너비우선방식으로 재귀동작이 수행된다. 
- `expandDeep()` : `DFS` 방식으로 깊이우선방식으로 재귀동작이 수행된다. 

```java
@Test
public void flux_expand() {
	// given
	Flux<String> source = Flux
			.just("/home", "/root")
			.expand(s -> {
				String[] split = s.split("/");

				if (split.length > 3) {
					return Mono.empty();
				} else {
					return Flux.just(s + "/dir1", s + "/dir2");
				}
			});

	StepVerifier
			.create(source)
			.expectNext("/home", "/root")
			.expectNext("/home/dir1", "/home/dir2")
			.expectNext("/root/dir1", "/root/dir2")
			.expectNext("/home/dir1/dir1", "/home/dir1/dir2", "/home/dir2/dir1", "/home/dir2/dir2")
			.expectNext("/root/dir1/dir1", "/root/dir1/dir2", "/root/dir2/dir1", "/root/dir2/dir2")
			.verifyComplete();
}

@Test
public void flux_expandDeep() {
	// given
	Flux<String> source = Flux
			.just("/home", "/root")
			.expandDeep(s -> {
				String[] split = s.split("/");

				if (split.length > 3) {
					return Mono.empty();
				} else {
					return Flux.just(s + "/dir1", s + "/dir2");
				}
			})
			;

	StepVerifier
			.create(source)
			.expectNext("/home")
			.expectNext("/home/dir1", "/home/dir1/dir1", "/home/dir1/dir2")
			.expectNext("/home/dir2", "/home/dir2/dir1", "/home/dir2/dir2")
			.expectNext("/root")
			.expectNext("/root/dir1", "/root/dir1/dir1", "/root/dir1/dir2")
			.expectNext("/root/dir2", "/root/dir2/dir1", "/root/dir2/dir2")
			.verifyComplete();
}
```  

### Peeking into a Sequence(시퀀스 엿보기)








---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


