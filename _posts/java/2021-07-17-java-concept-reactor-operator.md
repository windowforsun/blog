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
- 동기로 변경 `Going Back to the Synchronous World`
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

#### log() 
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

#### do*() 
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

#### materialize(), dematerialize()

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


#### distinct(), distinctUntilChanged() 

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

#### takeUntil(), takeUntilOther(), skipUntil(), skipUntilOther()

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

#### sample(), sampleTimeout()
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


#### retry(), retryWhen()
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

#### onBackPressure*()
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


### Working with Time(시간 작업)

구분|메소드|타입|설명
---|---|---|---
측정 시간 함께 방출|timed|Mono,Flux|시퀀스에서 요소를 방출할때 측정시간과 함께 방출
방출 지연 타임아웃 설정|timeout|Mono,Flux|방출 지연시간 제한을 설정 및 타임아웃시 `Fallback` 처리
일정한 간격으로 시간 값 방출|interval|Flux|정해진 지연시간마다 `Long` 값을 0부터 차례대로 방출
방출 지연시간 추가|delayElements|Mono,Flux|주어진 시퀀스의 요소 방출 지연시간 설정
 |delaySubscription|Mono,Flux|주어진 시퀀스에 `subscribe()` 이벤트까지 지연시간 설정

#### timed()
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

#### timeout()
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


