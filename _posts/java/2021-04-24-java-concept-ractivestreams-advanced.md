--- 
layout: single
classes: wide
title: "[Java 개념] Reactive Streams 활용"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Reactive Streams 를 활용해'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Reactive
    - Java 9
    - Reactive Streams
    - Operator
    - Scheduler
toc: true
use_math: true
---  

## Reactive Streams 활용하기 
2021-04-18-java-concept-reactive-before-and-after
에서는 `Reactive Streams` 가 무엇이고 구성요소와 어떻게 사용하는 지에 대해서 알아보았다. 
`Reactive Streams` 는 간단하게 생산자와 소비자로 분리되어서 데이터를 주고 받고 처리하는 일련의 흐름이라고 할 수 있다. 
여기서 데이터를 주고 받는 다는 것을 중간에 어떤 별도의 처리자의 역할을 수행하는 요소가 필요할 수 있다.  

아래는 `1 ~ 5` 까지 데이터를 차례대로 생산하는 `MyPublisher` 와 이를 받아 처리하는 `MySubscriber` 에 대한 예시이다. 

```java
static class MyPublisher implements Flow.Publisher<String> {
    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        subscriber.onSubscribe(new Flow.Subscription() {
            @Override
            public void request(long n) {
                subscriber.onNext("1");
                subscriber.onNext("2");
                subscriber.onNext("3");
                subscriber.onNext("4");
                subscriber.onNext("5");
                subscriber.onComplete();
            }

            @Override
            public void cancel() {

            }
        });
    }
}

static class MySubscriber implements Flow.Subscriber<String> {
    private List<String> actual;

    public MySubscriber(List<String> actual) {
        this.actual = actual;
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.actual.add("onSubscribe");
        subscription.request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(String s) {
        this.actual.add(s);
    }

    @Override
    public void onError(Throwable t) {
        this.actual.add("onError");
    }

    @Override
    public void onComplete() {
        this.actual.add("onComplete");
    }
}

@Test
public void pub_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    pub.subscribe(sub);

    // then
    assertThat(actual, hasSize(7));
    assertThat(actual, contains(
            "onSubscribe",
            "1",
            "2",
            "3",
            "4",
            "5",
            "onComplete"
    ));
}
```  

위 데이터 스트림을 도식화 하면 아래와 같이 아주 간단한 구조로 `pub`(`MyPublisher`) 이 데이터를 생산해서 넘겨주고, 
`sub`(`MySubscriber`) 넘어온 데이터를 처리하는 아주 단순한 구조이다.  

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_1)  



### Operator
`Publisher`, `Subscriber` 데이터 흐름사이에 어떠한 동작을 추가하는 것을 `Operator` 라고 한다. 
`Publisher`, `Subscriber`, `Processor` 를 사용해서 `Operator` 를 직접 구현해보고 테스트를 진행해 본다.  


#### DelegatePublisher

`DelegatePublisher` 라는 새로운 `Publisher` 를 사용해서, 
`MyPublisher` 와 `MySubscriber` 사이에서 단순히 데이터 스트림의 전달자 역할만 하는 스트림을 구성하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_2)  


```java
static class DelegatePublisher implements Flow.Publisher<String> {
    private Flow.Publisher<String> upper;

    public DelegatePublisher(Flow.Publisher<String> upper) {
        this.upper = upper;
    }

    @Override
    public void subscribe(Flow.Subscriber subscriber) {
        this.upper.subscribe(subscriber);
    }
}

@Test
public void pub_delegatePub_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Publisher<String> delegatePub = new DelegatePublisher(pub);
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    delegatePub.subscribe(sub);

    // then
    assertThat(actual, hasSize(7));
    assertThat(actual, contains(
            "onSubscribe",
            "1",
            "2",
            "3",
            "4",
            "5",
            "onComplete"
    ));
}
```  

`DelegatePublisher` 의 맴버 변수인 `upper` 는 데이터 스트림에서 이전 `Publisher` 인 `MyPublisher` 의 객체가 된다. 
그리고 `subscribe()` 메소드로 전달되는 `Subscriber` 인 `MySubscriber` 와 `MyPublisher` 를 `upper.subscribe(subscriber)` 메소드를 통해 연결하는 역할을 한다.  


#### PrefixPublisher

`DelegatePublisher` 를 조금만 활용하면 아래와 같이 `MyPublisher`, `MySubscriber` 데이터 스트림 중간에서 
데이터의 접두사를 붙이는 `PrefixPublisher` 를 추가할 수 있다. 

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_3)  

```java
static class PrefixPublisher implements Flow.Publisher<String> {
    private Flow.Publisher<String> upper;

    public PrefixPublisher(Flow.Publisher<String> upper) {
        this.upper = upper;
    }

    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        this.upper.subscribe(new Flow.Subscriber<String>() {
            @Override
            public void onSubscribe(Flow.Subscription subscription) {
                subscriber.onSubscribe(subscription);
            }

            @Override
            public void onNext(String item) {
                subscriber.onNext("Prefix-" + item);
            }

            @Override
            public void onError(Throwable throwable) {
                subscriber.onError(throwable);
            }

            @Override
            public void onComplete() {
                subscriber.onComplete();
            }
        });
    }
}

@Test
public void pub_prefixPub_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Publisher<String> prefixPub = new PrefixPublisher(pub);
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    prefixPub.subscribe(sub);

    // then
    assertThat(actual, hasSize(7));
    assertThat(actual, contains(
            "onSubscribe",
            "Prefix-1",
            "Prefix-2",
            "Prefix-3",
            "Prefix-4",
            "Prefix-5",
            "onComplete"
    ));
}
```  

#### DelegateSubscriber

`PrefixPublisher` 의 `subscribe()` 메소드 내용을 보면, 
`upper.subscribe(new Subscribe() ...)` 와 같이  익명 `Subscriber` 객체를 사용하고 있다. 
여기서 사용되는 익명 `Subsciber` 객체는 `subscribe()` 메소드에 전달된 `MySubscriber` 와 `MyPublisher` 중간에서 
데이터 전달을 중계 해주거나, 데이터 처리, 에러 처리 등을 추가 할 수 있는 역할을 한다.  

이를 `DelegateSubscriber` 라는 추상 클래스로 다시 구성하면 아래와 같다.  

```java
static abstract class DelegateSubscriber implements Flow.Subscriber<String> {
    private Flow.Subscriber next;

    public DelegateSubscriber(Flow.Subscriber next) {
        this.next = next;
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.next.onSubscribe(subscription);
    }

    @Override
    public void onError(Throwable throwable) {
        this.next.onError(throwable);
    }

    @Override
    public void onComplete() {
        this.next.onComplete();
    }
}
```  

#### MapPublisher
`Java Stream API` 에서 `Map` 은 스트림내 요소를 특정 값으로 변환하는 작업을 의미한다. 
앞서 미리 살펴본 `PrefixPublisher` 와 `DelegateSubscriber` 를 조금 더 응용하면 `Reactive Streams` 에서 `Map` 동작을 구현할 수 있다. 
`Map` 에서 실제 수행되는 동작은 하나의 파라미터와 리턴 값을 가지는 `Function` 인터페이스를 사용한다.  

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    // stub
}
```  

`MapPublisher` 를 구현하고 이를 통해 접두사를 붙이는 `Publisher` 를 구성해서 테스트 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_4)  

```java
static class MapPublisher implements Flow.Publisher<String> {
    private Flow.Publisher<String> upper;
    private Function<String, String> function;

    public MapPublisher(Flow.Publisher<String> upper, Function<String, String> function) {
        this.upper = upper;
        this.function = function;
    }

    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        this.upper.subscribe(new DelegateSubscriber(subscriber) {
            @Override
            public void onNext(String item) {
                subscriber.onNext(function.apply(item));
            }
        });
    }
}

@Test
public void pub_mapPrefixPub_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Publisher<String> mapPrefixPub = new MapPublisher(
            pub,
            item -> "MapPrefix-" + item
    );
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    mapPrefixPub.subscribe(sub);

    // then
    assertThat(actual, hasSize(7));
    assertThat(actual, contains(
            "onSubscribe",
            "MapPrefix-1",
            "MapPrefix-2",
            "MapPrefix-3",
            "MapPrefix-4",
            "MapPrefix-5",
            "onComplete"
    ));
}
```  

#### ReducePublisher
`Java Stream API` 에서 `Reduce`(`Reduction`) 는 스트림의 요소를 사용자가 정의할 수 있는 결과로 만드는 동작이다.  
이 또한 앞선 예제들을 활용하면 `Reactive Streams` 에서 사용될 수 있는 `Reduce` 동작을 구현할 수 있다. 
`Reduce` 에서 실제로 사용자가 정의할 수 있는 결과로 요소를 만드는 동작을 수행하는 부분은 2개의 파라미터와 리턴 값을 가지는 `BiFunction` 을 사용한다.  

```java
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
    
    // stub
}
```  

`ReducePbulisher` 를 구현하고 스트림의 요소가 문자열일 때 모두 더해 하나의 문자열로 만드는 `concat` 동작에 대한 테스트는 아래와 같다.  


![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_5)  


```java
static class ReducePublisher implements Flow.Publisher<String> {
    private Flow.Publisher<String> upper;
    private BiFunction<String, String, String> biFunction;
    private String init;

    public ReducePublisher(Flow.Publisher<String> upper, BiFunction<String, String, String> biFunction, String init) {
        this.upper = upper;
        this.biFunction = biFunction;
        this.init = init;
    }

    @Override
    public void subscribe(Flow.Subscriber subscriber) {

        this.upper.subscribe(new DelegateSubscriber(subscriber) {
            String result = init;

            // onNext() 호출하지 않고 데이터 취합
            @Override
            public void onNext(String item) {
                result = biFunction.apply(result, item);
            }

            // 완료되면 onNext() 로 데이터 전달 후 완료 처리
            @Override
            public void onComplete() {
                subscriber.onNext(result);
                super.onComplete();
            }
        });
    }
}

@Test
public void pub_reduceConcatPub_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Publisher<String> reduceConcatPub = new ReducePublisher(
            pub,
            String::concat, // (init, item) -> init.concat(item)
            ""
    );
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    reduceConcatPub.subscribe(sub);

    // then
    assertThat(actual, hasSize(3));
    assertThat(actual, contains(
            "onSubscribe",
            "12345",
            "onComplete"
    ));
}
```  

#### Processor
`Reactive Streams` 에는 `Publisher`, `Subscriber` 뿐만 아니라 
`Publisher` 와 `Subscriber` 를 모두 상속하는 `Processor` 인터페이스가 있다. 
앞서 구현된 `Operator` 인 `MapPublisher` 와 `ReducePublisher` 를 `Processor` 로 구현해 보면 다음과 같이 하나의 클래스로 구성이 가능하다.  

- MapProcessor

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_6)  

```java

static class MapProcessor implements Flow.Processor<String, String> {
    private Flow.Publisher<String> upperPub;
    private Flow.Subscriber<String> nextSub;
    private Function<String, String> function;

    public MapProcessor(Flow.Publisher<String> upperPub, Function<String, String> function) {
        this.upperPub = upperPub;
        this.function = function;
    }

    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        this.nextSub = (Flow.Subscriber<String>) subscriber;
        this.upperPub.subscribe(this);
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.nextSub.onSubscribe(subscription);
    }

    @Override
    public void onNext(String item) {
        this.nextSub.onNext(this.function.apply(item));
    }

    @Override
    public void onError(Throwable throwable) {
        this.nextSub.onError(throwable);
    }

    @Override
    public void onComplete() {
        this.nextSub.onComplete();
    }
}

@Test
public void pub_mapPrefixProcessor_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Processor<String, String> mapPrefixProcessor = new MapProcessor(
            pub,
            item -> "MapProcessorPrefix-" + item
    );
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    mapPrefixProcessor.subscribe(sub);

    // then
    System.out.println(Arrays.toString(actual.toArray()));
    assertThat(actual, hasSize(7));
    assertThat(actual, contains(
            "onSubscribe",
            "MapProcessorPrefix-1",
            "MapProcessorPrefix-2",
            "MapProcessorPrefix-3",
            "MapProcessorPrefix-4",
            "MapProcessorPrefix-5",
            "onComplete"
    ));
}
```  


- ReduceProcessor

![그림 1]({{site.baseurl}}/img/java/concept_reactivestreams_advanced_7)  

```java
static class ReduceProcessor implements Flow.Processor<String, String> {
    private Flow.Publisher<String> upperPub;
    private Flow.Subscriber<String> nextSub;
    private BiFunction<String, String, String> biFunction;
    private String result;

    public ReduceProcessor(Flow.Publisher<String> upperPub, BiFunction<String, String, String> biFunction, String init) {
        this.upperPub = upperPub;
        this.biFunction = biFunction;
        this.result = init;
    }

    @Override
    public void subscribe(Flow.Subscriber<? super String> subscriber) {
        this.nextSub = (Flow.Subscriber<String>) subscriber;
        this.upperPub.subscribe(this);
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.nextSub.onSubscribe(subscription);
    }

    @Override
    public void onNext(String item) {
        this.result = this.biFunction.apply(this.result, item);
    }

    @Override
    public void onError(Throwable throwable) {
        this.nextSub.onError(throwable);
    }

    @Override
    public void onComplete() {
        this.nextSub.onNext(this.result);
        this.nextSub.onComplete();
    }
}

@Test
public void pub_reduceConcatProcessor_sub() {
    // given
    List<String> actual = new LinkedList<>();
    Flow.Publisher<String> pub = new MyPublisher();
    Flow.Processor<String, String> reduceConcatProcessor = new ReduceProcessor(
            pub,
            String::concat, // (init, item) -> init.concat(item),
            ""
    );
    Flow.Subscriber<String> sub = new MySubscriber(actual);

    // when
    reduceConcatProcessor.subscribe(sub);

    // then
    System.out.println(Arrays.toString(actual.toArray()));
    assertThat(actual, hasSize(3));
    assertThat(actual, contains(
            "onSubscribe",
            "12345",
            "onComplete"
    ));
}
```  



### Scheduler 






---
## Reference
[Reactive Streams](https://www.reactive-streams.org/)  
[reactive-streams/reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm)  
[Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/index.html)  
[The Reactive Manifesto](https://www.reactivemanifesto.org/)  
[Flow (Java SE 9 & JDK 9 )](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html)  
