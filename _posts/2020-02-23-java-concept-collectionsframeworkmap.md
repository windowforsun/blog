--- 
layout: single
classes: wide
title: "[Java 개념] Collections Framework - Map"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Collections Framework 중 Map 부분에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Map
    - Java Collection Framework
toc: true
use_math: true
---  

# Java Map

![그림 1]({{site.baseurl}}/img/java/concept_collectionsframeworkcollection_1.png)

- Java 에서 Map 은 `key-value` 형식으로 데이터를 저장하는 구조에서 효과적으로 처리할 수 있는 표준화된 방법을 제공하는 클래스의 집합니다.
- 위 그림은 Java Collections Framework 에서 Map 부분을 구성하고 있는 다이어그램이다.
- Map 을 구성하는 모든 인터페이스와 클래스는 Map 의 하위에 있다.
- Map 을 상속하는 하위 인터페이스들은 보다 구체적인 Map 구조의 인터페이스를 정의한다.
- 인터페이스를 구현하는 추상클래스들은 해당 Map 구조에서 공통적인 부분을 구현하거나, 보다 구체적으로 정의한다.
- 인터페이스 혹은 추상 클래스의 하위 클래스는 실질적인 Map 구조를 구현하는 클래스이다.

# Map
- Java Map 의 루트 인터페이스로 Map 동작에 필요한 공통적인 메소드가 정의돼 있다.
- Map 구조에서 `key` 는 중복된 값을 가질 수 없고, 각 `key` 는 대응되는 `value` 를 갖는다.
- Java 의 Map 구조는 Dictionary 구조를 대체할 수 있다.
- Map 은 기본적으로 데이터 조회를 위해 세가지 방식의 Collection 을 제공한다.
	- `Set` 형식의 `key` 집합을 받아, `value` 를 조회한다.
	- `Collection` 형식의 `value` 를 받는다.
	- `Set` 형식의 `key-value` 구조를 받는다.
- `Collection` 의 순서는 `iterator` 에서 정의된 것을 따르고, 몇가지(`TreeMap`)는 별도의 순서를 가질 수 있다.
- Map 에서 `key`, `value` 에 대한 값 제한은 하위 구현체에 따라 상이 할 수 있다.
	- 어떤 구현체에서는 `null` 값을 허용하지 않는다.
	- 어떤 구현체에서는 `key` 타입에 제한이 있다.

## AbstractMap

### EnumMap

### HashMap

#### LinkedHashMap

### IdentityHashMap

### WeakHashMap

## SortedSetMap
- `Map` 의 하위 인터페이스로 특정 기준으로 `key` 를 정렬한 Map 구조의 인터페이스이다.
- 정렬 기준에 대한 정의가 없을 경우 기본 `comparator` 를 기준으로 `key` 가 정렬된다.


## NavigableMap

### TreeMap






































































---
## Reference
[Hierarchy For Package java.util](https://docs.oracle.com/javase/8/docs/api/java/util/package-tree.html)  
[Java Collections – Performance (Time Complexity)](http://infotechgems.blogspot.com/2011/11/java-collections-performance-time.html)  
