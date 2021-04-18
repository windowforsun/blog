--- 
layout: single
classes: wide
title: "[Java 개념] Reactive Streams 의 이전과 이후"
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
    - Reactive
    - Java 9
    - Reactive Streams
    - Back-pressure
    - Iterable
    - Observable
toc: true
use_math: true
---  

## Reactive 이전의 Java
`Java 9` 에서 공개된 `Reactive`(`java.util.concurrent.Flow`) 방식의 이벤트 처리 방식의 등장 이전에 
`Java` 진영에서는 어떻게 이벤트 데이터를 처리했는지에 대해서 먼저 알아본다.  

### Iterable
`Iterable` 은 `Java` 언어를 사용해본 경험이 있다면 아주 친숙하고 한번쯤은 사용할 수 밖에 없는 
아주 익숙한 인터페이스이다. 
가장 큰 특징은 `Java` 에서 데이터 집합의 최상위 클래스인 `Collection` 의 상위 클래스라는 점이다. 
이는 다른 말로 하면 모든 `Collection` 의 하위 클래스들은 `Iterable` 기능을 사용할 수 있다고 할 수 있다. 
기능적인 특징으로는 `Iterable` 을 상속하는 클래스는 `for-each loop` 구문을 통해 집합 데이터를 순회 할 수 있다.  

`Collection`, `Iterable` 인터페이스의 정의 부분을 살펴보면 아래와 같다. 

```java
/**
 * The root interface in the <i>collection hierarchy</i>.  A collection
 * represents a group of objects, known as its <i>elements</i>.
 */
public interface Collection<E> extends Iterable<E> {
    // stub
}

/**
 * Implementing this interface allows an object to be the target of the enhanced
 * {@code for} statement (sometimes called the "for-each loop" statement).
 */
public interface Iterable<T> {
    /**
     * Returns an iterator over elements of type {@code T}.
     *
     * @return an Iterator.
     */
    Iterator<T> iterator();

    // stub
}
```  

`for-each loop` 로 사용한다면 아래와 같은 형태가 된다.  

```java
Collection<String> datas = new LinkedList<>();

for(String data : datas) {
   doSometing(data);
}
```  

그리고 `Iterable` 인터페이스가 `for-each loop` 구문에서 사용 될 수 있도록 실질적인 인터페이스가 
정의 된 것은 `Iterator` 인터페이스 이다.  

```java
/**
 * An iterator over a collection.  {@code Iterator} takes the place of
 * {@link Enumeration} in the Java Collections Framework.  Iterators
 * differ from enumerations in two ways:
 */
public interface Iterator<E> {
    boolean hasNext();

    E next();
    
    // stub
}
```  

`Iterable` 은 `Iterator` 에서 제공하는 데이터를 원하는 시점에 
`Pull` 하는 방식으로 가져다가 사용할 수 있는 인터페이스 인것을 확인 할 수 있다.  

간단하게 `1 ~ 5` 인트형 데이터를 제공하는 `Iterable` 를 만들어서 테스트 수행하면 아래와 같다.  

```java
@Test
public void iterable_foreach() {
    // given
    Iterator<Integer> iterator = new Iterator<Integer>() {
        public final static int MAX = 5;
        private int i = 0;

        @Override
        public boolean hasNext() {
            return i < MAX;
        }

        @Override
        public Integer next() {
            return ++i;
        }
    };
    Iterable<Integer> iterable = new Iterable() {
        @Override
        public Iterator iterator() {
            return iterator;
        }
    };

    // when
    List<Integer> actual = new LinkedList<>();
    for (int num : iterable) {
        actual.add(num);
    }

    // then
    assertThat(actual, hasSize(5));
    assertThat(actual, contains(1, 2, 3, 4, 5));
}
```  

`Iterator` 만 사용해서 `1 ~5` 데이터를 제공하도록 만들어서 테스트 하면 아래와 같다. 

```java
@Test
public void iterator_foreach() {
    // given
    Iterator<Integer> iterator = new Iterator<Integer>() {
        public final static int MAX = 5;
        private int i = 0;

        @Override
        public boolean hasNext() {
            return i < MAX;
        }

        @Override
        public Integer next() {
            return ++i;
        }
    };

    // when
    List<Integer> actual = new LinkedList<>();
    for (; iterator.hasNext(); ) {
        actual.add(iterator.next());
    }

    // then
    assertThat(actual, hasSize(5));
    assertThat(actual, contains(1, 2, 3, 4, 5));
}
```  

`Iterable` 와 `for-each loop` 를 사용하는 경우 직접적으로 `Iterator` 인터페이스를 사용하지 않더라도, 
원하는 시점에 차례대로 데이터를 받을 수 있다. 
그리고 `Iterator` 인터페이스를 직접 구현해서 사용하면 원하는 시점에 `next()` 메소드를 호출해서  
데이터를 `Pull` 해서 받는 방식으로 처리 가능하다.  


### Observable
`Java` 에서 `Native` 하게 지원하는 `Observable` 인터페이스는 말그대로 데이터 혹은 이벤트를 
`Observing` 하면서 처리를 수행하는 디자인 패턴의 `Observer Pattern` 의 `Java` 구현체이다.  

>Observable 클래스는 Java 9 부터 `Deprecated` 되었다. 

`Observable` 클래스는 앞서 살펴본 `Iterable` 인터페이스와 구조와 동작에 있어 비슷하지만 약간에 차이가 있다. 
차이를 설명하기 위해 `Iterable` 에서 진행한 `1 ~ 5` 데이터를 주는 것을 `Observable` 로 구현하면 아래와 같다.  

>`Observable` 은 단독으로는 사용될 수 없고, `Observer` 라는 관찰자가 필요하다. 
>`Observable` 은 상태 변화 혹은 데이터, 이벤트를 주는 역할을 수행하고
>`Observer` 는 `Oberservable` 의 이벤트를 받아 처리하는 역할을 수행한다. 
>`Observable` 과 `Observer` 런타임을 분리하기 위해 `Observable` 은 별도의 스레드에서 수행될 수 있도록 한다.  

```java
static class MyObservable extends Observable implements Runnable {
    private int max;

    public MyObservable(int max) {
        this.max = max;
    }

    @Override
    public void run() {
        for(int i = 1; i <= this.max; i++) {
            setChanged();
            notifyObservers(i);
        }
    }
}

static class MyObserver implements Observer {
    private List<Integer> list;

    public MyObserver(List<Integer> list) {
        this.list = list;
    }

    @Override
    public void update(Observable o, Object arg) {
        this.list.add(Integer.parseInt(arg + ""));
    }
}

@Test
public void observable() throws Exception {
    // given
    MyObservable myObservable = new MyObservable(5);
    MyObserver myObserver = new MyObserver(actual);
    myObservable.addObserver(myObserver);

    // when
    List<Integer> actual = new LinkedList<>();
    Thread thread = new Thread(myObservable);
    thread.start();
    thread.join();

    // then
    assertThat(actual, hasSize(5));
    assertThat(actual, contains(1, 2, 3, 4, 5));
}
```  

`Iterable` 에서 진행한 예제와 동일하게 `1 ~ 5` 까지의 데이터를 `Observable` 이 생산하고, `Observer` 가 데이터 받아 처리해서 동일한 결과를 얻는 것을 확인 할 수 있다. 
여기서 `Iterable` 과 `Observable` 은 [Duality(쌍대성)](https://ko.wikipedia.org/wiki/%EC%8C%8D%EB%8C%80%EC%84%B1)
의 관계에 있다고 할 수 있다.  

하지만 `Iterable` 과 `Observable` 에는 큰 차이가 있다. 
이는 `Iterable` 은 앞서 언급했던 것처럼 `Pull` 방식으로 데이터를 처리자(소비자)가 원하는 시점에 데이터를 가져다가 사용할 수 있지만, 
`Observable` 은 `Push` 방식으로 데이터 생산자가 데이터 처리자(소비자)에게 데이터를 밀어서 주는 방식을 가지고 있다.  

이러한 특징으로 `Observable` 은 좀 더 다이나믹하게 데이터 혹은 이벤트에 대한 전파를 `Observer` 에게 수행할 수 있다. 
아래는 2개의 `Observer` 에게 데이터를 전파하는 예제이다. 

```java
 @Test
public void observable_multiple() throws  Exception {
    // given
    MyObservable myObservable = new MyObservable(5);
    MyObserver myObserver1 = new MyObserver(actual1);
    MyObserver myObserver2 = new MyObserver(actual2);
    myObservable.addObserver(myObserver1);
    myObservable.addObserver(myObserver2);

    // when
    List<Integer> actual1 = new LinkedList<>();
    List<Integer> actual2 = new LinkedList<>();
    Thread thread = new Thread(myObservable);
    thread.start();
    thread.join();

    // then
    assertThat(actual1, hasSize(5));
    assertThat(actual2, hasSize(5));
    assertThat(actual1, contains(1, 2, 3, 4, 5));
    assertThat(actual2, contains(1, 2, 3, 4, 5));
}
```  

위와 같이 `Observable` 의 데이터가 필요한 소비자를 `Observer` 로만 추가 등록해주면 간단하게 추가해서 전파할 수 있다.  

`Observable` 에서 알리는 상태변화나 이벤트, 데이터는 등록된 `Observer` 에서 전파되기 때문에 이 또한 `Reactive` 하다라고 할 수 있다.  
하지만 `Observable` 의 `Push` 방식은 데이터를 받아 처리하는 소비자 상태에 대한 고려 없이 수행된다는 큰 단점이 있다. 
만약 `Observable` 가 초당 100건의 데이터를 생산 가능한 스펙이라서 `Observer` 에게 전달 된다고 가정해 보자. 
하지만 각 `Observer` 마다 `Observable` 에서 전달된 데이터를 처리 가능한 스펙이 다를 것이고, 초당 100건 처리가 불가능 할 수 있다. 
`Observer` 가 `Observable` 이 데이터를 주는 속도를 따라가지 못한다면 버퍼를 두거나, 심각한 경우에는 데이터 유실이 발생할 수 있게 된다.  

>위에서 언급한 상황 해결할 수 있는 방법을 `Back-pressure` 라고 한다. 
>`Back-pressure` 는 시스템을 구성하는 컴포넌트들 간에 자신의 상황을 주고 받을 수 있는 피드백 시스템과 같다. 
>두 컴포넌트가 데이터를 주고 받으면서 처리가 수행 될때, 만약 데이터를 받는 쪽에서 부하가 걸린다면 데이터를 주는 쪽에서 적은 양의 데이터를 
>줌으로써 부하를 줄일 수 있도록 도와 주어야 한다. 
>반대로 데이터 받는 쪽에서 더 많은 데이터를 처리할 수 있다면 데이터를 주는 쪽에 더 많은 데이터를 요청할 수도 있다. 


## Java Reactive

### Reactive
[The Reactive Manifesto](https://www.reactivemanifesto.org/)
를 보면 `Reactive` 의 개념에 대해 파악할 수 있다.  

`Reactive` 이전 시스템 및 서비스의 경우 몇 십개의 서버 구성과 초단위 응답을 주는 스펙이러더라도 큰 이슈가 될 부분이 없었다. 
또 배포, 점검을 위한 다운타임에 대한 고려도 크지 않았고, 데이터의 사이즈 또한 `GB` 단위 수준이 였다.  

하지만 현재 대부분의 시스템을 보더라도 몇 백, 몇 천개의 서버가 수행되는 클라우드 환경에서 모든게 이뤄진다. 
그리고 초단위 응답시간은 사용자들로 하여금 큰 불편함을 주게 되었고, 서비스가 단 몇분만이라도 다운되면 큰 불만을 가지는 세상이 되었다. 
사용자가 많아지고 서비스 아키텍쳐가 복잡해진 만큼 데이터 사이즈 또한 `PB` 단위 수준이 되었다.  

서비스 규모의 급격한 발전과 전통적인 소프트웨어 아키텍쳐의 한계를 극복해서 위 요구사항을 만족시키기 위해, 
`Reactive` 시스템이 등장하게 되었다. 
`Reactive` 시스템은 아래와 같은 특징이 있다. (더 자세한 내용은 선언문 링크를 참조한다.)
- `Responsive`(응답성)
- `Resilient`(탄력성)
- `Elastic`(유연성)
- `Message Driven`(메시지 중심)

선언문에 있는 이미지로 `Reactive` 를 표현하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/concept-java-reactive-before-and-after-1.svg)  

### Java Reactive Streams




---
## Reference
[Reactive Streams](https://www.reactive-streams.org/)  
[reactive-streams/reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm)  
[Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/index.html)  
[The Reactive Manifesto](https://www.reactivemanifesto.org/)  

