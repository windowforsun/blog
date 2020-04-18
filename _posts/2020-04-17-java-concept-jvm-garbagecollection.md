--- 
layout: single
classes: wide
title: "[Java 개념] JVM Garbage Collection"
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
    - JVM
    - Garbage Collection
toc: true
use_math: true
---  

## Garbage Collection 이란
- `GC` 라고 축약해서 부르기도 한다.
- `GC` 는 이미 할당된 메모리에서 더 이상 사용되지 않는 메모리를 해제하는 동작을 의미한다.
- 여기서 사용되지 않는 메모리는 `Stack Area` 에서 참조하지 않는 메모리 영역을 의미한다.
- Java 에서 메모리를 해제하기 위해서는 변수에 `null` 을 설정하거나, `System.gc()` 를 호출 할 수도 있다.
	- `System.gc()` 메소드를 호출하는 것은 매우 주의해야 하는데, 이는 구동 중인 시스템에 성능적으로 큰 영향을 끼칠 수 있기 때문이다.
- `GC` 작업은 `stop-the-world` 라는 동작을 수행하게 되는데, 이는 `GC` 를 실행하기 위해 `JVM` 이 `GC` 를 담당하는 쓰레드외 모든 쓰레드의 작업을 멈추는 것을 의미한다.
	- `GC` 작업 이후에 중지됐던 쓰레드는 다시 실행된다.
- 애플리케이션이 구동된다는 것은 호스트 머신에서 필요한 자원을 할당받아 사용한다는 의미이고, Java 는 다른 언어(C, C++) 과 달리 메모리 관리를 개발자가 직접하지 않고 `GC` 가 이를 담당해 주기 때문에 이러한 작업이 필요하다.
- `GC` 튜닝이란 피할 수 없는 `stop-the-world` 시간을 줄이는 것을 의미한다.

## Garbage Collection 의 구성




---
## Reference
[Java Garbage Collection](https://d2.naver.com/helloworld/1329)  
[What is Java Garbage Collection? How It Works, Best Practices, Tutorials, and More](https://stackify.com/what-is-java-garbage-collection/)  
[Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)  
[Java - Garbage Collection(GC,가비지 컬렉션) 란?](https://coding-start.tistory.com/206)  