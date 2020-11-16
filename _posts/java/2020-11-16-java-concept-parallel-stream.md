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

컬렉션이나 배열과 같은 데이터셋에 병렬성을 부여하기 위해서는 데이터셋이 하나의 병목지점이 될 수 있다. 
이는 데이터셋이 위치하는 메모리 데이터에 일관성을 해치지않고 병렬로 데이터셋을 조작하기에는 어려움이 있다는 것을 의미한다. 
자바나 컬렉션에서 제공하는 동기화처리를 바탕으로 데이터의 일관성을 부여하더라도, 
동기화 처리 부분은 구현에 따라 크거나 작게 동시성을 떨어뜨릴 수 있다. 
여기서 `Parallel Stream` 을 사용하면 데이터셋 수정이 없다는 가정하에, 
동기화 처리없이 병렬처리를 구현할 수 있다.  

그리고 `Parallel Stream` 은 `JVM` 에서 글로벌한 `Thread Pool` 을 사용한다는 점을 기억해야 한다. 
하개의 처리를 `Parallel Stream` 으로 성능을 향상 시켰더라도, 
실제 운영에 환경에서는 이로인해 전체적인 성능 저하가 발생할 수 있다는 의미이다.






































---
## Reference
[Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)  
[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)  
[Fork and Join: Java Can Excel at Painless Parallel Programming Too!](https://www.oracle.com/technical-resources/articles/java/fork-join.html)  
[The fork/join framework in Java 7](http://www.h-online.com/developer/features/The-fork-join-framework-in-Java-7-1762357.html)  