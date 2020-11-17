--- 
layout: single
classes: wide
title: "[Java 개념] Parallel Stream"
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

### 사용 및 일반 Stream 과 비교
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





































---
## Reference
[Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)  
[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)  
[Fork and Join: Java Can Excel at Painless Parallel Programming Too!](https://www.oracle.com/technical-resources/articles/java/fork-join.html)  
[The fork/join framework in Java 7](http://www.h-online.com/developer/features/The-fork-join-framework-in-Java-7-1762357.html)  