--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Cold Hot Sequence"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Reactor 의 시퀀스를 생성하는 방식 중 Cold, Hot 시퀀스에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Flux
  - Mono
  - Cold Sequence
  - Hot Sequence
toc: true
use_math: true
---  

## Hot / Cold Sequence
`Reactor` 에서는 시퀀스를 제공하는 `Mono`, `Flux` 의 2가지 타입을 제공한다. 
위 두가지 타입의 시퀀스를 생성할때 `Hot`, `Cold` 의 두가지 `Sequence`(`Publisher`)로 생성 방식을 나눌 수 있다.  
두가지 방식을 개념적으로 먼저 설명하면 아래와 같다. 
- `Hot Sequence` : 실제 구독(`Subscribe`) 전 시퀀스의 데이터 생성 및 동작이 수행되고, 한번 생성된 데이터는 구독시 계속해서 사용된다. 
  - 여러 구독자가 하나의 시퀀스의 데이터를 함께 구독해야하는 경우 `Hot Sequence` 방식으로 생성하는 것이 좋다. 
- `Cold Sequence` : 실제 구독(`Subscribe`) 전까지 시퀀스는 아무 동작을 수행하지 않고, 구독이 이뤄질 때 그때마다 시퀀스의 데이터생성 및 동작이 수행된다. 
  - 시퀀스가 매번 구독 때마다 가장 최신 혹은 초기화 과정까지 필요하다면 `Cold Sequence` 방식으로 생성하는 것이 좋다. 

>어떤 글에서는 `Cold Publisher`, `Hot Publisher` 라고 불리기도 한다.

테스트는 각 시퀀스에서 아래 메소드를 사용해 아이템을 생성한다고 가정하고 진행한다. 
아래 메소드는 인자값으로 로그에 출력할 문자열을 받고 리턴 값으로는 메소드가 호출된 밀리초를 리턴한다. 
그리고 해당 메소드는 호출 시마다 `100 millis` 정도 슬립을 수행한다.  

```java
public long doSome(String str) {
	long timestamp = 0L;

	try {
		Thread.sleep(100);
		timestamp = System.currentTimeMillis();
		log.info("doSome {}: {}", str, timestamp);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}

	return timestamp;
}
```  

### Cold Sequence
`Cold Sequence` 는 실제 구독이 이뤄지기 전까지는 아무 동작을 수행하지 않고 있다가, 
구독이 이뤄질때마다 새로운 데이터를 생산해 구독자에게 전달하는 시퀀스를 의미한다. 
이런 `Cold Sequence` 가 우리가 일반적으로 알고 있는 `Recative Streams` 의 시퀀스이다.  

음악 스트리밍 서비스를 예로 들어보자. 
아무도 애국가를 스트리밍 하고 있지 않다면 애국가는 어느 누구에게도 데이터를 전달하지 않는다. 
만약 애국가를 듣고 있는 2명의 사용자가 있다고 가정 하면, 
2명의 사용자가 듣는 음악의 구간은 서로 다르기 때문에 서로 다른 시퀀스 생성이 필요하다. 
이런 음악 스트리밍과 같은 상황에서 사용할 수 있는 것이바로 `Cold Sequence` 이다.  

`Cold Sequence` 를 생성하는 `Mono`, `Flux` 의 팩토리 메소드는 많지만 그 중에서 
이번 포스트는 `fromSupplier` 와 `fromStream` 을 사용해서 그 특징을 살펴 본다.    

먼저 `Mono` 의 `Cold Sequence` 는 아래와 같다.  

```java
@Test
public void mono_create_cold() throws Exception {
	// given
	Mono<Long> mono = Mono.fromSupplier(() -> this.doSome("mono"));
	long current = this.doSome("current");

	// when
	Long actual_1 = mono.block();
	Long actual_2 = mono.block();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_2, notNullValue());
	assertThat(actual_1, not(actual_2));
	assertThat(actual_1, greaterThan(current));
	assertThat(actual_2, allOf(greaterThan(current), greaterThan(actual_1)));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624736767100
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome mono: 1624736767211
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome mono: 1624736767323
```  

테스트 코드의 순서는 아래와 같다. 
1. `Mono` 시퀀스 생성(`doSome(mono)` 사용)
1. `doSome(current)` 호출 
1. 첫 번째 시퀀스 구독 `actual_1`
1. 두 번째 시퀀스 구독 `actual_2`

그리고 로그에 출력된 순서는 아래와 같고 로그 출력 값과 변수의 크기는 아래 순서의 역순이다. (`current` < `actual_1` < `actual_2`)
1. `doSome(current)`, `current`
1. 첫 번째 시퀀스 구독 `doSome(mono)`, `actual_1`
1. 두 번째 시퀀스 구독 `doSome(mono)`, `actual_2`

`Mono` 시퀀스를 `fromSupplier()` 팩토리 메소드를 사용해서 선언하고, `doSome(current)` 를 호출 했지만 `doSome(current)` 가 가장 먼저 찍힌 로그인 것을 보면 
시퀀스 선언시에 어떠한 동작도 수행되지 않았다. 
그리고 `actual_1`, `actual_2` 값이 서로 다른 것을 통해 구독 마다 새로운 데이터가 생성되었음을 알 수 있다.  

다음으로 `Flux` 의 `Cold Sequence` 예제는 아래와 같다.  

```java
@Test
public void flux_create_cold() throws Exception {
	// given
	Flux<Long> flux = Flux.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")));
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = flux.collectList().block();
	List<Long> actual_2 = flux.collectList().block();

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, not(actual_2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(allOf(
			greaterThan(current),
			greaterThan(actual_1.get(0)),
			greaterThan(actual_1.get(1))
	)));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624782802603
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624782802760
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624782802869
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624782802994
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624782803103
```  

먼저 살펴본 `Mono` 와 테스트 코드 실행 순서 및 결과를 비슷하다. 
`Flux` 시퀀스를 먼저 `fromStream()` 팩토리 메소드를 사용해서 선언하고 나서, 
`doSome(current)` 를 호출 했지만 `fromStream()` 이 실행된 시점에서는 어떠한 시퀀스 관련 동작도 이뤄지지 않았다. 
실제 시퀀스를 생성하는 시점은 첫 번째 구독이 이뤄지는 부분과 두 번째 구독이 이뤄지는 부분에서 
선언된 하나의 시퀀스에서 각 다른 시퀀스가 생성된 것을 확인 할 수 있다.  


### Hot Sequence
`Hot Sequence` 는 시퀀스를 선언하는 시점에 실제 시퀀스 생성 동작이 일어나게 되고, 
해당 동작으로 생성된 시퀀스에서 사용하는 아이템은 이후 구독하는 구독자들에게 공동으로 사용되는 방식이다.  

TV, 라디오 방송을 예로 들어보자. 
TV, 라디오는 실제 시청자가 자신이 송출하는 방송을 실제 구독 여부와 관계없이 계속해서 방송 송출은 이뤄진다. 
2명의 사용자가 있을 때, 
한 명은 방송 시작과 동시에 방송을 구독했고, 다른 한명은 방송이 시작하고 30분 후에 구독을 했다고 해보자. 
이때 2명 사용자가 모두 방송을 구독한 이후에는 2명 모두 30분 이후 데이터는 함께 수신하게 된다. 
즉 방송의 `Broadcast` 는 전체 사용자 모두 항상 같은 데이터를 수신하는 것이기 때문에, 
30분 후에 구독한 사용자관점에서는 30분 이전 데이터는 모두 유실되는 것과 같다.  

`Mono`, `Flux` 시퀀스를 생성하는 많은 팩토리 메소드 중에서 
`Hot Sequence` 방식으로 생성할 수 있는 `just()` 를 사용해서 특징을 알아본다.  

먼저 `Mono` 를 `Hot Sequence` 로 생성하는 예제는 아래와 같다.  

```java
@Test
public void mono_just_hot() throws Exception {
	// given
	Mono<Long> mono = Mono.just(this.doSome("mono"));
	long current = this.doSome("current");

	// when
	Long actual_1 = mono.block();
	Long actual_2 = mono.block();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_2, notNullValue());
	assertThat(actual_1, lessThan(current));
	assertThat(actual_2, lessThan(current));
	assertThat(actual_1, is(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.HotSequenceTest - doSome mono: 1624784472457
[main] INFO com.windowforsun.coldhot.HotSequenceTest - doSome current: 1624784472741
```  

`Mono` 시퀀스를 선언하는 시점에 바로 실제 시퀀스 생성 한번만 이뤄지고, 
그 다음에 `doSome(current)` 가 호출 되었다. 
이후 시퀀스를 구독하는 2번 모두 앞서 생성된 시퀀스의 데이터가 전달되는 것을 확인 할 수 있다.  

다음으로는 `Flux` 를 `Hot Sequence` 로 생성하는 예제이다.  

```java
@Test
public void flux_just_hot() throws Exception {
	// given
	Flux<Long> flux = Flux.just(this.doSome("flux 1"), this.doSome("flux 2"));
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = flux.collectList().block();
	List<Long> actual_2 = flux.collectList().block();

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(lessThan(current)));
	assertThat(actual_2, everyItem(lessThan(current)));
	assertThat(actual_1, is(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.HotSequenceTest - doSome flux 1: 1624785830837
[main] INFO com.windowforsun.coldhot.HotSequenceTest - doSome flux 2: 1624785830946
[main] INFO com.windowforsun.coldhot.HotSequenceTest - doSome current: 1624785831164
```  

`Mono` 의 `Hot Sequence` 결과와 크게 다르지 않다. 
`Flux` 시퀀스를 선언하는 시점에 바로 실제 시퀀스 생성이 한번만 이뤄지고, 
이 후 `doSome(current)` 호출이 이뤄졌다.  
그리고 시퀀스를 구독하는 2번 모두 앞서 생성된 시퀀스의 데이터가 전달되고 있다.  


### Cold to Hot
`Cold Sequence` 는 매번 새로우 시퀀스를 생성하기 때문에 방송과 같은 `Broadcast` 방식의 송출에서는 
많은 단점을 가지고 있다. 
이러한 이유로 `Mono` 와 `Flux` 에서는 `Cold Sequence` 를 `Hot Sequence` 로 전환할 수 있는 다양한 방법을 제공한다.  


#### cache()
`Cold Sequence` 에 `cache()` 메소드를 사용해주면 `Cold Sequence` 를 `Hot Sequence` 로 전환할 수 있다. 

![그림 1]({{site.baseurl}}/img/java/concept-reactor-cold-hot-sequence-cacheForFlux.svg)  

```java
@Test
public void mono_cold_to_hot_cache() throws Exception {
	// given
	Mono<Long> mono = Mono.fromSupplier(() -> this.doSome("mono"));
	mono = mono.cache();
	long current = this.doSome("current");

	// when
	Long actual_1 = mono.block();
	Long actual_2 = mono.block();

	// then
	assertThat(actual_1, notNullValue());
	assertThat(actual_2, notNullValue());
	assertThat(actual_1, greaterThan(current));
	assertThat(actual_2, greaterThan(current));
	assertThat(actual_1, is(actual_2));
}

/*
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624787565726
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome mono: 1624787565851
 */


@Test
public void flux_cold_to_hot_cache() throws Exception {
	// given
	Flux<Long> flux = Flux.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")));
	flux = flux.cache();
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = flux.collectList().block();
	List<Long> actual_2 = flux.collectList().block();

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(greaterThan(current)));
	assertThat(actual_1, is(actual_2));
}

/*
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624787655846
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624787656068
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624787656175	
 */
```  

`Mono`, `Flux` 모두 이전 `Cold Sequence` 를 생성하는 팩토리 메소드로 시퀀스를 선언했다. 
테스트 결과와 출력된 로그를 보면 `Mono`,`Flux` 모두 `cache()` 메소드를 사용한 이후부터 
`Cold Sequence` 가 `Hot Sequence` 로 전환되면서 시퀀스는 단 한번만 생성하고, 
이후 구독이 이뤄지더라도 항상 이전에 생성해둔 시퀀스의 데이터를 전달해주는 것을 확인 할 수 있다.  

#### ConnectableFlux
`Cold Sequence` 를 `Hot Sequence` 로 전환하는 다른 방법은 `ConnectableFlux` 를 사용하는 것이다. 
`ConnectableFlux` 는 `Flux.publish()` 메소드를 통해 리턴 받을 수 있다. 
역할은 `Flux` 시퀀스의 데이터 `ConnectableFlux` 에 연결된  구독자들에게 전달하는 역할을 한다.  

```java
@Test
public void flux_cold_to_hot_connectableFlux() throws Exception {
	// given
	Flux<Long> flux = Flux.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")));
	ConnectableFlux<Long> connectableFlux = flux.publish();
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = new ArrayList<>();
	List<Long> actual_2 = new ArrayList<>();
	connectableFlux.subscribe(aLong -> actual_1.add(aLong), e -> {}, () -> {});
	connectableFlux.subscribe(aLong -> actual_2.add(aLong), e -> {}, () -> {});
	this.doSome("connect");
	connectableFlux.connect();

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(greaterThan(current)));
	assertThat(actual_1, is(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624788546137
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome connect: 1624788546249
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624788546357
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624788546466
```  

`Cold Sequence` 를 생성하는 방식과 동일하게 `Flux` 시퀀스를 선언하고, 
해당 시퀀스에서 `publish()` 메소드를 사용해서 `ConnectableFlux` 를 다시 생성했다. 
그리고 서로다른 2개의 구독을 정의하고 `connect()` 메소드를 호출 했다. 
`connect()` 메소드 호출 시점에 실제 `Flux` 시퀀스가 단 한번만 생성되고 해당 시퀀스의 데이터가 구독자들에게 전달된다.  

추가로 `ConnectableFlux` 를 사용할 때 최소 구독자 수를 `autoConnect(n)` 과 같이 설정해서 
최소 구독자 수만큼 구독이 이뤄지면 `connect()` 메소드 호출이 명시적으로 수행되지 않아도 자동으로 데이터 발행이 이뤄지도록 할 수 있다. 

```java
@Test
public void flux_cold_to_hot_autoConnect() throws Exception {
	// given
	Flux<Long> flux = Flux.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")));
	Flux<Long> autoConnect = flux.publish().autoConnect(2);
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = new ArrayList<>();
	List<Long> actual_2 = new ArrayList<>();
	autoConnect.subscribe(aLong -> actual_1.add(aLong), e -> {}, () -> {});
	this.doSome("subscribe 1");
	autoConnect.subscribe(aLong -> actual_2.add(aLong), e -> {}, () -> {});

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(greaterThan(current)));
	assertThat(actual_1, is(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624789105530
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome subscribe 1: 1624789105642
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624789105752
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624789105863
```  

테스트 결과와 로그는 `ConnectableFlux` 와 동일하다. 
차이점은 `connect()` 메소드 호출 대신 `publish()` 호출 시점에 `autoConnect(2)` 를 함께 호출해서 
최소 구독자 수를 지정하고 최소 구독자 수만큼 구독이 이뤄지면 자동으로 데이터 발행이 이뤄지도록 했다. 
로그를 살펴보면 실제로 첫번째 구독은 이뤄지고 두번째 구독 전의 로그인 `subscribe 1` 이후에 시퀀스가 한번 생성되고 
두 구독자 모두 동일한 시퀀스에서 데이터를 전달 받은 것을 확인 할 수 있다.  

#### share()
지금까지는 시퀀스 자체를 `Cold` 에서 `Hot` 으로 변경하는 방법에 대해서 살펴보았다. 
그와 다르게 `Mono`, `Flux` 에 있는 `share()` 메소드는 비슷하지만 약간 다른 동작을 보여준다.  

`share()` 은 새로은 `Flux` 를 반환하는 메소드인데, 가장 
먼저 구독한 구독자가 사용하는 시퀀스를 이후 구독자들과 함께 공유 가능한 시퀀스가 생성된다. 
자세한 동작은 아래 예제를 통해 살펴본다.  

먼저 아래 예제를 살펴보면 `Cold Sequence` 과 동일한 결과가 나오는 것을 확인 할 수 있다.  

```java
@Test
public void flux_cold_to_hot_share_cold() throws Exception {
	// given
	Flux<Long> flux = Flux
			.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")))
			.delayElements(Duration.ofMillis(100));
	flux = flux.share();
	long current = this.doSome("current");

	// when
	List<Long> actual_1 = flux.collectList().block();
	List<Long> actual_2 = flux.collectList().block();

	// then
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(greaterThan(current)));
	assertThat(actual_1, not(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624813739776
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624813739917
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624813740027
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624813740354
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624813740463
```  

`Flux` 시퀀스는 우선 `Cold Sequence` 로 생성되고 구독하게 되면 `delayElements(Duration.ofMillis(100))` 설정으로 
100 밀리초마다 아이템을 발행하는 동작을 하게 된다. 
그리고 총 2번의 구독이 이뤄지는데 구독이 `block()` 메소드로 `Blocking` 방식으로 이뤄지기 때문에 
1번째 구독이 완료되고나서 2번째 구독이 수행된다. 
이때 시퀀스 동작이 `Cold Sequence` 와 동일하게 구독 마다 시퀀스가 새롭게 생성되는 것을 확인 할 수 있다.  

다음으로 아래 예제를 확인하면 `Hot Sequence` 처럼 동작되는 것을 확인 할 수 있다.  

```java
@Test
public void flux_cold_to_hot_share_hot() throws Exception {
	// given
	Flux<Long> flux = Flux
			.fromStream(() -> Stream.of(this.doSome("flux 1"), this.doSome("flux 2")))
			.delayElements(Duration.ofMillis(100));
	flux = flux.share();
	long current = this.doSome("current");

	// when
	List<Long> actual_1  = new ArrayList<>();
	flux.subscribe(aLong -> actual_1.add(aLong));
	List<Long> actual_2 = new ArrayList<>();
	flux.subscribe(aLong -> actual_2.add(aLong));

	// then
	Thread.sleep(400);
	assertThat(actual_1, hasSize(2));
	assertThat(actual_2, hasSize(2));
	assertThat(actual_1, everyItem(greaterThan(current)));
	assertThat(actual_2, everyItem(greaterThan(current)));
	assertThat(actual_1, is(actual_2));
}
```  

```
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome current: 1624814119552
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 1: 1624814119677
[main] INFO com.windowforsun.coldhot.ColdSequenceTest - doSome flux 2: 1624814119786
```  

시퀀스는 동일하게 `Cold Sequence` 로 생성되고 구독이 이뤄지면 동일하게 100 밀리초마다 아이템을 발행하는 동작이 이뤄진다. 
그리고 총 2번의 구독이 이뤄지는데, 이번에는 `subscribe()` 를 사용해서 `Callback` 방식으로 `Blocking` 없이 구독이 함께 이뤄진다. 
이때 시퀀스의 구독 동작을 보면 `Hot Sequence` 처럼 한번 시퀀스를 생성하고 
그 시퀀스의 아이템들이 2개의 구독자들에게 발행되는 것을 확인 할 수 있다.  

---
## Reference
[Hot vs Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactive.hotCold)  
[Hot Versus Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactor.hotCold)  
[Reactor Hot Publisher vs Cold Publisher](https://www.vinsguru.com/reactor-hot-publisher-vs-cold-publisher/)  

