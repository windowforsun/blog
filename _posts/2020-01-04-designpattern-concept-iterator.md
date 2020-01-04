--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Iterator Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '반복자 역할을 하는 Iterator 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - DesignPattern
tags:
  - DesignPattern
  - Iterator
---  


## Iterator 패턴이란
- 반복자라는 의미에서 유추할 수 있듯이, 여러개의 데이터에 대해서 순새대로 지정하면서 전체를 검색할 수 있도록 하는 패턴이다.
- 아래와 같은 전체 탐색 반복문에서 인덱스 역할을 하는 `i` 를 좀 더 추상화해서 일반화한 디자인 패턴이다.

	```java
	for(int i = 0; i < array.length; i++) {
		// do something by array[i]
	}
	```  


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_iterator_1.png)

- 패턴의 구성 요소
	- `Iterator` : 요소를 순서대로 검색하는 인터페이로 관련 메서드가 선언되어 있다.
	- `ConcreteIterator` : `Iterator` 인터페이스의 구현체로, 검색에 필요한 데이터를 통해 실제 검색 동작을 수행한다.
	- `Aggregate` : `Iterator` 인터페이스를 만들어내는 메서드가 선언되어 있는 인터페이스이다.
	- `ConcreteAggregate` : `Aggregate` 인터페이스의 구현체로, 검색하고자 하는 데이터들의 `ConcreteIterator` 를 만든다.

## 전화번호부 전체 검색

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_iterator_2.png)


---
## Reference
