--- 
layout: single
classes: wide
title: "[Java 개념] Parallel Stream 과 Fork/Join Framework"
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
    - ForkJoinPool
    - Fork/Join Framework
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

컬렉션이나 배열과 같은 데이터셋에 병렬성을 부여하기 위해서는 데이터셋이 하나의 병목지점이 될 수 있다. 
이는 데이터셋이 위치하는 메모리 데이터에 일관성을 해치지않고 병렬로 데이터셋을 조작하기에는 어려움이 있다는 것을 의미한다. 
자바나 컬렉션에서 제공하는 동기화처리를 바탕으로 데이터의 일관성을 부여하더라도, 
동기화 처리 부분은 구현에 따라 크거나 작게 동시성을 떨어뜨릴 수 있다. 
여기서 `Parallel Stream` 을 사용하면 데이터셋 수정이 없다는 가정하에, 
동기화 처리없이 병렬처리를 구현할 수 있다.  

그리고 `Parallel Stream` 은 `JVM` 에서 글로벌한 `Thread Pool` 을 사용한다는 점을 기억해야 한다. 
하개의 처리를 `Parallel Stream` 으로 성능을 향상 시켰더라도, 
실제 운영에 환경에서는 이로인해 전체적인 성능 저하가 발생할 수 있다는 의미이다.  


### Fork/Join Framework
`Fork/Join Framework` 는 멀티 프로세서를 활용할 수 있도록 지원하는 `ExecutorService` 인터페이스의 구현체로 구성된다. 
하나의 큰 문제를 분할 가능한 작은 문제로 반복해서 분할해서, 작은 문제를 해결하고 그 결과를 합쳐 큰 문제를 해결 및 결과를 도출하는 방법을 사용한다. 
이는 분할정복 알고리즘과 비슷한 성격을 띈다. 
그리고 시스템에서 사용가능한 모든 프로세서의 자원을 최대한으로 활용해서 문제 해결에 대한 성능을 향상시키는 것을 목표로 하고 있다.  

`For/Join` 의 과정을 나열하면 아래와 같다. 
1. 큰 문제를 작은 단위로 분할 한다. 
1. 부모 스레드로 부터 처리로직을 복사해서 새로운 스레드에 분할된 문제를 수행(`Fork`) 시킨다. 
1. 더 이상 `Fork` 가 일어나지 않고, 분할된 모든 문제가 완료될 때까지 위 과정을 반복한다. 
1. 분할된 모든 문제가 완료되면, 분할된 문제의 결과를 `Join` 해서 취합한다. 
1. 사용한 모든 스레드에 대해 위 과정을 반복하면서 큰 문제의 결과를 도출해 낸다. 

위 과정을 그림으로 도식화 하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept_parallelstream_forkjoin_1.png)  


`ExecutorService` 를 사용한 다른 구현체와 비슷하게 스레드에 처리할 작업을 할당한다. 
차이점이 있다면 `Fork/Join` 은 [work-stealing](https://en.wikipedia.org/wiki/Work_stealing)
알고리즘을 사용해서 `ExecutorService` 를 구현했기 때문에 노는 스레드 없이 모든 스레드가 함께 처리를 계속해서 문제를 빠르게 해결한다. 

![그림 1]({{site.baseurl}}/img/java/concept_parallelstream_forkjoin_2.png)  

1. 앞서 설명한 것처럼 큰 문제를 작은 문제(작업)로 분할 한다. 
1. 분할된 작업은 `ForkJoinPool` 에서 관리하는 `inbound queue` 에 `submit` 한다. 
1. 할당된 스레드들에서 `inbound queue` 에 있는 작업을 `take` 한다. 
1. 스레드는 `take` 한 작업을 각 스레드에서 관리하는 `deque` 에 `push` 한다. 
1. 각 스레드는 자신의 `deque` 에 있는 작업을 `pop` 하며 분할된 작업을 계속해서 처리한다. 
1. 만약 스레드가 자신의 `deque` 의 작업도 모두 처리하고 `inbound queue` 에도 작업이 없다면, 
다른 스레드의 `deque` 에서 작업을 `steal` 해서 처리한다. 

위와 같이 `Fork/Join` 은 하나의 큰 작업이 모두 완료되기 전까지 할당된 스레드들이 `Idle` 한 상태로 대기하지 않고, 
분할된 모든 작업을 빠른 시간내에 처리하는 방법을 사용한다.  

`Fork/Join Framework` 를 구성하는 주요 클래스는 아래와 같다. 
- `ForkJoinPool` : `Fork/Join` 방식으로 분할 및 정복을 수행하는 `Fork/Join Framework` 의 메인 클래스이면서 `ExecutorService` 의 구현체이다. 
- `RecursiveTask<V>` : 결과가 존재하는 작업으로, 실제 작업은 해당 클래스를 상속해서 `compute` 메소드에 처리과정을 구현하고 결과를 리턴한다. 
- `RecursiveAction` : 결과가 존재하지 않는 작업으로 작업은 해당 클래스를 상속해서  `compute` 메소드에  처리과정을 구현한다. 
- `ForkJoinTask<V>` : `RecursiveTask`, `RecursiveAction` 의 부모 클래스로 `fork`, `join` 메소드가 정의돼 있고, `Future` 의 구현체이다. 



























































---
## Reference
[Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)  
[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)  
[Fork and Join: Java Can Excel at Painless Parallel Programming Too!](https://www.oracle.com/technical-resources/articles/java/fork-join.html)  
[The fork/join framework in Java 7](http://www.h-online.com/developer/features/The-fork-join-framework-in-Java-7-1762357.html)  
