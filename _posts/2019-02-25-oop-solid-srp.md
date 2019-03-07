--- 
layout: single
classes: wide
title: "객체 지향 프로그래밍(OOP) 객체 지향 디자인 5원칙(SOLID 원칙) - SRP"
header:
  overlay_image: /img/oop-bg.jpg
excerpt: 'SOLID-SRP 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - OOP
tags:
    - OOP
    - SRP
    - 객체 지향
    - SOLID
    - 디자인 패턴
    - Design Pattern
---  

### [SOLID 개요]({{site.baseurl}}{% link _posts/2019-02-24-oop-solid.md %})

## S - SRP(Single Responsibility Principle) 단일 책임 원칙
1. 정의
	- 작성된 클래스는 하나의 기능만 가지며 클래스가 제공하는 모든 서비스는 그 하나의 책임을 수행하는 데 집중되어 있어야 한다는 원칙
	- 어떤 변화에 의해 클래스를 변겨여해야 하는 이유는 오직 하나뿐이어야 함을 의미
	- SRP원리를 적용하면 무엇보다도 책임 영역이 확실해지기 때문에 한 책임의 변경에서 다른 책임의 변경으로의 연쇄작용에서 자유로움
	- 책임을 적절히 분배함으로써 코드의 가독성 향상, 유지보수 용이라는 이점을 누릴 수 있고, 객체 지향 원리의 OCP원리뿐 아니라 다른 원리들의 기초가 됨
	- 실무 프로세스는 매우 복잡 다양하고 변경 또한 빈번하기 때문에 간단하지만 직접 적용 및 설계가 어려움
	- 평소에 많은 연습(책임이란 단어를 상기하는)과 경험이 필요함
1. 적용 방법
	- 리팩토링(Refactoring)에서 대부분의 상황에 대한 해결방법은 직/간접/적으로 SRP원리와 관련이 있으며, 이는 항상 코드를 최상으로 유지한다는 리팩토링의 근본정신도 항상 객체들의 책임을 최상위 상태로 분배한다는 것에서 비롯 된다.
		- 여러 원인에 의한 변경(Divergent Change)
			- Extract Class를 통해 혼재된 각 책임을 각각의 개별 클래스로 분할하여 클래스 당 하나의 책임만을 맡도록 하는 것이다.
			- 책임만 분리하는 것이 아니라, 분리된 두 클래스간의 관계 복잡도를 줄이도록 설계해야 한다.
			- Extract Class 된 각각의 클래스들이 유사하고 비슷한 책임을 중복해서 갖고 있다면 Extract SuperClass를 사용할 수 있다.
				- Extract 된 각각의 클래스들에서 공유되는 요소를 부모 클래스로 정의하여 부모 클래스에 위임한다.
				- 각각의 Extract Class 들의 유사한 책임들은 부모에게 위임하고 다른 책임들은 각자에게 정의 할 수 있다.
		- 산탄총 수술(Shotgun Surgery)
			- Move Field와 Move Method를 통해 책임을 기존의 어떤 클래스로 모으거나, 새로운 클래스를 만드는 것이다.
			- 산발적으로 여러 곳에 분포된 책임들을 한 곳에 모으면서 설계를 깨끗하게 하여 응집성을 높이는 작업을 수행한다.
1. 예시
	- SRP 적용 전
		- ![OOP SOLID SRP 예시1]({{site.baseurl}}/img/oop-solid-srp-ex-1-classdiagram.png)
	
		```java
		class Guitar {
			private String serialNumber;
			private Double price;
			private Maker maker;
			private Type type;
			private String model;
			private Wood topWood;
			private Wood backWood;
			private int stringNum;
			
			// construct, getter, setter, etc ...
		}
		```  
	
		- serailNumber는 변화 요소가 아닌 고유정보이다. 
		- serialNumber를 제외한 price, maker, type 등은 모두 특성 정보군으로 변경이 발생 할 수 있는 변화 요소이다.
		- 특성 정보군에 변화가 발생하면 항상 해당 클래스를 수정해야 하므로 SRP 적용 대상이 된다.
	- SRP 적용 후
		- ![OOP SOLID SRP 예시2]({{site.baseurl}}/img/oop-solid-srp-ex-2-classdiagram.png)
	
		```java
		class Guitar {
			private String serialNumber;
			private GuitarSpec guitarSpec;
			
			// construct, getter, setter, etc ...
		}
		
		class GuitarSpec {
	        private Double price;
	        private Maker maker;
	        private Type type;
	        private String model;
	        private Wood topWood;
	        private Wood backWood;
	        private int stringNum;
	        
	        // construct, getter, setter, etc ...
	    }
		```  
		
		- 위와 같은 분리를 통해 특성 정보 변경이 일어나 더라도 GuitarSpec 클래스만 변경하면 된다.
		- 변화에 의해 변경되는 부분을 한곳에서 관리 할 수 있게 되었다.
1. 적용 이슈
- 클래스는 자신의 이름을 나타내는 책임이 있어야 한다.
- 올바른 클래스 이름은 해당 클래스의 책임을 나타낼 수 있는 가장 좋은 방법이다.
- 각 클래스는 하나의 개념을 나타내어야 한다.
- 무조건 책임을 분리한닫고 SRP가 적용되는 것은 아니다.
- 개체 간의 응집력이 있다면 병합이 순 작용의 수단이고, 결합력이 있다면 분리가 순 작용의 수단이 된다. (?)


---
## Reference
[객체지향 개발 5대 원리: SOLID](http://www.nextree.co.kr/p6960/)  