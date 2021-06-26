--- 
layout: single
classes: wide
title: "[Java 개념] "
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

`Reactive Streams` 의 특징으로 알고 있는 `Mono`, `Flux` 시퀀스를 구독하기 전까지 등록된 콜백의 수행이 되지 않는다는 
우리가 흔히 알고 있는 `Reactive Streams` 의 동작방식은 `Cold Sequence`(?)

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

음악 스트리밍 서비스를 예로 들어보자. 
아무도 애국가를 스트리밍 하고 있지 않다면 애국가는 어느 누구에게도 데이터를 전달하지 않는다. 
만약 애국가를 듣고 있는 2명의 사용자가 있다고 가정 하면, 
2명의 사용자가 듣는 음악의 구간은 서로 다르기 때문에 서로 다른 시퀀스 생성이 필요하다. 
이런 음악 스트리밍과 같은 상황에서 사용할 수 있는 것이바로 `Cold Sequence` 이다.  

`Cold Sequence` 를 생성하는 `Mono`, `Flux` 의 팩토리 메소드는 많지만 그 중에서 
이번 포스트는 `from` 으로 시작하는 메소드를 사용한다.  

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

`Mono` 시퀀스를 선언하고, `doSome(current)` 를 호출 했지만 `doSome(current)` 가 가장 먼저 찍힌 로그인 것을 보면 
시퀀스 선언시에 어떠한 동작도 수행되지 않았다. 
그리고 `actual_1`, `actual_2` 값이 서로 다른 것을 통해 구독 마다 새로운 데이터가 생성되었음을 알 수 있다. 


### Hot Sequence

### Cold to Hot


---
## Reference
[Hot vs Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactive.hotCold)  
[Hot Versus Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactor.hotCold)  
[Reactor Hot Publisher vs Cold Publisher](https://www.vinsguru.com/reactor-hot-publisher-vs-cold-publisher/)  

