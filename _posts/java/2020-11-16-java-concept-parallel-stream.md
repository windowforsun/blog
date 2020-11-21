--- 
layout: single
classes: wide
title: "[Java 개념] Parallel Stream"
header:
  overlay_image: /img/java-bg.jpg
excerpt: '스트림 API 를 사용해서 병렬 처리를 수행하 수 있는 Parallel Stream 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Stream
    - Lamda
    - ParallelStream
toc: true
use_math: true
---  

## Parallel Stream
[여기]({{site.baseurl}}{% link _posts/java/2020-11-07-java-concept-stream.md %})
에서 `Stream` 에 대해 알아보았다. 
`Stream` 에서 `Parallel Stream` 을 사용하면 지금까지 복잡하게 구현해야 했던 
병렬처리 관련 연산을 간단하게 사용할 수 있다. 
`Parallel Stream` 은 문제를 작은 문제로 나누고 작은 문제를 병렬로 수행한다음 
해결된 작은 문제의 결과를 결합하는 방식으로 동작한다. 
`Parallel Stream` 은 내부적으로 [Fork/Join Framework](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)
를 사용해서 병렬 작업을 보다 쉽게 구현할 수 있도록 한다. 
`Fork/Join Framework` 관련 더 자세한 내용은 [여기]({{site.baseurl}}{% link _posts/java/2020-11-11-java-concept-fork-join-framework.md %})
에서 확인 할 수 있다. 

컬렉션이나 배열과 같은 데이터셋에 병렬성을 부여하기 위해서는 데이터셋이 하나의 병목지점이 될 수 있다. 
이는 데이터셋이 위치하는 메모리 데이터에 일관성을 해치지않고 병렬로 데이터셋을 조작하기에는 어려움이 있다는 것을 의미한다. 
자바나 컬렉션에서 제공하는 동기화처리를 바탕으로 데이터의 일관성을 부여하더라도, 
동기화 처리 부분은 구현에 따라 크거나 작게 동시성을 떨어뜨릴 수 있다. 
여기서 `Parallel Stream` 을 사용하면 데이터셋 수정이 없다는 가정하에, 
동기화 처리없이 병렬처리를 구현할 수 있다.  

그리고 `Parallel Stream` 은 `JVM` 에서 글로벌한 `Thread Pool` 을 사용한다는 점을 기억해야 한다. 
하개의 처리를 `Parallel Stream` 으로 성능을 향상 시켰더라도, 
실제 운영에 환경에서는 이로인해 전체적인 성능 저하가 발생할 수 있다는 의미이다.  

`Stream` 의 연산은 여러 종류의 `Iteration` 의 집합으로 구성된다. 
`filter`, `map` 등 `Stream` 에서 제공하는 모든 연산을 수행하기 위해서는 데이터 집합에 대해 `Iteration` 을 통해 작업을 처리한다. 
이를 `Sequential Stream` 과 `Parallel Stream` 을 비교해서 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept_parallelstream_1.png)

그림에서 알 수 있듯이 `Parallel Stream` 스트림의 데이터 셋을 작은 서브 셋으로 나누고, 
스트림에 정의된 연산을 각 서브 셋에서 수행하는 방식이다. 
여기서 서브 셋은 하나의 스레드와 매핑될 수 있을 것이다.  

### 사용방법 및 일반 Stream 과 비교
`Parallel Stream` 사용은 `Stream` 객체에서 `BaseStream.parallel()` 메소드를 호출하거나, 
`Collection` 객체에서 `Collection.parallelStream()` 메소드 호출등으로 생성할 수 있다. 

```java
public class SimpleParallelStreamTest {
    @Test
    public void sequentialStream() {
        // given
        Stream<Integer> stream = IntStream.iterate(1, (i) -> i + 1)
                .limit(10)
                .boxed();

        // when
        List<Integer> actual = new ArrayList<>();
        stream.forEach((i) -> actual.add(i));

        // then
        assertThat(actual, contains(
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10
        ));
    }

    @Test
    public void parallelStream() {
        // given
        Stream<Integer> stream = IntStream.iterate(1, (i) -> i + 1)
                .limit(10)
                .boxed();

        // when
        List<Integer> actual = new Vector<>();
        stream.parallel().forEach((i) -> actual.add(i));

        // then
        assertThat(actual, allOf(
                not(contains(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)),
                containsInAnyOrder(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        ));
    }
}
```  

위 테스트는 간단한 `Sequential Stream` 과 `Parallel Stream` 의 비교 예시이다. 
`[1 ~ 10]` 원소를 갖는 스트림을 생성하고 이를 `Stream.forEach()` 를 사용해서 `List` 에 추가하는 방식으로 테스트를 진행한다. 
`Sequential Stream` 은 1 ~ 10까지 순서대로 데이터가 들어간 상태지만, 
`Parallel STream` 은 1 ~ 10까지 모든 데이터가 추가된건 맞지만 순서가 뒤죽박죽인 것을 확인 할 수 있다.  

### 병렬 Reduction 연산
스트림의 요소를 그룹화 하는 `Reduction` 연산을 수행하는 상황을 살펴보자. 
`Parallel Stream` 에서 `Reduction` 의 결과는 `Map` 또는 `ConcurrentMap` 타입으로 리턴받을 수 있다. 

```java
public class ReductionParallelStream {
    private List<MyClass> list;
    private static final int EACH_COUNT = 200000;
    private static final int ALPHABET_COUNT = 'z' - 'a' + 1;

    @Before
    public void setUp() {
        this.list = new ArrayList<>();
        for(int i = 0; i < EACH_COUNT; i++) {
            for(int j = 0; j < ALPHABET_COUNT; j++) {
                list.add(new MyClass(i, Character.toString((char)('a' + j))));
            }
        }
    }

    @Test
    public void parallelStream_Reduction_ConcurrentMap() {
        // given
        Stream<MyClass> parallelStream = this.list.parallelStream();

        // when
        long start = System.currentTimeMillis();
        Map<String, List<MyClass>> actual = parallelStream
                .collect(Collectors.groupingByConcurrent(MyClass::getStr));
        long end = System.currentTimeMillis();

        // then
        System.out.println("concurrent map during : " + (end - start));
        for(int i = 0; i < ALPHABET_COUNT; i++) {
            String key = Character.toString((char)('a' + i));
            assertThat(actual, hasEntry(is(key), hasSize(EACH_COUNT)));
        }
    }

    @Test
    public void parallelStream_Reduction_Map() {
        // given
        Stream<MyClass> parallelStream = this.list.parallelStream();

        // when
        long start = System.currentTimeMillis();
        Map<String, List<MyClass>> actual = parallelStream
                .collect(Collectors.groupingBy(MyClass::getStr));
        long end = System.currentTimeMillis();

        // then
        System.out.println("map during : " + (end - start));
        for(int i = 0; i < ALPHABET_COUNT; i++) {
            String key = Character.toString((char)('a' + i));
            assertThat(actual, hasEntry(is(key), hasSize(EACH_COUNT)));
        }
    }
}
```  

멤버 변수로 선언된 `list` 에는 `MyClass` 의 멤버 변수인 `str` 의 값이 `a ~ z` 까지로 구성된다. 
그리고 하나의 알파벳당 중복되는 개수는 클래스 변수로 선언된 `EACH_COUNT` 의 값 만큼이다. 
그리고 이를 `MyClass::getStr` 을 기준으로 `Reduction` 연산을 수행하면, 
`a ~ z` 를 키로 `EACH_COUNT` 의 크기를 갖는 `List` 로 구성된 `Map` 이 결과값이 될 것이다.  

각 테스트를 살펴보면 한개는 `ConcurrentMap` 을 리턴하고, 한개는 `Map` 을 리턴하고 있다. 
여기서 실제로 `Reduction` 연산을 수행하는데 소요되는 시간을 측정하고 출력하면 아래와 같다. 

```
concurrent map during : 201
map during : 341
```  

`ConcurrentMap(groupingByConcurrent)` 을 사용하는 경우가 `Map(groupingBy)` 을 사용하는 경우보다 더욱 빠른 속도인 것을 확인 할 수 있다. 
그 이유는 `groupBy` 는 병렬로 수행된 두개의 `Map` 결과를 하나로 다시 합치는 과정에서 추가적인 비용이 사용되기 때문이다.  


### 처리 순서
스트림을 사용한 연산은 항상 스트림의 소스가 되는 배열 혹은 컬렉션의 순서대로 처리된다. 
하지만 `Parallel Stream` 을 사용하면 연산이 실제로 수행되는 순서는 개발자가 예측할 수 없다. 
여기서 `Parallel Stream` 을 사용하더라도 배열 혹은 컬렉션의 순서대로 처리가 필요한 경우 `forEachOrdered` 를 사용할 수 있다. 

```java
public class OrderingParallelStream {
    private List<Integer> list;

    @Before
    public void setUp() {
        this.list = IntStream
                .iterate(1, i -> i + 1)
                .limit(10000)
                .boxed()
                .collect(Collectors.toList());
    }

    @Test
    public void parallelStream_Iteration_Random_Ordering() {
        // given
        List<Integer> list1 = new Vector<>();
        List<Integer> list2 = new Vector<>();

        // when
        this.list.parallelStream().forEach(i -> list1.add(i));
        this.list.parallelStream().forEach(i -> list2.add(i));

        // then
        assertThat(list1, not(contains(list2.toArray())));
        assertThat(list1, containsInAnyOrder(this.list.toArray()));
        assertThat(list2, containsInAnyOrder(this.list.toArray()));
    }

    @Test
    public void parallelStream_Iteration_Ordered() {
        // given
        List<Integer> list1 = new Vector<>();
        List<Integer> list2 = new Vector<>();

        // when
        this.list.parallelStream().forEachOrdered(i -> list1.add(i));
        this.list.parallelStream().forEachOrdered(i -> list2.add(i));

        // then
        assertThat(list1, contains(list2.toArray()));
        assertThat(list1, contains(this.list.toArray()));
        assertThat(list2, contains(this.list.toArray()));
    }

    @Test
    public void parallelStream_forEach_Faster_Than_forEachOrdered() {
        // given
        long start, end, duringForEach, duringForEachOrdered;

        // when
        start = System.currentTimeMillis();
        this.list.parallelStream().forEach(i -> this.sleepNano(1000));
        end = System.currentTimeMillis();
        duringForEach = end - start;
        start = System.currentTimeMillis();
        this.list.parallelStream().forEachOrdered(i -> this.sleepNano(1000));
        end = System.currentTimeMillis();
        duringForEachOrdered = end - start;

        // then
        assertThat(duringForEach, lessThan(duringForEachOrdered));
    }

    public void sleepNano(long nano) {
        long end = System.nanoTime() + nano;

        while(System.nanoTime() < end) {}
    }
}
```  

테스트는 `1~10000` 까지의 요소를 담고 있는 `List` 를 `forEach` 혹은 `forEachOrdered` 를 통해 순회하며, 
새로운 `List` 에 추가해서 추가된 순서를 바탕으로 순서가 유지됐는지 검증한다. 
`forEach` 를 사용해서 추가한 `list1`, `list2` 의 순서도 서로 다를 뿐만 아니라 `list1`, `list2` 모두 원래 순서인 `list` 의 순서가 아닌 것을 확인 할 수 있다.  

하지만 `forEachOrdered` 를 사용해서 추가한 `list1`, `list2` 의 경우 두 순서도 같고 `list` 와의 순서도 모두 같은 것을 확인 할 수 있다.  

`Parallel Stream` 에서 원래의 소스의 순서를 유지하며 연산을 수행한다는 것을 병목지점을 갖는다는 의미와 크게 다르지 않다. 
이러한 이유로 실제 테스트를 수행하면 `forEachOrdered` 가 `forEeach` 보다 성능적으로는 좋지않은 것도 확인 가능하다.  


### 스트림 소스 간섭
스트림 연산을 수행하면서 스트림 소스를 수정(간섭) 하는 행위는 매우 위험할 수 있다. 
파이프라인 스트림을 처리하는 동안 스트림 소스가 수정되면 간섭으로 인해 `ConcurrentModification` 예외가 발생할 수 있다. 

```java
public class InterferenceParallelStream {
    private List<Integer> list;

    @Before
    public void setUp() {
        this.list = new ArrayList<>(Arrays.asList(
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10
        ));
    }

    @Test(expected = ConcurrentModificationException.class)
    public void sequentialStream_Interference_Intermediate_Operation() {
        // given
        Stream<Integer> sequentialStream = this.list.stream();

        // when
        sequentialStream
                .peek(i -> this.list.add(11111))
                .forEach(i -> {});

    }

    @Test(expected = ConcurrentModificationException.class)
    public void parallelStream_Interference_Intermediate_Operation() {
        // given
        Stream<Integer> parallelStream = this.list.parallelStream();

        // when
        parallelStream
                .peek(i -> this.list.add(11111))
                .forEach(i -> {});
    }

    @Test(expected = ConcurrentModificationException.class)
    public void sequentialStream_Interference_Terminal_Operation() {
        // given
        Stream<Integer> sequentialStream = this.list.stream();

        // when
        sequentialStream
                .forEach(i -> this.list.add(22222));

    }

    @Test(expected = ConcurrentModificationException.class)
    public void parallelStream_Interference_Terminal_Operation() {
        // given
        Stream<Integer> parallelStream = this.list.parallelStream();

        // when
        parallelStream
                .forEach(i -> this.list.add(22222));
    }

    @Test
    public void parallelStream_LinkedList_Interference_Intermediate_Operation() {
        // given
        LinkedList<Integer> linkedList = new LinkedList<>(this.list);
        Stream<Integer> parallelStream = linkedList.parallelStream();

        // when
        parallelStream
                .peek(i -> linkedList.add(22222))
                .forEach(i -> {});

        // then
        assertThat(linkedList, hasItem(22222));
    }

    @Test
    public void parallelStream_LinkedList_Interference_Terminal_Operation() {
        // given
        LinkedList<Integer> linkedList = new LinkedList<>(this.list);
        Stream<Integer> parallelStream = linkedList.parallelStream();

        // when
        parallelStream
                .peek(i -> {})
                .forEach(i -> linkedList.add(22222));

        // then
        assertThat(linkedList, hasItem(22222));
    }
}
```  

파이프라인 스트림을 처리하는 동안 스트림 소스를 수정하게 될경우 발생하는 예외는 `Stream`, `Paralell Stream` 모두 동일하다. 
하지만 `LinkedList` 가 스트림 소스인 경우, 
`Intermediate Operation` 에서 수정하거나 `Terminal Operation` 에서 스트림 소스를 수정하더라도 예외가 발생하지 않는 다는 점을 주의해야 한다.  

### Stateful Lamda
`Stream` 연산에서 `Stateless` 성격을 가진 연산으로는 `filter()`, `map()`, `flatMap()` 등이 있다. 
그리고 `Stateful` 성격을 가진 연산으로는 `distict()`, `limit()`, `sorted()`, `reduce()`, `collect()` 가 있다.  
여기서 `Stateless` 와 `Stateful` 의 차이는 처리 순서에 따라 결과가 달라지느냐 달라지지 않느냐로 볼 수 있다.  

특히 `Parallel Stream` 을 사용할때 `Stateless` 성격을 지닌 연산에 `Statefule` 한 `Lamda` 식을 사용해서는 안된다. 

```java
public class StatefulLamdaParallelStream {
    private List<Integer> list;

    @Before
    public void setUp() {
        this.list = IntStream
                .iterate(1, i -> i + 1)
                .limit(10)
                .boxed()
                .collect(Collectors.toList());
    }

    @Test
    public void sequentialStream() {
        // given
        List<Integer> intermediateVector = new Vector<>();
        List<Integer> terminalVector = new Vector<>();

        // when
        this.list
                .stream()
                .map(i -> {
                    intermediateVector.add(i);
                    return i;
                })
                .forEachOrdered(i -> terminalVector.add(i));

        // then
        assertThat(terminalVector, contains(intermediateVector.toArray()));
        assertThat(terminalVector, contains(this.list.toArray()));
        assertThat(intermediateVector, contains(this.list.toArray()));
    }

    @Test
    public void parallelStream() {
        // given
        List<Integer> intermediateVector = new Vector<>();
        List<Integer> terminalVector = new Vector<>();

        // when
        this.list
                .parallelStream()
                .map(i -> {
                    intermediateVector.add(i);
                    return i;
                })
                .forEachOrdered(i -> terminalVector.add(i));

        // then
        assertThat(terminalVector, allOf(
                not(contains(intermediateVector.toArray())),
                containsInAnyOrder(intermediateVector.toArray())
        ));
        assertThat(terminalVector, contains(this.list.toArray()));
        assertThat(intermediateVector, allOf(
                not(contains(this.list.toArray())),
                containsInAnyOrder(this.list.toArray())
        ));
    }
}
```  

테스트 코드에서 `Stateless` 성격을 지닌 `map()` 메소드에 스트림 요소를 외부 리스트에 추가하는 `Stateful` 한 `Lamda` 를 정의 했다. 
특히 `Parallel Stream` 에서는 위와 같은 코드는 큰 의미를 가질 수 없다. 
이는 실행 될때마다 결과가 달라지고, `Stateless` 한 연산에 `Stateful` 한 성격의 식을 기술하는 것은 위험할 수 있다. 
`Stateless` 는 일반 스트림에서 병렬 스트림으로 전환할때 각 요소는 독립적으로 처리 가능하기 때문에 큰 문제가 되지 않지만, 
`Stateful` 은 각 요소가 독립적으로 처리될 수 없어 좀 더 복잡한 처리가 필요하기 때문이다. 

---
## Reference
[Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)  