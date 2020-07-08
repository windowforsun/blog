--- 
layout: single
classes: wide
title: "객체 지향 프로그래밍(OOP) 객체 지향 디자인 5원칙(SOLID 원칙) - OCP"
header:
  overlay_image: /img/oop-bg.jpg
excerpt: 'SOLID-OCP 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - OOP
tags:
    - OOP
    - OCP
    - 객체 지향
    - SOLID
    - 디자인 패턴
    - Design Pattern
---  

### [SOLID 개요]({{site.baseurl}}{% link _posts/oop/2019-02-24-oop-solid.md %})

## O - OCP(Open Close Principle) 개방 폐쇄 원칙
1. 정의
	- 소프트웨어의 구성요소(컴포넌트, 클래스, 모듈, 함수)는 확장에는 열려있고, 변경에는 닫혀있어야 한다는 원리이다.
	- 변경을 위한 비용은 가능한 줄이고, 확장을 위한 비ㅣ용은 가능한 극대화 해야 한다는 의미이다.
	- 요구사항의 변경이나 추가사항이 발생하더라도, 기존 구성요소는 수정이 일어아지 말아야 하며, 기존 구성요소를 쉽게 확장해서 재사용할 수 있어야 한다는 뜻이다.
	- 로버트 C.마틴은 OCP는 관리가능하고 재사용 가능한 코드를 만드는 기반이며, OCP를 가능케 하는 중요 메커니즘은 추상화와 다형성이라고 설명하고 있다.
	- OCP는 객체 지향의 장점을 극대화하는 아주 중요한 원리이다.
1. 적용 방법
	- 변경(확장)될 것과 변하지 않을 것을 엄격히 구분한다.
	- 이 두 모듈이 만나는 지점에 인터페이스를 정의한다.
	- 구현에 의존하기보다 정의한 인터페이스에 의존하도록 코드를 작성 한다.
1. 예시
	- OCP 적용 전
		- ![OOP SOLID OCP 예시1]({{site.baseurl}}/img/oop-solid-ocp-ex-1-classdiagram.png)
		```java
		public class Guitar {
			private GuitarSpec guitarSpec;
			// Property, Method ..
		}
		public class GuitarSpec {
			// Property, Method ..
		}
		public class Violin {
			private ViolinSpec violinSpec;
			// Property, Method ..
		}
		public class ViolinSpec {
			// Property, Method ..
		}
		```  
		- SRP에서 변경을 최소화 하는 클래스 구조로 분리 하였지만 Guitar 외에 바이올린, 첼로 등 다양한 악기가 추가 되었을 경우 위처럼 설계될 수 있다.
		- Guitar와 추가 될 다른 악기들을 추상화하는 작업이 필요하다.
	- OCP 적용 후
		- ![OOP SOLID OCP 예시2]({{site.baseurl}}/img/oop-solid-ocp-ex-2-classdiagram.png)
		
		```java
		public abstract class Instrument {
			private String serialNumber;
			private InstrumentSpec instrumentSpec;
			
			// Property, Method ..
		}
		public abstract class InstrumentSpec {
			private double price;
			private String model;
			
			// Property, Method ..
		}
		public class Guitar extends Instrument {
			// Property. Method ..
		}
		public class Violin extends Instrument {
			// Property, Method ..
		}		
		public class GuitarSpec extends InstrumentSpec {
			// Property, Method ..
		}
		public class ViolinSpec extends InstrumentSpec {
			// Property, Method ..
		}
		```  
		
		- 악기들의 공통 속성을 모두 담는 Instrument라는 인터페이스(예시에서는 추상 클래스)를 정의 한다.
		- 악기 특성 정보를 담는 InstrumenSpec 이라는 인터페이스(예시에서는 추상 클래스)도 정의한다.
		- 새로운 악기가 추가 되면서 변경이 발생하는 부분을 추상화 하여 분리 함
		- 코드의 수정을 최소화 하여 결합도는 줄이고 응집도는 높이는 효과를 가져옴
1. 적용 이슈
	- 확장되는 것과 변경되지 않는 모듈을 분리하는 과정에서 관계가 더 복잡해 질 수 있다.
	- 인터페이스는 변경 되어서는 안된다.
	- 인터페이스 설계에서 적당한 추상화 레벨을 선택해야 한다.
		- 추상화란 다른 모든 종류의 객체로부터 식별될 수 있는 객체의 본질적인 특성
		- 행위에 대한 본질적인 정의를 통해 인터페이스를 식별해야 한다.
	

---
## Reference
[객체지향 개발 5대 원리: SOLID](http://www.nextree.co.kr/p6960/)  