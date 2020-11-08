--- 
layout: single
classes: wide
title: "[Java 개념] Stream API"
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
    - Test
    - Mockito
toc: true
use_math: true
---  

## Stream
스트림(`Stream`)은 선언형으로 컬렉션이나 배열 데이터를 처리할 수 있다. 
컬렉션, 배열 데이터를 `for` 나 `foreach` 를 사용해서 하나씩 조회하며 다루는 것이 아닌 
조금 더 데이터 흐름에 초점을 맞춰 우아하게 처리를 명시할 수 있다. 
그리고 스트림을 사용하면 멀티 스레드관련 처리를 구현하지 않아도 데이터를 병렬로 처리 할 수 있다.  

스트림의 특징을 정리하면 아래와 같은 것들이 있다. 
- `No Storage` : 스트림은 저장소나 데이터 구조체가 아닌 
배열, 컬렉션, `I/O` 채널 과같은 소스의 요소들을 연산 파이프라인으로 전달하는 역할을 한다. 
- `Functional in nature` : 스트림을 통해 수행되는 작업은 결과를 생성하지만 그 소스(배열, 컬렉션 ..)를 수정하지 않는다. 
스트림을 사용해서 필터링 작업을 수행한다면, 스트림은 소스는 수정하지않고 필터링에 해당하는 요소만 포함되는 새로운 스트림을 생성한다. 
- `Laziness-seeking` : 스트림을 사용하면 다양한 조건을 통해 소스를 탐색, 필터링, 제거 연산을 수행하게 된다. 
여기서 스트림은 이러한 연산을 `Lazy` 하게 처리한다. 
사용자가 데이터를 요청할 때만 소스를 바탕으로 데이터 처리를 하는 방식을 사용해서, 
불필요한 연산을 최소화 한다. 
- `Possibly unbounded` : 컬렉션, 배열은 크기가 정해져 있지만 스트림은 무한한 크기를 가질 수 있다. 
`Short-circuiting` 에는 `limit()`, `findFirst()` 와 같은 것들이 있는데, 
무한한 크기의 스트림에서 유한시간 내에 연산을 완료할 수 있도록 한다. 
- `Consumable` : 스트림은 일회용의 성격을 갖는다. 
스트림의 요소들은 단 한번 방문 가능하고 이는 `Iterator` 의 성격과 같다. 
만약 다시한번 요소를 방문해야 한다면, 새로운 스트림을 생성해야 한다. 

스트림과 컬렉션은 모두 순차적으로 데이터를 순회하는 기능을 제공한다. 
하지만 몇가지 차이점이 존재하는데, 
첫번째로는 연산시점이 있다. 
앞서 설명한 것처럼 스트림은 연산을 `Lazy` 하게 처리한다. 
하지만 컬렉션은 연산을 위해서는 연산하 데이터들이 모두 메모리에 저장되어야 한다. 

두번째로는 외부반복과 내부반복이 있다. 
... 서술 ....





---
## Reference
[Package java.util.stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)  
[Stream (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)  
