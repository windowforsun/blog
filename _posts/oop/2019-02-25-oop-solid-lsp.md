--- 
layout: single
classes: wide
title: "객체 지향 프로그래밍(OOP) 객체 지향 디자인 5원칙(SOLID 원칙) - LSP"
header:
  overlay_image: /img/oop-bg.jpg
excerpt: 'SOLID-LSP 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - OOP
tags:
    - OOP
    - LSP
    - 객체 지향
    - SOLID
    - 디자인 패턴
    - Design Pattern
---  

### [SOLID 개요]({{site.baseurl}}{% link _posts/oop/2019-02-24-oop-solid.md %})

## L - LSP(Liskov Substitution Principle) 리스코프 치환 원칙
> *참고  
> 인터페이스 상속 : 순수 가상 함수의 상속
> 구현 상속 : 가상 함수와 비가상 함수의 상속  
> 순수 가상 : 가상함수나, 함수의 정의부분이 없고, 선언 부분만 있는 함수  
> 가상 함수 : 파생 클래스에서 가상 함수가 없다면, 기본 클래스 함수가 호출되고, 있다면 파생 클래스의 가상 함수를 호출시켜주는 매체가 되는 함수  
> 비 가상 함수 : 일반 멤버 함수  
> 사전 조건(Precondition) : 해당 연산이 호출되기 전 만족해야하는 조건이며 이 조건이 만족하지 않을 시 연산을 호출하면 안된다는 의미  
> 사후 조건(Postcondition) : 연산이 수행되고 나서 반드시 만족해야하는 조건을 의미한다.  

1. 정의
	- 서브 타입은 언제나 기반 타입으로 교체할 수 있어야 한다.
	- 서브 타입은 기반 타입과 호환될 수 있어야 한다.
	- 상속 구현 상속(extends), 인터페이스 상속(implements)은 궁극적으로 다형성을 통한 확장성 확득이 목표이다.
	- LSP 원리도 하위 클래스가 확장에 대한 상위 클래스를 준수해야 함을 의미한다.
	- 다형성과 확장성을 극대화 하려면 하위 클래스 사용보다는 상위 클래스 사용이 더 좋다.
	- 상속을 통한 재사용은 상위 클래스와 서브 클래스 사이에 IS-A관계가 있을 경우로만 제한 되어야 한다.
	- IS-A 관계 아닐 경우에는 합성(Composition)을 이용한 재사용을 해야 한다.
	- LSP는 OCP를 구상하는 구조가 되어, 다형성을 통한 확장 원리인 OCP를 제공한다.
		- 상속은 다형성과 따로 생각할 수 없고, 다형성을 위해서는 하위 클래스가 상위 클래스의 규약을 지켜야 하기때문이다.
	- 객체 지향 설계원리는 LSP를 통해 OCP를 제공하는 것처럼 서로가 서로를 이용하고 포함하는 특징을 가지고 있다.
1. 적용 방법
	- 두 개체가 같은 일을 한다면 둘을 하나의 클래스로 표현하고 이들을 구분할 수 있는 필드를 둔다.
	- 같은 연산을 제공하지만, 다르게 동작한다면 공통 인터페이스를 만들고 두 개체가 이를 구현한다. (인터페이스 상속)
	- 공통 연산이 없다면, 완전 별개인 2개의 클래스를 만든다.
	- 두 개체가 하는 일에 추가적인 연산이 필요하다면 구현 상속을 사용한다.
1. 예시
	- 컬렉션 프레임워크
		```java
		public class MyArray {
			private LinkedList array = new LinkedList();
			
			public void add(Object o) {
				this.array.add(o);
			}
		}
		```  
		
		- MyArray에서 List만 사용할 것이라면 문제가 없다.
		- 속도 개선을 위해 HashSet을 사용해야 하는 경우 처럼 변경이 잦다면, LinkedList와 HashSet이 모두 상속하고 있는 Collection 으로 변경하는 것이 바람직하다.
		
		```java
		public class MyArray {
			private Collection array = new HashSet();
			
			public void add(Object o) {
				this.array.add(0);
			}
		}
		```  
		
		- 위 코드는 LSP와 OCP 모두 찾아볼 수 있다.
		- 컬렉션 프레임워크가 LSP를 준수하지 않았다면 위 코드처럼 범용적 작업은 불가능하다. (LSP)
		- add() 메서드는 변화에 닫혀 있으면서, 컬력션의 변경과 확장에는 열려 있는 구조이다. (OCP)
1. 적용 이슈 
	- 트레이드 오프를 고려해야 한다면 그대로 둔다.
	- 다형성을 위한 상속 관계가 필요 없다면 Replace With Delegation 을 이용한다.
		- 상속은 깨지기 쉬운 기반 클래스 등을 포함하고 있으므로 IS-A 관계가 성립되지 않는다.
		- LSP 를 지키기 힘들다면 상속대신 합성(Composition)을 사용하는 것이 좋다.
	- 상속 구조가 필요 하다면 Extract SubClass, Push Down Field, Push Down Method 등의 리팩토링 기법을 이용하여 LSP 를 준수하는 사아속 계층 구조를 구성한다.
	- IS-A 관계 맺음은 이들의 역할과 이들 사이에 공유하는 연산이 있는지, 그리고 이들 연산이 어떻게 다른지 등 종합적으로 검토 해야 한다.
	- Design By Contract : 하위 클래스에서는 상위 클래스의 사전 조건과 같거나 더 약한 수준에서 사전 조건을 대체할 수 있고, 상위 클래스의 사후 조건과 같거나 더 강한 수준에서 사후 조건을 대체 할 수 있다.
		- 하위 클래스를 상위 클래스로 치환 가능하게 하려면, 사전 조건에서 하위 클래스의 제약사항이 상위 클래스의 제약 사항보다 느슨하거나 같아야 한다.
		- 하위 클래스의 사전 조건의 제약조건이 더 강하면 상위 클래스에서 실행되는 것이 하위 클래스의 강 조건으로 인해 실행되지 않을 수 있다.
		- 하위 클래스의 사후 조건은 같거나 더 강해야 하는데, 약하다면 상위 클래스의 사후 조건이 통과시키지 않는 상태를 통과 시킬 수 있기 때문이다.
	

---
## Reference
[객체지향 개발 5대 원리: SOLID](http://www.nextree.co.kr/p6960/)  