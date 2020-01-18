--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Prototype Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Prototype
---  


## Prototype 패턴이란
- 일반적으로 인스턴스를 생성할때 `new SomeClass();` 와 같은 식으로 클래스 이름을 지정해서 생성해야 한다.(Java 의 경우)
- 아래는 클래스 이름을 바탕으로 인스턴스를 생성하지 않고, 인스턴스를 복사해 새로운 인스턴스를 만들어야 하는 경우 이다.
	- 아주 많은 종류가 있어 클래스로 정의가 힘든 경우
		- 반지름이 1인 원, 반지름이 2인원, 반지름이 3인 원 ...
	- 인스턴스 생성에 큰 비용이 들때
		- 인스턴스 생성을 할때 많은 요청의 응답을 사용하는 경우
	- 인스턴스 생성이 복잡할 경우
		- 사용자 인터페이스를 통해 그린 원을 복사해서 원 인스턴스를 만들어야 하는 경우
	- 같은 속성을 가진 객체를 빈번하게 복사해야 할 경우
- `Prototype` 패턴은 원형, 모범 이라는 의미를 가진 것처럼, 만들어야 하는 인스턴스의 원형, 모범이 되는 인스턴스를 기반으로 인스턴스를 만드는 패턴이다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_prototype_1.png)

- 패턴의 구성요소
	- `Prototype` : 인스턴스를 복사해서 새로운 인스턴스를 만들기 위한 메소드가 정의된 인터페이스이다. (추상 클래스도 가능하다)
	- `ConcretePrototype` : `Prototype` 을 구현 또는 상속하는 클래스로, 인스턴스를 만드는 메소드를 실제로 구현한다.
	- `Client` : `Prototype` 의 인스턴스를 만드는 메서드를 사용해서 새로운 인스턴스를 만드는 클래스이다.
- `Prototype` 패턴은 인스턴스의 생성을 `framework`(`prototype`), 실제 생성하는(`ConcretePrototype`) 부분을 나눠서 구성한다.
	- 이후 예제에서 확인 할 수 있겠지만, 

---
## Reference

	