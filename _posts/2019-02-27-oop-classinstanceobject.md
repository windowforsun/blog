--- 
layout: single
classes: wide
title: "클래스, 객체 그리고 인스턴스의 차이(관계)"
header:
  overlay_image: /img/oop-bg.jpg
subtitle: '클래스, 객체, 인스턴스는 개념적으로 어떤 차이점(관계)을 보이는지'
author: "window_for_sun"
header-style: text
categories :
  - OOP
tags:
    - OOP
    - Class
    - Object
    - Instance
---  

'클래스, 객체, 인스턴스는 개념적으로 어떤 차이점(관계)을 보이는지'

## 객체(Object) 란
- 구현해야할 **대상**
- 클래스를 통해 만들어진 대상의 **실체**
- 클래스의 인스턴스
- 인스턴스를 대표하는 포괄적인 의미

## 클래스(Class) 란
- 특정 대상을 나누는 기준 즉 **분류**(Classification)하는 것
- 특정 대상이 **소속** 될수 있는 것
- 특정 대상의 **종류**
- 일종의 **집합**적인 개념
- 연관되어 있는 변수와 메서드의 **집합**
- 객체를 만들기 위한 **설계도** 혹은 틀

## 인스턴스(Instance) 란
- 특정 대상이 **실체화**가 된것
- 클래스를 바탕으로 객체를 **실체화**를 한것
- 실체화란 소프트웨어상에서 사용을 위해 메모리에 할당된 객체
- 인스턴스는 객체에 포함될 수 있음
- 추상적인 개념(또는 명세)과 구체적인 객체 사이의 **관계**에 초점을 맞출 때 사용
	- 객체는 클래스의 인스턴스다.
	- 실행 프로세스는 프로그램의 인스턴스다.
	
## 객체, 클래스, 인스턴스의 관계
![객체 클래스 인스턴스 관계1]({{site.baseurl}}/img/oop-classobjectinstance-relation-1-diagram.png)

- 사람이라는 특정 **대상** 즉 객체(Object)가 있다.
- 사람이라는 대상의 특징을 **분류**한다.
- 분류를 통해 구체적인 **설계도** 즉 클래스로 만든다.

```java
class Person {
	private String name;
	private int age;
	
	public Person(String name, int age) {
		this.name = name;
		this.age = age;
	}
	
	public void speak() {
		System.out.println("Hi");
	}
}
```  

- **설계도**는 속성(이름, 나이)와 기능(말하기) 로 구성되어 있다.
- **설계도**를 사용하여 대상을 **실체화** 한다.

```java
Person cs = new Person("CS", 20);
```  

- 위  코드에서는 Java문법의 new 키워드를 사용하여 **실체화**를 하였다.
- **실체화**를 통해 cs라는 이름을 가진 **실체** 즉 인스턴스를 만들었다.
- 대상(Object) -> 분류 및 설계도(Class) -> 실체화를 통한 실체(Instance)


---
## Reference
[클래스 객체 인스턴스](https://botbinoo.tistory.com/30)  
[객체, 클래스, 인스턴스의 차이점은?](https://www.slipp.net/questions/126)  
[[Java] 클래스, 객체, 인스턴스의 차이](https://gmlwjd9405.github.io/2018/09/17/class-object-instance.html)  
[[OOP] OOP 에서의 클래스와 인스턴스 개념](https://www.youtube.com/watch?v=8B2Wxks5Sig)  