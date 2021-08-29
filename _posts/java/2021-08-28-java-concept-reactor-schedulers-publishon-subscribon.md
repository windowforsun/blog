--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Schedulers 와 PublishOn, SubscribeOn"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 Schedulers 의 종류와 PublishOn, SubscribeOn 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Schedulers
  - PublishOn
  - SubscribeOn
toc: true 
use_math: true
---  

## Schedulers
`Reactor` 에서는 기본적으로 [Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html)
라는 것을 통해 비동기 스크림 처리를 지원한다. 
사용자는 `Schedulers` 를 사용해서 작업 성격에 맞는 비동기 및 `Non-Blocking` 작업을 수행할 수 있다. 
`Schedulers` 에서는 팩토리 메소드를 사용해서 성격에 따라 사용할 수 있는 스레드 모델을 제공한다. 
`Schedulers` 에서 제공하는 스레드 모델은 내부적으로 `ExecutorService` 에서 제공하는 다양한 스레드 모델을 기반으로 구성된다. 

Schedulers Method|Desc
---|---
[parallel()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#parallel--)|고정된 크기의(host core size) 스레드 풀을 사용하고 병령 작업에 적합. 
[single()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#single--)|단일 스레드를 사용하고 지연이 적은 일회성 병렬작업에 적합
[elastic()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#elastic--)|무한대 크기를 가지는 스레드 풀을 사용하고 지연시간이 오래걸리는 블로킹 작업에서 사용(`Deprecated`)
[boundedElastic()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#boundedElastic--)|고정된 크기의(host core size * 10) 스레드 풀을 사용하고, `elastic()` 과 같이 지연시간이 큰 블로킹 작업에 적합 
[immediate()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#immediate--)|호출자의 스레드를 그대로 사용 
[fromExecutorService(ExecutorService)](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#fromExecutorService-java.util.concurrent.ExecutorService-)|사용자가 정의한 `ExecutorService` 의 스레드 풀을 사용

추가로 `immediate()`, `fromExecutorService()` 를 제외한 스레드 모델은 `new` 프리픽스가 붙은 팩토리 메소드를 통해 커스텀한 설정으로 구성해서 사용할 수 있다.  
- `newBoundedElastic()`
- `newElastic()`(`Deprecated`)
- `newParallel()`
- `newSingle()`

## PublishOn, SubscribeOn
`Schedulers` 의 존재와 다양한 스레드 모델을 제공하는 이유는 `PublishOn` 과 `SubscribeOn` 을 사용하기 위함에 있다. 
그렇다면 왜 `Schedulers` 을 바탕으로 적절한 스레드 모델을 선택하고 `PublishOn` 과 `SubscribeOn` 을 사용해야하는 지에 대해 의문을 가질 수 있다.  

`Reactor` 를 사용해서 `Non-Blocking` 모델을 사용해서 애플리케이션을 구성하는 목적은 적은 자원으로 최대의 효율을 얻기 위함이다. 
하지만 특정 `Publisher`, `Subscriber` 시퀀스 처리에 있어 불가피 하게 지연이 발생한다면(`Blocking`) 전체적인 애플리케이션 레이턴시에 여향을 미칠 수 있다.  

요청을 처리하는 스레드의 수행 중 `Blocking` 구간이 존재한다고 가정해보자. 
요청은 쉬지 않고 계속 들어올 때 `Blocking` 동작으로 인해 많은 요청들이 스레드를 할당받기 위해 계속해서 큐같은 공간에 대기하거나 타임아웃이 발생하게 될 것이다. 
이때 `Blocking` 동작에 대한 부분을 별도의 스레드 풀에 넘기고 완료되면 다시 요청 스레드에 반환하는 동작으로 수행한다면, 
과하게 요청 스레드를 늘리지 않고도 `Blocking` 작업을 효율적으로 처리할 수 있을 것이다.  

위 예시를 `Reactor` 의 `Publisher` 와 `Subscriber` 로 바꿔 생각해보자. 
`Publisher` 에서 요소 방출에 있어 오랜 시간(`Blocking`)이 걸린다면 `Subscriber` 입장에서는 불필요하게 지연시간이 길어지고 되고, 
이는 `Subscriber` 가 수행되는 스레드 지연으로 이어진다. 
반대로 `Publisher` 는 빠른 속도로 요소를 방출하지만 `Subscriber` 가 처리하는데 오랜 시간이 걸린다면 `Publisher` 입장에서도 지연시간이 길어지고, 
이또한 `Publisher` 가 수행되는 스레드 지연으로 이어질 것이다.  

그리고 `publishOn`, `subscribeOn` 또한 `Operator` 의 한종류 이다. 
일반적인 `Operator` 와 차이점이 있다면 이미 설명한 것처럼 `Schedulers` 를 기반으로 시퀀스의 동작을 비동기, 병렬 작업으로 전환한다는 점에 있다.   

### PublishOn

![그림 1]({{site.baseurl}}/img/java/concept-reactor-schedulers-1.svg)

`Publisher` 는 빠르지만 `Subscriber` 에서 큰 지연이 발생할 때, `Publisher` 에 사용할 수 있다. 
마블 다이어그램을 보면 `publishOn()` 이 `upstream`(`Publisher`) 을 `subscribe()` 를 통해 구독하는 동작까지는 기존 스레드에서 동작하지만, 
`downstream`(`Subscriber`) 쪽으로 `doNext()`, `onComplete()`, `onError()` 에 해당하는 시그널들이 `Schedulers` 에 의해 동작하는 것을 확인 할 수 있다. 
다시 말하면 `downstream` 으로 방출하는 동작에 대해서만 `Schedulers` 에 의해 수행된다고 할 수 있다.  


### SubscribeOn

![그림 1]({{site.baseurl}}/img/java/concept-reactor-schedulers-2.svg)

`publishOn()` 이 필요한 상황을 비롯해서 
`Publisher` 에서 큰 지연이 발생하지만 `Subscriber` 는 빨리 처리가 가능할 때, `Subscriber` 에 사용할 수 있다. (`Publisher` 에 원래 `publishOn()` 이 추가돼야 하지만 추가되지 않은 경우를 가정 할 수 있다.)
마블 다이어그램을 보면 `subscribeOn()` 이 `subscribe()` 으로 `upstream`(`Publisher`)를 구독하는 동작부터 `Schedulers` 에 의해 동작되기 때문에, 
`onSubscribe()`, `request()` 와 `onNext()`, `onComplete()`, `onError()` 시그널들이 모두 `Schedulers` 에 의해 동작된다. 
다시 말하면 `publishOn()` 의 시그널를 포함해서 `upstream` 을 구독하는 동작 부터 `downstream` 으로 방출까지 `Schedulers` 을 바탕으로 수행된다고 할 수 있다. 
`subscribeOn` 은 `publishOn` 에 비해 `Schedulers` 로 수행되는 시그널이 많기 때문에, `Publisher` 의 지연상태나 동작에 따라 `Subscriber` 에서 사용해서 지연시간을 단축 시킬 수 있다.  


## 테스트
작업 시간에 있어 오랜 지연이 발생하는 `heavyPublisher` 와 `heavySubscriber` 를 임의로 구성해서 테스트를 진행해 본다. 
진행 전에 먼저 테스트에서 사용되는 유틸성 메소드에 대해서 간단하게 소개하고 진행한다.  

```java
@SneakyThrows
public static void sleep(long millis) {
	Thread.sleep(millis);
}

public static void waitSubscribe(Disposable... disposables) {
	while (true) {
		sleep(10);
		boolean isEnd = Arrays.stream(disposables)
				.map(disposable -> disposable.isDisposed())
				.reduce(true, (aBoolean, aBoolean2) -> aBoolean && aBoolean2);

		if (isEnd) {
			break;
		}
	}
}
```  

`sleep()` 메소드는 `@SneakyThrows` 로 예외 처리를 미리 해둔 실제 지연동작을 수행하는 메소드이다. 
다음으로 `waitSbuscribe()` 는 인자로 `subscribe()` 의 리턴값인 `Disposable` 를 받아 구독이 완전히 완료 될때까지 기다리는 동작을 수행한다.  

아래와 같이 `1 ~ 3` 정수를 방출할때 1초의 지연이 걸리는 `heavyPublisher` 가 있다고 가정해 보자. 
이후 예제에서도 계속해서 사용되니 동작에 대해서 전반적으로 인지해두는 것이 좋다. 

```java
Flux<Integer> source = Flux
		.<Integer>create(integerFluxSink -> {
			IntStream.range(1, 4).forEach(value -> {
				sleep(1000);
				integerFluxSink.next(value);
			});

			integerFluxSink.complete();
		})
		.doOnNext(integer -> log.info("publish : {}", integer));
```  

위 `heavyPublisher` 를 한번 구독할 때마다 최소 3초의 지연시간이 발생하게 될 것이다. 
테스트에서는 2개의 구독을 수행해서 실제로 6초의 시간이 소요되는 지 확인해 본다.  

```java
@Test
public void heavyPublisher_lightSubscriber() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source.subscribe(integer -> {
		actual_1.add(integer);
		log.info("subscribe_1 : {}", integer);
	});
	Disposable disposable_2 = source.subscribe(integer -> {
		actual_2.add(integer);
		log.info("subscribe_2 : {}", integer);
	});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
04:27:56.752 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:27:56.754 [main] INFO SchedulersTest - request(unbounded)
04:27:57.760 [main] INFO SchedulersTest - onNext(1)
04:27:57.760 [main] INFO SchedulersTest - subscribe_1 : 1
04:27:58.771 [main] INFO SchedulersTest - onNext(2)
04:27:58.771 [main] INFO SchedulersTest - subscribe_1 : 2
04:27:59.781 [main] INFO SchedulersTest - onNext(3)
04:27:59.781 [main] INFO SchedulersTest - subscribe_1 : 3
04:27:59.781 [main] INFO SchedulersTest - onComplete()
04:27:59.782 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:27:59.782 [main] INFO SchedulersTest - request(unbounded)
04:28:00.796 [main] INFO SchedulersTest - onNext(1)
04:28:00.796 [main] INFO SchedulersTest - subscribe_2 : 1
04:28:01.810 [main] INFO SchedulersTest - onNext(2)
04:28:01.810 [main] INFO SchedulersTest - subscribe_2 : 2
04:28:02.824 [main] INFO SchedulersTest - onNext(3)
04:28:02.824 [main] INFO SchedulersTest - subscribe_2 : 3
04:28:02.824 [main] INFO SchedulersTest - onComplete()
```  

</div>
</details>


테스트 코드에서 검증문과 로그를 확인해 보면 2개의 구독을 수행하는데 6초가 소요 됐음을 알 수 있고, 
모두 `main` 스레드에서만 수행된 것을 확인 할 수 있다. 
이는 `N` 번 구독한다면 총 `N * 3` 초의 시간이 소요 될것이라고 예상할 수 있다. 
이후 이 예제는 `subscribeOn` 을 사용해서 `N` 번 구독하더라도 최대 3초의 지연시간이 걸리도록 개선해 볼것이다.  


다음으로 구독 동작이 방출된 요소를 처리할 때마다 1초의 지연이 발생하는 `heavySubscriber` 가 있다고 가정해보자. 
이 또한 이후 예제에서도 계속해서 사용되니 동작에 대해 전반적으로 이해해두는 것이 좋다.  

```java
source.subscribe(integer -> {
            sleep(1000);
            log.info("subscribe : {}", integer);
        });
```  

위 `Subscriber` 는 `Publisher` 에서 아무리 빠른 속도로 요소를 방출해 주더라도 `요소 수 * 1초` 의 지연이 발생할 것이다.

```java
@Test
public void lightPublisher_heavySubscriber() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();

	// when
	stopWatch.start();
	Disposable disposable_1 = source.subscribe(integer -> {
		sleep(1000);
		actual_1.add(integer);
		log.info("subscribe_1 : {}", integer);
	});
	Disposable disposable_2 = source.subscribe(integer -> {
		sleep(1000);
		actual_2.add(integer);
		log.info("subscribe_2 : {}", integer);
	});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```

<details><summary>출력 로그</summary>
<div markdown="1">

```
04:30:04.099 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:30:04.100 [main] INFO SchedulersTest - request(unbounded)
04:30:04.102 [main] INFO SchedulersTest - onNext(1)
04:30:05.109 [main] INFO SchedulersTest - subscribe_1 : 1
04:30:05.109 [main] INFO SchedulersTest - onNext(2)
04:30:06.124 [main] INFO SchedulersTest - subscribe_1 : 2
04:30:06.124 [main] INFO SchedulersTest - onNext(3)
04:30:07.127 [main] INFO SchedulersTest - subscribe_1 : 3
04:30:07.127 [main] INFO SchedulersTest - onComplete()
04:30:07.128 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:30:07.128 [main] INFO SchedulersTest - request(unbounded)
04:30:07.128 [main] INFO SchedulersTest - onNext(1)
04:30:08.143 [main] INFO SchedulersTest - subscribe_2 : 1
04:30:08.143 [main] INFO SchedulersTest - onNext(2)
04:30:09.145 [main] INFO SchedulersTest - subscribe_2 : 2
04:30:09.145 [main] INFO SchedulersTest - onNext(3)
04:30:10.158 [main] INFO SchedulersTest - subscribe_2 : 3
04:30:10.158 [main] INFO SchedulersTest - onComplete()
```  

</div>
</details>

2개의 `heavySubscriber` 가 `Publisher` 하나를 구독하게 될 때 6초의 지연이 발생할 것을 확인할 수 있다. 
이 또한 `구독 수 * 3초` 만큼의 지연이 발생할 것으로 예상해볼 수 있다. 
이후 이 예제 또한 `publishOn` 을 사용해서 `N` 개의 `heavySubscriber` 가 구독을 수행하더라도 최대 3초의 지연이 발생하도록 개선해 본다.  

### PublishOn, SubscribeOn 만들어보기 
앞선 예제에서 `Publisher` 와 `Subscriber` 에서 지연이 발생할 때의 상황과 실제 동작에 따른 지연시간이 얼마나 소요되는지 확인해 봤다. 
간단하게 `publishOn` 과 `subscribeOn` 의 구현체를 만들어보고 적용해서 발생한 지연시간을 단축해 본다.  

그렇다면 어떻게 지연시간을 단축할 수 있을까 ? 
해결방법은 별도의 스레드를 구성해서 지연이 있는 동작을 병렬로 처리하면 `N` 번의 구독 동작이 수행 되더라도 
`N * 지연시간` 만큼 지연이 발생되지 않을 것이다. 
그리고 `Publisher`, `Subscriber` 에 적절하게 스레드를 사용해서 병렬로 시그널 동작이 수행될 수 있도록 구성해주면 된다.  

커스텀한 `publishOn()`, `subscribeOn()` 을 구현하고 나서 이를 실제 시퀀스에 `Operator` 로 등록은 [transform()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#transform-java.util.function.Function-)
과 [transformDeffer()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#transformDeferred-java.util.function.Function-)
이라는 `Operator` 를 사용해서 수행한다. 
`transform()` 은 간단하게 인자로 전달하는 시퀀스를 사용해서 기존 시퀀스의 동작, 시그널 등을 변환할 수 있는 `Operator` 이다.  

#### MySubscribeOn
먼저 `subscribeOn` 의 특정을 바탕으로 직접 `MySubscribeOn` 을 구현해 본다. 
`subscribeOn()` 은 `onSubscribe()` 로 시작해서 `onComplete()` 까지 모든 시그널이 별도의 스레드에서 수행된다. 

```java
public static class MySubscribeOn<T> implements Publisher<T> {
	private Publisher<T> upstream;
	private ExecutorService es;

	public MySubscribeOn(Publisher<T> upstream, ExecutorService es) {
		this.upstream = upstream;
		this.es = es;
	}

	@Override
	public void subscribe(Subscriber<? super T> subscriber) {
		this.es.execute(() -> upstream.subscribe(subscriber));
	}
}
```  

`MySubscribeOn` 의 구현은 간단하다. 
`upstream` 시퀀스를 실제 구독자인 `Subscriber` 가 구독할 수 있도록 연결해주면 되는데, 
이 연결을 별도의 스레드로 수행해주면 된다. 
`upstream` 은 실제 요소를 방출하는 시퀀스인 `Publisher` 를 의미한다.   
이렇게 되면 별도의 스레드에서 시퀀스를 구독하고 요소 방출까지 수행하기 때문에 `onSubscribe()` 부터 시작해서 `onComplete()` 시그널까지 비동기로 수행할 수 있다.  

실제로 시퀀스가 `heavyPublisher` 일때 `MySubscribeOn` 을 사용하면 `N` 번 구독하더라도 지연시간이 늘어나지 않는 것을 확인 할 수 있다.   

```java
@Test
public void heavyPublisher_myPublishOn_lightSubscriber_shorttime() {
	// given
	ExecutorService es = Executors.newFixedThreadPool(16, new CustomizableThreadFactory("subscribeOn-"));
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.transform(integerFlux -> new MySubscribeOn<>(integerFlux, es))
			.subscribe(integer -> {
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.transform(integerFlux -> new MySubscribeOn<>(integerFlux, es))
			.subscribe(integer -> {
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
05:09:52.569 [mySubscribeOn-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
05:09:52.569 [mySubscribeOn-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
05:09:52.574 [mySubscribeOn-1] INFO SchedulersTest - request(unbounded)
05:09:52.575 [mySubscribeOn-2] INFO SchedulersTest - request(unbounded)
05:09:53.588 [mySubscribeOn-2] INFO SchedulersTest - onNext(1)
05:09:53.588 [mySubscribeOn-1] INFO SchedulersTest - onNext(1)
05:09:53.588 [mySubscribeOn-2] INFO SchedulersTest - subscribe_2 : 1
05:09:53.588 [mySubscribeOn-1] INFO SchedulersTest - subscribe_1 : 1
05:09:54.600 [mySubscribeOn-1] INFO SchedulersTest - onNext(2)
05:09:54.600 [mySubscribeOn-1] INFO SchedulersTest - subscribe_1 : 2
05:09:54.600 [mySubscribeOn-2] INFO SchedulersTest - onNext(2)
05:09:54.600 [mySubscribeOn-2] INFO SchedulersTest - subscribe_2 : 2
05:09:55.609 [mySubscribeOn-1] INFO SchedulersTest - onNext(3)
05:09:55.609 [mySubscribeOn-2] INFO SchedulersTest - onNext(3)
05:09:55.609 [mySubscribeOn-1] INFO SchedulersTest - subscribe_1 : 3
05:09:55.609 [mySubscribeOn-2] INFO SchedulersTest - subscribe_2 : 3
05:09:55.610 [mySubscribeOn-2] INFO SchedulersTest - onComplete()
05:09:55.610 [mySubscribeOn-1] INFO SchedulersTest - onComplete()
```

</div>
</details>

`heavyPublisher` 시퀀스가 1초마다 요소를 방출하고 총 2번의 구독이 이뤄졌지만, 
`MySubscribeOn` 을 통해 `heavyPublisher` 구독이 별도의 스레드 풀에서 이뤄지기 때문에 6초라는 시간이 걸리는 것이 아니라, 
3초의 지연으로 모든 동작이 가능하다. 
그리고 실제로 `onSubscribe()` 부터 `onComplete()` 시그널까지 모두 각 스레드에서 수행한 것을 확인 할 수 있다.  

### MyPublishOn
다음으로 `publishOn()` 특성을 바탕으로 `MyPublishOn` 을 구현해본다. 
`publishOn()` 은 시퀀스에서 요소를 방출하고 완료/에러에 대한 시그널인 `onNext()`, `onComplete()`, `onError()` 시그널만 별도의 스레드에서 수행된다. 

```java
public static class MyPublishOn implements Processor<Integer, Integer> {
	public static int COUNTER = 0;
	private ExecutorService es;
	private Publisher<Integer> upstream;
	private Subscriber<Integer> downstream;

	public MyPublishOn(Publisher<Integer> upstream, String name) {
		this.upstream = upstream;
		this.es = Executors.newSingleThreadExecutor(new CustomizableThreadFactory(name + ++COUNTER + "-"));
	}

	@Override
	public void subscribe(Subscriber<? super Integer> s) {
		this.downstream = (Subscriber<Integer>) s;
		this.upstream.subscribe(this);
	}

	@Override
	public void onSubscribe(Subscription s) {
		this.downstream.onSubscribe(s);
	}

	@Override
	public void onNext(Integer data) {
		this.es.execute(() -> downstream.onNext(data));
	}

	@Override
	public void onError(Throwable t) {
		this.es.execute(() -> downstream.onError(t));
		this.es.shutdown();
	}

	@Override
	public void onComplete() {
		this.es.execute(() -> downstream.onComplete());
		this.es.shutdown();
	}
}
```  

`MyPublishOn` 은 비교적 `MySubscribeOn` 보다는 구현이 복잡하다. 
방식은 `Publisher` 인 `upstream` 과 `Subscriber` 인 `downstream` 을 모두 사용해서, 
`subscribe()` 메소드에서는 `downstream` 이 아닌, `MyPublishOn` 을 `upstream` 을 구독하도록 설정하는데 이때 비동기가 아닌 현재 스레드로 수행한다. 
그리고 이후 부터는 `upstream` 으로 부터 전달되는 시그널을 `downstream` 으로 전달하는데, 이때 비동기로 전달하느냐 현제 스레드에서 전달하느냐에 차이가 있다. 
`onSubscribe()` 는 까지만 비동기가 아닌 현재 스레드에서 전달하고 이후 `onNext()`, `onComplete()`, `onError()` 에 대해서만 비동기를 바탕으로 `downstream` 쪽으로 시그널을 전달해준다.  

이렇게 구현된 `MyPublishOn` 은 `Subscriber` 가 `heavySubscriber` 일때, `Publisher` 에 사용해서 `N` 번 구독 동작이 이뤄지더라도 지연이 증가하지 않는다.  

```java
@Test
public void lightPublisher_myPublishOn_heavySubscriber_shorttime() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.transformDeferred(integerFlux -> new MyPublishOn(integerFlux, "myPublishOn-"))
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
05:30:51.882 [main] INFO SchedulersTest - onSubscribe(StrictSubscriber)
05:30:51.884 [main] INFO SchedulersTest - request(unbounded)
05:30:51.889 [myPublishOn-1-1] INFO SchedulersTest - onNext(1)
05:30:51.892 [main] INFO SchedulersTest - onSubscribe(StrictSubscriber)
05:30:51.892 [main] INFO SchedulersTest - request(unbounded)
05:30:51.893 [myPublishOn-2-1] INFO SchedulersTest - onNext(1)
05:30:52.895 [myPublishOn-2-1] INFO SchedulersTest - subscribe_2 : 1
05:30:52.895 [myPublishOn-2-1] INFO SchedulersTest - onNext(2)
05:30:52.895 [myPublishOn-1-1] INFO SchedulersTest - subscribe_1 : 1
05:30:52.896 [myPublishOn-1-1] INFO SchedulersTest - onNext(2)
05:30:53.905 [myPublishOn-2-1] INFO SchedulersTest - subscribe_2 : 2
05:30:53.905 [myPublishOn-2-1] INFO SchedulersTest - onNext(3)
05:30:53.905 [myPublishOn-1-1] INFO SchedulersTest - subscribe_1 : 2
05:30:53.905 [myPublishOn-1-1] INFO SchedulersTest - onNext(3)
05:30:54.912 [myPublishOn-1-1] INFO SchedulersTest - subscribe_1 : 3
05:30:54.912 [myPublishOn-2-1] INFO SchedulersTest - subscribe_2 : 3
05:30:54.912 [myPublishOn-1-1] INFO SchedulersTest - onComplete()
05:30:54.912 [myPublishOn-2-1] INFO SchedulersTest - onComplete()
```  

</div>
</details>

`heavySubscriber` 동작으로 인해 각 구독이 총 3초가 소요되지만, `MyPublishOn` 을 통해 2번 구독이 이뤄지더라도 3초 이상의 지연은 발생하지 않았다. 
그리고 실제로 `onNext()`, `onComplete()` 시그널에 대해서만 비동기로 동작이 수행되는 것을 확인 할 수 있다.  


### publishOn()
`publishOn()` 을 사용했을 때 `Schedulers` 에 의해 동작하는 시그널은 아래와 같다. 
- `onNext()`
- `onComplete()`
- `onError()`

`publishOn()` 이 위와같은 특징을 가질 때, `heavyPublisher`, `heavySubscriber` 에 대해서 어떠한 동작 결과를 보일지 생각해보자. 
시퀀스에서 `downstream`(`Subscriber`) 쪽으로 요소를 방출하는 시그널에 대해서만 비동기로 동작하기 때문에 `heavySubscriber` 와 같이 
`Subscriber` 에서 지연이 발생하는 경우 지연시간을 단축할 수 있을 것이다.

Publisher|Subscriber|소요시간
---|---|---
heavyPublisher|lightSubscriber|longtime
lightPublisher|heavySubscriber|shorttime
heavyPublisher|heavySubscriber|longtime

실제로 위와 같은 결과가 나오는지 테스트 코드를 통해 살펴본다.  

#### heavyPublisher, lightSubscriber
먼저 `heavyPublisher` 와 `lightSubscriber` 가 사용되는 경우의 결과는 아래와 같다.  

```java
@Test
public void heavyPublisher_publishOn_lightSubscriber_longtime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.publishOn(Schedulers.newParallel("publishOn"))
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribe(integer -> {
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribe(integer -> {
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, hasItems(1, 2, 3));
	assertThat(actual_2, hasItems(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
18:13:41.341 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:13:41.343 [main] INFO SchedulersTest - | request(unbounded)
18:13:42.367 [publishOn-1] INFO SchedulersTest - | onNext(1)
18:13:42.369 [publishOn-1] INFO SchedulersTest - subscribe_1 : 1
18:13:43.376 [publishOn-1] INFO SchedulersTest - | onNext(2)
18:13:43.377 [publishOn-1] INFO SchedulersTest - subscribe_1 : 2
18:13:44.390 [publishOn-1] INFO SchedulersTest - | onNext(3)
18:13:44.390 [publishOn-1] INFO SchedulersTest - subscribe_1 : 3
18:13:44.391 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:13:44.391 [publishOn-1] INFO SchedulersTest - | onComplete()
18:13:44.391 [main] INFO SchedulersTest - | request(unbounded)
18:13:45.405 [publishOn-2] INFO SchedulersTest - | onNext(1)
18:13:45.405 [publishOn-2] INFO SchedulersTest - subscribe_2 : 1
18:13:46.413 [publishOn-2] INFO SchedulersTest - | onNext(2)
18:13:46.413 [publishOn-2] INFO SchedulersTest - subscribe_2 : 2
18:13:47.426 [publishOn-2] INFO SchedulersTest - | onNext(3)
18:13:47.426 [publishOn-2] INFO SchedulersTest - subscribe_2 : 3
18:13:47.426 [publishOn-2] INFO SchedulersTest - | onComplete()
```  

</div>
</details>

`heavyPublisher` 인 상태에서 `publishOn()` 을 사용하게 되면 `publishOn()` 이 `heavyPublisher` 를 구독하는 동작은 동기로 이뤄지기 때문에, 
모든 구독이 완료된 이후에야 다음 구독이 수행될 수 있으므로 구독 횟수만큼 지연이 발생하게 된다.  

#### lightPublisher, heavySubscriber
다음으로 `lightPublisher` 와 `heavySubscriber` 가 사용되는 경우의 결과는 아래와 같다.  

```java
@Test
public void lightPublisher_publishOn_heavySubscriber_shorttime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.publishOn(Schedulers.newParallel("publishOn"))
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, hasItems(1, 2, 3));
	assertThat(actual_2, hasItems(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
18:29:44.148 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:29:44.152 [main] INFO SchedulersTest - | request(unbounded)
18:29:44.162 [publishOn-1] INFO SchedulersTest - | onNext(1)
18:29:44.162 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:29:44.162 [main] INFO SchedulersTest - | request(unbounded)
18:29:44.163 [publishOn-2] INFO SchedulersTest - | onNext(1)
18:29:45.175 [publishOn-1] INFO SchedulersTest - subscribe_1 : 1
18:29:45.175 [publishOn-2] INFO SchedulersTest - subscribe_2 : 1
18:29:45.175 [publishOn-1] INFO SchedulersTest - | onNext(2)
18:29:45.175 [publishOn-2] INFO SchedulersTest - | onNext(2)
18:29:46.186 [publishOn-1] INFO SchedulersTest - subscribe_1 : 2
18:29:46.186 [publishOn-2] INFO SchedulersTest - subscribe_2 : 2
18:29:46.186 [publishOn-1] INFO SchedulersTest - | onNext(3)
18:29:46.186 [publishOn-2] INFO SchedulersTest - | onNext(3)
18:29:47.200 [publishOn-1] INFO SchedulersTest - subscribe_1 : 3
18:29:47.200 [publishOn-2] INFO SchedulersTest - subscribe_2 : 3
18:29:47.201 [publishOn-1] INFO SchedulersTest - | onComplete()
18:29:47.201 [publishOn-2] INFO SchedulersTest - | onComplete()
```  

</div>
</details>

`heavySubscriber` 인 상태에서 `publishOn()` 을 사용하면 `publishOn()` 이 `lightPublisher` 를 구독하는 동작은 매우 빠르게 이뤄지고 
`publishOn()` 이 `heavySubscriber` 쪽으로 요소를 방출하는 시그널이 비동기로 동작되기 때문에 여러번 구독하더도 지연은 발생하지 않는다.  


#### heavyPublisher, heavySubscriber
마지막으로 `heavyPublisher` 와 `heavySubscriber` 가 함께 사용되는 경우는 아래와 같다.  

```java
@Test
public void heavyPublisher_publishOn_heavySubscriber_longtime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.publishOn(Schedulers.newParallel("publishOn"))
			.doOnNext(integer -> log.info("publish : {}", integer))
			.log();
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribe(integer -> {
				sleep(1000);
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, hasItems(1, 2, 3));
	assertThat(actual_2, hasItems(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
18:41:08.891 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:41:08.895 [main] INFO SchedulersTest - | request(unbounded)
18:41:09.906 [publishOn-1] INFO SchedulersTest - | onNext(1)
18:41:10.912 [publishOn-1] INFO SchedulersTest - subscribe_1 : 1
18:41:10.912 [publishOn-1] INFO SchedulersTest - | onNext(2)
18:41:11.921 [publishOn-1] INFO SchedulersTest - subscribe_1 : 2
18:41:11.921 [publishOn-1] INFO SchedulersTest - | onNext(3)
18:41:11.922 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
18:41:11.922 [main] INFO SchedulersTest - | request(unbounded)
18:41:12.936 [publishOn-1] INFO SchedulersTest - subscribe_1 : 3
18:41:12.936 [publishOn-2] INFO SchedulersTest - | onNext(1)
18:41:12.937 [publishOn-1] INFO SchedulersTest - | onComplete()
18:41:13.943 [publishOn-2] INFO SchedulersTest - subscribe_2 : 1
18:41:13.943 [publishOn-2] INFO SchedulersTest - | onNext(2)
18:41:14.949 [publishOn-2] INFO SchedulersTest - subscribe_2 : 2
18:41:14.949 [publishOn-2] INFO SchedulersTest - | onNext(3)
18:41:15.963 [publishOn-2] INFO SchedulersTest - subscribe_2 : 3
18:41:15.963 [publishOn-2] INFO SchedulersTest - | onComplete()
```  

</div>
</details>

`heavyPublisher`, `heavySubscriber` 인 상태에서 `publisherOn` 을 사용하면 `heavyPublisher` 의 지연과 `heavySubscriber` 의 지연이 복합적으로 발생된다. 
그러므로 `heavyPublisher` 만 사용된 상태보다 더 큰 지연시간을 보이게 된다. 
만약 여기서 구독을 한번 더 하게 되면 3초가 추가된 10초 정도 지연이 발생하게 될 것이다.  


### subscribeOn()
`subscribeOn()` 을 사용했을 때 `Schedulers` 에 의해 동작하는 시그널은 아래와 같다. 
- `onSubscribe()`
- `onRequest()`
- `onNext()`
- `onComplete()`
- `onError()`

`subscribeOn()` 이 위와같은 특징을 가질 때, `heavyPublisher`, `heavySubscriber` 에 대해서 어떠한 동작 결과를 보일지 생각해보자.
`subscribeOn()` 이 `uptream`(`Publisher`) 를 구독하고 `downstream`(`Subscriber`) 에게 `upstream` 의 시그널을 전달하는 것까지 모두 
비동기로 동작되기 때문에, `Publisher`, `Subscriber` 모두에 지연에 발생하더라도 해결이 가능할 것이다.  

Publisher|Subscriber|소요시간
---|---|---
heavyPublisher|lightSubscriber|shorttime
lightPublisher|heavySubscriber|shorttime
heavyPublisher|heavySubscriber|shorttime

실제로 위와 같은 결과가 나오는지 테스트 코드를 통해 살펴본다.

#### heavyPublisher, lightSubscriber
먼저 `heavyPublisher` 가 사용되는 결과는 아래와 같다.  

```java
@Test
public void heavyPublisher_subscribeOn_lightSubscriber_shorttime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.newParallel("subscribeOn");
	
	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, hasItems(1, 2, 3));
	assertThat(actual_2, hasItems(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
20:16:06.383 [subscribeOn-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:16:06.383 [subscribeOn-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:16:06.388 [subscribeOn-1] INFO SchedulersTest - request(unbounded)
20:16:06.388 [subscribeOn-2] INFO SchedulersTest - request(unbounded)
20:16:07.410 [subscribeOn-1] INFO SchedulersTest - onNext(1)
20:16:07.410 [subscribeOn-2] INFO SchedulersTest - onNext(1)
20:16:07.410 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 1
20:16:07.410 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 1
20:16:08.425 [subscribeOn-2] INFO SchedulersTest - onNext(2)
20:16:08.425 [subscribeOn-1] INFO SchedulersTest - onNext(2)
20:16:08.425 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 2
20:16:08.425 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 2
20:16:09.433 [subscribeOn-1] INFO SchedulersTest - onNext(3)
20:16:09.433 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 3
20:16:09.434 [subscribeOn-1] INFO SchedulersTest - onComplete()
20:16:09.434 [subscribeOn-2] INFO SchedulersTest - onNext(3)
20:16:09.434 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 3
20:16:09.434 [subscribeOn-2] INFO SchedulersTest - onComplete()
```  

</div>
</details>

`heavyPublisher` 인 상태라도 `subscribeOn()` 은 `heavyPublisher` 를 구독하는 동작부터 비동기로 이뤄지기 때문에, 
구독 횟수에 따른 지연은 발생하지 않는다.  


#### lightPublisher, heavySubscriber
다음으로 `heavySubscriber` 가 사용되는 테스트 결과는 아래와 같다.  

```java
@Test
public void lightPublisher_subscribeOn_heavySubscriber_shorttime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.newParallel("subscribeOn");

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				sleep(1000);
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				sleep(1000);
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
20:12:17.932 [subscribeOn-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:12:17.932 [subscribeOn-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:12:17.937 [subscribeOn-2] INFO SchedulersTest - request(unbounded)
20:12:17.937 [subscribeOn-1] INFO SchedulersTest - request(unbounded)
20:12:17.943 [subscribeOn-2] INFO SchedulersTest - onNext(1)
20:12:17.943 [subscribeOn-1] INFO SchedulersTest - onNext(1)
20:12:18.947 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 1
20:12:18.947 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 1
20:12:18.947 [subscribeOn-1] INFO SchedulersTest - onNext(2)
20:12:18.947 [subscribeOn-2] INFO SchedulersTest - onNext(2)
20:12:19.966 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 2
20:12:19.966 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 2
20:12:19.966 [subscribeOn-1] INFO SchedulersTest - onNext(3)
20:12:19.966 [subscribeOn-2] INFO SchedulersTest - onNext(3)
20:12:20.973 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 3
20:12:20.973 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 3
20:12:20.973 [subscribeOn-2] INFO SchedulersTest - onComplete()
20:12:20.973 [subscribeOn-1] INFO SchedulersTest - onComplete()
```  

</div>
</details>

`heavySubscriber` 인 경우에 `subscriberOn()` 을 사용한 경우에도 `lightPublisher` 를 구독하는 시점 부터 비동기로 이뤄지기 때문에 
구독 횟수에 따른 지연은 발생하지 않는다.  

#### heavyPublisher, heavySubscriber
마지막으로 `heavyPublisher` 와 `heavySubscriber` 가 사용되는 경우의 테스트 결과는 아래와 같다.  

```java
@Test
public void heavyPublisher_subscribeOn_heavySubscriber_shorttime() {
    // given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	List<Integer> actual_1 = new ArrayList<>();
	List<Integer> actual_2 = new ArrayList<>();
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.newParallel("subscribeOn");

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				sleep(1000);
				actual_1.add(integer);
				log.info("subscribe_1 : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				sleep(1000);
				actual_2.add(integer);
				log.info("subscribe_2 : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(actual_1, contains(1, 2, 3));
	assertThat(actual_2, contains(1, 2, 3));
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
20:08:37.671 [subscribeOn-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:08:37.671 [subscribeOn-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
20:08:37.675 [subscribeOn-2] INFO SchedulersTest - request(unbounded)
20:08:37.675 [subscribeOn-1] INFO SchedulersTest - request(unbounded)
20:08:38.689 [subscribeOn-1] INFO SchedulersTest - onNext(1)
20:08:38.689 [subscribeOn-2] INFO SchedulersTest - onNext(1)
20:08:39.703 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 1
20:08:39.703 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 1
20:08:40.703 [subscribeOn-1] INFO SchedulersTest - onNext(2)
20:08:40.703 [subscribeOn-2] INFO SchedulersTest - onNext(2)
20:08:41.714 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 2
20:08:41.714 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 2
20:08:42.717 [subscribeOn-1] INFO SchedulersTest - onNext(3)
20:08:42.717 [subscribeOn-2] INFO SchedulersTest - onNext(3)
20:08:43.731 [subscribeOn-1] INFO SchedulersTest - subscribe_1 : 3
20:08:43.731 [subscribeOn-2] INFO SchedulersTest - subscribe_2 : 3
20:08:43.731 [subscribeOn-2] INFO SchedulersTest - onComplete()
20:08:43.731 [subscribeOn-1] INFO SchedulersTest - onComplete()
```  

</div>
</details>

`heavyPublisher` 와 `heavySubscriber` 가 모두 사용되는 경우에 `subscribeOn()` 을 사용하면,  
소요시간은 총 6초가 나오지만 이는 구독 수에 따른 추가 지연은 발생하지 않았다. 
6초가 걸린 이유는 `heavyPublisher` 가 요소를 하나 방출하는데 1초가 걸리고 방출된 요소를 `heavySubscriber` 가 받는데 까지 1초가 소요되므로 
요소하나에 대해 2초가 소요된다. 
이러한 이유로 구독 한번이 모두 완료 될때까지 총 6초가 소요되기 때문에 추가 지연은 발생되지 않은 것이다.  


### Schedulers Thread Model
이번 섹션에서는 앞서 설명한 `Schedulers` 에서 팩토리 메소드로 제공하는 스레드 모델에 대해서 좀더 자세히 알아본다. 
`Schedulers.parallel()` 와 같은 형식으로 사용되는데, 
이때 `Schedulers.parallel()` 를 여러 `publishOn()` 혹은 `subscribeOn()` 에 사용한다면 동일한 `parallel()` 스레드 풀을 공유하게 된다. 
새로운 스레드 풀을 등록하고 싶다면 `Schedulers.newParallel()` 과 같은 방식으로 생성해서 사용할 수 있다.  

이후 예제에서는 `subscribeOn()` 만 사용해서 스레드 모델에 대해서 알아본다.  

#### single()
[single()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#single--)
은 `ExecutorService` 을 베이스로 하는 단일 스레드를 제공하는 모델이다.  

```java
@Test
public void scheduler_single_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.single();

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				log.info("subscribe : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				log.info("subscribe : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
02:07:15.909 [single-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:07:15.914 [single-1] INFO SchedulersTest - request(unbounded)
02:07:16.928 [single-1] INFO SchedulersTest - onNext(1)
02:07:16.928 [single-1] INFO SchedulersTest - subscribe : 1
02:07:17.939 [single-1] INFO SchedulersTest - onNext(2)
02:07:17.939 [single-1] INFO SchedulersTest - subscribe : 2
02:07:18.948 [single-1] INFO SchedulersTest - onNext(3)
02:07:18.948 [single-1] INFO SchedulersTest - subscribe : 3
02:07:18.948 [single-1] INFO SchedulersTest - onComplete()
02:07:18.949 [single-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:07:18.949 [single-1] INFO SchedulersTest - request(unbounded)
02:07:19.958 [single-1] INFO SchedulersTest - onNext(1)
02:07:19.958 [single-1] INFO SchedulersTest - subscribe : 1
02:07:20.971 [single-1] INFO SchedulersTest - onNext(2)
02:07:20.971 [single-1] INFO SchedulersTest - subscribe : 2
02:07:21.980 [single-1] INFO SchedulersTest - onNext(3)
02:07:21.980 [single-1] INFO SchedulersTest - subscribe : 3
02:07:21.980 [single-1] INFO SchedulersTest - onComplete()
```  

</div>
</details>

동일한 `single()` 을 2개의 `subscribeOn()` 에 사용하게 되는 경우 `single-1` 스레드 하나를 공유해서 사용하기 때문에, 
`heavyPublisher` 일때 추가 지연이 발생하게 된다.  


#### immediate()
[immediate()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#immediate--)
는 `publishOn()`, `subscribeOn()` 을 호출한 호출 스레드를 그대로 사용하는 스레드 모델이다.  

```java
@Test
public void scheduler_immediate_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.immediate();

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				log.info("subscribe : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribeOn(scheduler)
			.subscribe(integer -> {
				log.info("subscribe : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(5500L), lessThan(6500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
02:15:03.473 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:15:03.476 [main] INFO SchedulersTest - request(unbounded)
02:15:04.487 [main] INFO SchedulersTest - onNext(1)
02:15:04.488 [main] INFO SchedulersTest - subscribe : 1
02:15:05.496 [main] INFO SchedulersTest - onNext(2)
02:15:05.496 [main] INFO SchedulersTest - subscribe : 2
02:15:06.499 [main] INFO SchedulersTest - onNext(3)
02:15:06.499 [main] INFO SchedulersTest - subscribe : 3
02:15:06.499 [main] INFO SchedulersTest - onComplete()
02:15:06.500 [main] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:15:06.500 [main] INFO SchedulersTest - request(unbounded)
02:15:07.503 [main] INFO SchedulersTest - onNext(1)
02:15:07.503 [main] INFO SchedulersTest - subscribe : 1
02:15:08.507 [main] INFO SchedulersTest - onNext(2)
02:15:08.507 [main] INFO SchedulersTest - subscribe : 2
02:15:09.518 [main] INFO SchedulersTest - onNext(3)
02:15:09.518 [main] INFO SchedulersTest - subscribe : 3
02:15:09.518 [main] INFO SchedulersTest - onComplete()
```  

</div>
</details>

`immdeidate()` 를 사용한 경우 `subscribeOn()` 호출이 `main` 스레드에서 수행되었기 때문에 `subscribeOn()` 이 사용 됐더라도 모든 시그널 또한 `main` 스레드에서 수행된다.  

아래와 같이 `subscribeOn()` 이 별도의 스레드에서 수행되는 경우를 살펴보자.  

```java
@Test
public void scheduler_immediate_heavyPublisher_subscribeOn_byThread() throws ExecutionException, InterruptedException {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.immediate();
	ExecutorService es = Executors.newFixedThreadPool(2);

	// when
	stopWatch.start();
	Future<Disposable> future_1 = (Future<Disposable>) es.submit(() -> {
		source
				.subscribeOn(scheduler)
				.subscribe(integer -> {
					log.info("subscribe : {}", integer);
				});
	});
	Future<Disposable> future_2 = (Future<Disposable>) es.submit(() -> {
		source
				.subscribeOn(scheduler)
				.subscribe(integer -> {
					log.info("subscribe : {}", integer);
				});
	});
	future_1.get();
	future_2.get();
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
02:24:02.626 [pool-1-thread-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:24:02.626 [pool-1-thread-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:24:02.631 [pool-1-thread-1] INFO SchedulersTest - request(unbounded)
02:24:02.631 [pool-1-thread-2] INFO SchedulersTest - request(unbounded)
02:24:03.639 [pool-1-thread-1] INFO SchedulersTest - onNext(1)
02:24:03.639 [pool-1-thread-2] INFO SchedulersTest - onNext(1)
02:24:03.639 [pool-1-thread-1] INFO SchedulersTest - subscribe : 1
02:24:03.639 [pool-1-thread-2] INFO SchedulersTest - subscribe : 1
02:24:04.645 [pool-1-thread-1] INFO SchedulersTest - onNext(2)
02:24:04.645 [pool-1-thread-2] INFO SchedulersTest - onNext(2)
02:24:04.645 [pool-1-thread-1] INFO SchedulersTest - subscribe : 2
02:24:04.645 [pool-1-thread-2] INFO SchedulersTest - subscribe : 2
02:24:05.703 [pool-1-thread-1] INFO SchedulersTest - onNext(3)
02:24:05.703 [pool-1-thread-2] INFO SchedulersTest - onNext(3)
02:24:05.703 [pool-1-thread-1] INFO SchedulersTest - subscribe : 3
02:24:05.703 [pool-1-thread-2] INFO SchedulersTest - subscribe : 3
02:24:05.704 [pool-1-thread-2] INFO SchedulersTest - onComplete()
02:24:05.707 [pool-1-thread-1] INFO SchedulersTest - onComplete()
```  

</div>
</details>

별도의 `ExecutorService` 로 스레드 풀을 만들어 `immediate()` 를 사용하는 `subscribeOn()` 을 호출하는 경우, 
출력되는 로그와 같이 `ExecutorService` 에서 `subscribeOn()` 의 시그널이 수행된다. 
그리고 스레드 풀의 크기가 2이기 때문에 비동기로 2번의 구독 동작이 수행된 것을 확인 할 수 있다.

#### boundedElastic()
[boundedElastic()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#boundedElastic--)
은 `ExecutorService` 를 사용하는 스레드 모델로 스레드 풀 수는 최대 `host core size * 10` 이다. 
현재 테스트를 수행하는 호스트는 16코어 이기 때문에 최대 160개의 스레드가 생성 될 수 있다. 
처음 부터 모든 스레드 풀을 구성하는 것이 아니라 추가로 스레드가 필요한 경우 스레드를 늘려가고 기본 설정 60초 동안 특정 스레드가 실제로 할당 되지 않으면 
해당 스레드는 소멸된다. 
그리고 또하나의 특징은 `boundedElastic()` 은 스레드 풀이 모두 작업을 수행중일 때 추가되는 작업은 대기열에 추가된다. 
대기열의 최대 크기는 기본 설정 값으로 `100,000` 이다.  

위와 같은 특징으로 `boundedElastic()` 은 지연시간이 큰 작업에 대해서도 호율적으로 사용 할 수 있다.  


```java
@Test
public void scheduler_boundedElastic_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.boundedElastic();
	AtomicInteger counter = new AtomicInteger();

	// when
	stopWatch.start();
	Stream<Disposable> subscribeStream = Stream.generate(() -> {
		int num = counter.incrementAndGet();
		return source
				.subscribeOn(Schedulers.boundedElastic())
				.subscribe(integer -> {
					log.info("subscribe_{} : {}", num, integer);
				});
	}).limit(160);
	waitSubscribe(subscribeStream.toArray(Disposable[]::new));
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
02:42:40.276 [boundedElastic-153] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:42:40.254 [boundedElastic-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
.. 생략 ..
02:42:40.281 [boundedElastic-159] INFO SchedulersTest - request(unbounded)
02:42:40.281 [boundedElastic-2] INFO SchedulersTest - request(unbounded)
.. 생략 ..
02:42:41.297 [boundedElastic-116] INFO SchedulersTest - onNext(1)
02:42:41.297 [boundedElastic-116] INFO SchedulersTest - subscribe_116 : 1
02:42:41.311 [boundedElastic-159] INFO SchedulersTest - onNext(1)
02:42:41.317 [boundedElastic-159] INFO SchedulersTest - subscribe_159 : 1
.. 생략 ..
02:42:42.305 [boundedElastic-148] INFO SchedulersTest - onNext(2)
02:42:42.305 [boundedElastic-148] INFO SchedulersTest - subscribe_148 : 2
02:42:42.305 [boundedElastic-41] INFO SchedulersTest - onNext(2)
02:42:42.305 [boundedElastic-41] INFO SchedulersTest - subscribe_41 : 2
.. 생략 ..
02:42:43.306 [boundedElastic-44] INFO SchedulersTest - onNext(3)
02:42:43.306 [boundedElastic-44] INFO SchedulersTest - subscribe_44 : 3
02:42:43.306 [boundedElastic-148] INFO SchedulersTest - onNext(3)
02:42:43.306 [boundedElastic-148] INFO SchedulersTest - subscribe_148 : 3
.. 생략 ..
02:42:43.307 [boundedElastic-19] INFO SchedulersTest - onComplete()
02:42:43.337 [boundedElastic-160] INFO SchedulersTest - onComplete()
.. 생략 ..
```  

</div>
</details>

160 번의 구독을 수행했지만 현재 테스트 환경에서는 `boundedElastic()` 스레드 풀이 160개 까지 생성될 수 있으므로 모든 구독 작업이 비동기로 수행 된 것을 확인 할 수 있다.  


#### parallel()
[parallel()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#parallel--)
은 `ExecutorService` 를 사용하는 스레드 모델로 스레드 풀 수는 `host core size` 로 설정된다.
현재 테스트를 수행하는 호스트는 코어 수가 16이기 때문에 `parallel()` 스레드 풀 또한 16으로 생성된다.

`parallel()` 은 `boundedElastic()` 처럼 대기열이 존재하는 것이 아니기 때문에,
큰 지연시간을 가지는 작업에는 사용하지 않는 것이 좋고 짧은 지연시간이 예상되는 작업에 사용하기 좋다.

```java
@Test
public void scheduler_parallel_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.parallel();
	AtomicInteger counter = new AtomicInteger();

	// when
	stopWatch.start();
	Stream<Disposable> subscribeStream = Stream.generate(() -> {
		int num = counter.incrementAndGet();
		return source
				.subscribeOn(scheduler)
				.subscribe(integer -> {
					log.info("subscribe_{} : {}", num, integer);
				});
	}).limit(16);
	waitSubscribe(subscribeStream.toArray(Disposable[]::new));
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  


<details><summary>출력 로그</summary>
<div markdown="1">

```
02:28:20.997 [parallel-15] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
02:28:20.997 [parallel-1] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
.. 생략 .. 
02:28:21.008 [parallel-14] INFO SchedulersTest - request(unbounded)
02:28:21.010 [parallel-12] INFO SchedulersTest - request(unbounded)
.. 생략 ..
02:28:22.029 [parallel-10] INFO SchedulersTest - onNext(1)
02:28:22.029 [parallel-12] INFO SchedulersTest - subscribe_12 : 1
02:28:22.031 [parallel-15] INFO SchedulersTest - onNext(1)
02:28:22.032 [parallel-15] INFO SchedulersTest - subscribe_15 : 1
.. 생략 ..
02:28:23.035 [parallel-2] INFO SchedulersTest - onNext(2)
02:28:23.035 [parallel-2] INFO SchedulersTest - subscribe_2 : 2
02:28:23.036 [parallel-5] INFO SchedulersTest - onNext(2)
02:28:23.036 [parallel-5] INFO SchedulersTest - subscribe_5 : 2
.. 생략 ..
02:28:24.048 [parallel-8] INFO SchedulersTest - onNext(3)
02:28:24.048 [parallel-8] INFO SchedulersTest - subscribe_8 : 3
02:28:24.048 [parallel-15] INFO SchedulersTest - onNext(3)
02:28:24.048 [parallel-15] INFO SchedulersTest - subscribe_15 : 3
.. 생략 ..
02:28:24.059 [parallel-3] INFO SchedulersTest - onComplete()
02:28:24.058 [parallel-4] INFO SchedulersTest - onComplete()
.. 생략 ..
```  

</div>
</details>

`parallel()` 스레드 풀의 수가 16이기 때문에 `heavyPublisher` 를 16번 구독하더라도 모든 구독 동작이 비동기로 동작되므로 3초의 소요시간을 보여주고 있다.


#### elastic()
[elastic()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#elastic--)
은 `boundedElastic()` 과 대부분 동일한 특성을 가지고 있고, 가장 큰 차이는 스레드 풀의 수에 제한이 존재하지 않고 무한대로 스레드의 수가 늘어난다. 
사용되지 않는 스레드의 경우 동일하게 60초의 만료시간을 가지고 소멸된다.  

> `elastic()` 은 Deprecated 되어서  3.5.0 버전 이후 부터는 사용할 수 없다. 

```java
@Test
public void scheduler_elastic_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.doOnNext(integer -> log.info("publish : {}", integer));
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.elastic();
	AtomicInteger counter = new AtomicInteger();

	// when
	stopWatch.start();
	Stream<Disposable> subscribeStream = Stream.generate(() -> {
		int num = counter.incrementAndGet();
		return source
				.subscribeOn(Schedulers.elastic())
				.subscribe(integer -> {
					log.info("subscribe_{} : {}", num, integer);
				});
	}).limit(200);
	waitSubscribe(subscribeStream.toArray(Disposable[]::new));
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
03:04:07.993 [elastic-6] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
03:04:07.993 [elastic-4] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
.. 생략 ..
03:04:08.040 [elastic-200] INFO SchedulersTest - request(unbounded)
03:04:08.040 [elastic-201] INFO SchedulersTest - request(unbounded)
.. 생략 ..
03:04:09.048 [elastic-199] INFO SchedulersTest - onNext(1)
03:04:09.048 [elastic-199] INFO SchedulersTest - subscribe_198 : 1
03:04:09.048 [elastic-190] INFO SchedulersTest - onNext(1)
03:04:09.048 [elastic-190] INFO SchedulersTest - subscribe_189 : 1
.. 생략 ..
03:04:10.050 [elastic-8] INFO SchedulersTest - onNext(2)
03:04:10.054 [elastic-8] INFO SchedulersTest - subscribe_7 : 2
03:04:10.051 [elastic-180] INFO SchedulersTest - onNext(2)
03:04:10.054 [elastic-180] INFO SchedulersTest - subscribe_179 : 2
.. 생략 ..
03:04:11.047 [elastic-117] INFO SchedulersTest - onNext(3)
03:04:11.047 [elastic-117] INFO SchedulersTest - subscribe_116 : 3
03:04:11.048 [elastic-5] INFO SchedulersTest - onNext(3)
03:04:11.048 [elastic-5] INFO SchedulersTest - subscribe_4 : 3
.. 생략 ..
03:04:11.064 [elastic-54] INFO SchedulersTest - onComplete()
03:04:11.065 [elastic-200] INFO SchedulersTest - onComplete()
.. 생략 ..
```  

</div>
</details>

200 번의 구독을 수행 했지만 `elastic()` 의 스레드 풀의 수는 제한이 없기 때문에 병렬로 처리 가능한 최대한의 스레드를 생성해서, 
모든 구독 동작이 비동기로 수행된 것을 확인 할 수 있다.  

#### fromExecutorService
[fromExecutorService()](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#fromExecutorService-java.util.concurrent.ExecutorService-)
는 사용자가 사용할 스레드 풀을 `ExecutorServic` 를 통해 전달하는 방법으로 `Schedulers` 의 스레드 모델을 정의할 수 있다.  

```java
@Test
public void scheduler_fromExecutorService_heavyPublisher_subscribeOn() {
	// given
	Flux<Integer> source = Flux
			.<Integer>create(integerFluxSink -> {
				IntStream.range(1, 4).forEach(value -> {
					sleep(1000);
					integerFluxSink.next(value);
				});

				integerFluxSink.complete();
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();
	Scheduler scheduler = Schedulers.fromExecutorService(Executors.newFixedThreadPool(20));
	AtomicInteger counter = new AtomicInteger();

	// when
	stopWatch.start();
	Stream<Disposable> subscribeStream = Stream.generate(() -> {
		int num = counter.incrementAndGet();
		return source
				.subscribeOn(scheduler)
				.subscribe(integer -> {
					log.info("subscribe_{} : {}", num, integer);
				});
	}).limit(20);
	waitSubscribe(subscribeStream.toArray(Disposable[]::new));
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
03:13:33.426 [pool-1-thread-2] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
03:13:33.426 [pool-1-thread-20] INFO SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
.. 생략 ..
03:13:33.435 [pool-1-thread-1] INFO SchedulersTest - request(unbounded)
03:13:33.435 [pool-1-thread-10] INFO SchedulersTest - request(unbounded)
.. 생략 ..
03:13:34.452 [pool-1-thread-6] INFO SchedulersTest - onNext(1)
03:13:34.453 [pool-1-thread-6] INFO SchedulersTest - subscribe_6 : 1
03:13:34.452 [pool-1-thread-17] INFO SchedulersTest - onNext(1)
03:13:34.453 [pool-1-thread-17] INFO SchedulersTest - subscribe_17 : 1
.. 생략 ..
03:13:35.466 [pool-1-thread-14] INFO SchedulersTest - onNext(2)
03:13:35.466 [pool-1-thread-14] INFO SchedulersTest - subscribe_14 : 2
03:13:35.466 [pool-1-thread-10] INFO SchedulersTest - onNext(2)
03:13:35.466 [pool-1-thread-10] INFO SchedulersTest - subscribe_10 : 2
.. 생략 ..
03:13:36.467 [pool-1-thread-19] INFO SchedulersTest - onNext(3)
03:13:36.467 [pool-1-thread-19] INFO SchedulersTest - subscribe_19 : 3
03:13:36.467 [pool-1-thread-2] INFO SchedulersTest - onNext(3)
03:13:36.467 [pool-1-thread-2] INFO SchedulersTest - subscribe_2 : 3
.. 생략 ..
03:13:36.483 [pool-1-thread-17] INFO SchedulersTest - onComplete()
03:13:36.484 [pool-1-thread-16] INFO SchedulersTest - onComplete()
.. 생략 ..
```  

</div>
</details>

`ExecutorService` 를 사용해서 20개의 스레드 풀을 생성하고 20번 구독을 수행하면 구독 동작이 모두 비동기로 수행되는 것을 확인 할 수 있다.  


### Multiple PublishOn, SubscribeOn
`Reactor` 를 사용해서 시퀀스에 대한 스트림을 구성할 때 여러 동작에 따라 적절한 `publishOn()` 을 사용해서 효율을 올릴 수 있다.  

```java
@Test
public void multiple_scheduler_publishOn() {
	// given
	Flux<String> source = Flux
			.just("first", "second", "third")
			.doOnNext(s -> log.info("publish : {}", s))
			.publishOn(Schedulers.parallel())
			.map(s -> {
				String prefix = "prefix-" + s;
				log.info("prefix : {}", prefix);

				return s;
			})
			.publishOn(Schedulers.boundedElastic())
			.map(s -> {
				String suffix = s + "-suffix";
				log.info("suffix : {}", s);

				return suffix;
			})
			.log(log.getName());
	StopWatch stopWatch = new StopWatch();

	// when
	stopWatch.start();
	Disposable disposable_1 = source
			.subscribe(integer -> {
				sleep(1000);
				log.info("subscribe : {}", integer);
			});
	Disposable disposable_2 = source
			.subscribe(integer -> {
				sleep(1000);
				log.info("subscribe : {}", integer);
			});
	waitSubscribe(disposable_1, disposable_2);
	stopWatch.stop();

	// then
	assertThat(stopWatch.getTotalTimeMillis(), allOf(greaterThan(2500L), lessThan(3500L)));
}
```  

<details><summary>출력 로그</summary>
<div markdown="1">

```
03:20:35.821 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
03:20:35.823 [main] INFO SchedulersTest - | request(unbounded)
03:20:35.824 [main] INFO SchedulersTest - publish : first
03:20:35.825 [main] INFO SchedulersTest - publish : second
03:20:35.825 [main] INFO SchedulersTest - publish : third
03:20:35.825 [main] INFO SchedulersTest - | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
03:20:35.825 [main] INFO SchedulersTest - | request(unbounded)
03:20:35.826 [main] INFO SchedulersTest - publish : first
03:20:35.826 [main] INFO SchedulersTest - publish : second
03:20:35.827 [main] INFO SchedulersTest - publish : third
03:20:35.830 [parallel-2] INFO SchedulersTest - prefix : prefix-first
03:20:35.830 [parallel-1] INFO SchedulersTest - prefix : prefix-first
03:20:35.831 [parallel-2] INFO SchedulersTest - prefix : prefix-second
03:20:35.831 [parallel-1] INFO SchedulersTest - prefix : prefix-second
03:20:35.831 [parallel-2] INFO SchedulersTest - prefix : prefix-third
03:20:35.831 [parallel-1] INFO SchedulersTest - prefix : prefix-third
03:20:35.831 [boundedElastic-1] INFO SchedulersTest - suffix : first
03:20:35.831 [boundedElastic-2] INFO SchedulersTest - suffix : first
03:20:35.832 [boundedElastic-1] INFO SchedulersTest - | onNext(first-suffix)
03:20:35.832 [boundedElastic-2] INFO SchedulersTest - | onNext(first-suffix)
03:20:36.843 [boundedElastic-1] INFO SchedulersTest - subscribe : first-suffix
03:20:36.843 [boundedElastic-2] INFO SchedulersTest - subscribe : first-suffix
03:20:36.843 [boundedElastic-1] INFO SchedulersTest - suffix : second
03:20:36.843 [boundedElastic-2] INFO SchedulersTest - suffix : second
03:20:36.843 [boundedElastic-1] INFO SchedulersTest - | onNext(second-suffix)
03:20:36.843 [boundedElastic-2] INFO SchedulersTest - | onNext(second-suffix)
03:20:37.858 [boundedElastic-1] INFO SchedulersTest - subscribe : second-suffix
03:20:37.858 [boundedElastic-2] INFO SchedulersTest - subscribe : second-suffix
03:20:37.858 [boundedElastic-1] INFO SchedulersTest - suffix : third
03:20:37.858 [boundedElastic-2] INFO SchedulersTest - suffix : third
03:20:37.858 [boundedElastic-1] INFO SchedulersTest - | onNext(third-suffix)
03:20:37.858 [boundedElastic-2] INFO SchedulersTest - | onNext(third-suffix)
03:20:38.865 [boundedElastic-1] INFO SchedulersTest - subscribe : third-suffix
03:20:38.865 [boundedElastic-2] INFO SchedulersTest - subscribe : third-suffix
03:20:38.865 [boundedElastic-1] INFO SchedulersTest - | onComplete()
03:20:38.865 [boundedElastic-2] INFO SchedulersTest - | onComplete()
```  

</div>
</details>

위와 같이 특정 동작의 지연이나 특징에 따라 원하는 `Schedulers` 를 바탕으로 스트림을 구성할 수 있다. 
등록된 `Schedulers` 는 이후 하위 스트림에 모두 적용된다는 점을 주의해야 한다.  


---
## Reference
[Reactor Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html)  
[Flight of the Flux 3 - Hopping Threads and Schedulers](https://spring.io/blog/2019/12/13/flight-of-the-flux-3-hopping-threads-and-schedulers)  
[Reactor Schedulers – PublishOn vs SubscribeOn](https://www.vinsguru.com/reactor-schedulers-publishon-vs-subscribeon/)  


