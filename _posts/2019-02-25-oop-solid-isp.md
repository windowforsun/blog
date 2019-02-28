--- 
layout: single
classes: wide
title: "객체 지향 프로그래밍(OOP) 객체 지향 디자인 5원칙(SOLID 원칙) - ISP"
header:
  overlay_image: /img/oop-bg.jpg
subtitle: 'SOLID-ISP 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - OOP
tags:
    - OOP
    - ISP
    - 객체 지향
    - SOLID
    - 디자인 패턴
    - Design Pattern
---  

'SOLID-ISP 란 무엇이고, 어떠한 특징을 가지고 있는지'


## I - ISP(Interface Segregation Principle) 인터페이스 분리 원칙
1. 정의
	- ISP 원칙은 한 클래스는 자신이 사용하지 않는 인터페이스는 구현하지 말아야 한다는 원칙이다.
	- 어떤 클래스가 다른 클래스에 종속될 때에는 가능한 최소한의 인터페이스만 사용해야 한다.
	- 어떤 클래스를 이용하는 클라이언트가 여러 개고 이들이 해당 클래스의 특정 부분집합만을 이용한다면, 이들을 따로 인터페이스를 빼내어 클라이언트가 기대하는 메시지만을 전달 할 수 있도록 한다.
	- SRP가 클래스의 단일 책임을 강조한다면, ISP는 인터페이스의 단일책임을 강조한다.
	- SRP가 클래스 분리를 통해 변화의 적응성을 획득하는 반면, ISP는 인터페이스 분리를 통해 같은 목표를 도달한다.
1. 적용 방법
	- 클래스 인터페이스를 통한 분리
		- 클래스의 상속을 이용하면 인터페이스를 나눌 수 있다.
		- 이는 클라이언트에게 변화를 주지 않을 뿐 아니라, 인터페이스를 분리하는 효과를 얻는다.
		- 객체지향 언어에서 상속을 이용한 확장은 상속 받는 클래스의 성격을 디자인 시점에 규정한다.
		- 인터페이스를 상속받는 순간 인터페이스에 예속되어 제공하는 서비스의 성격이 제한된다.
	- 객체 인터페이스를 통한 분리
		- 위임(Delegation)을 이용하여 인터페이스를 나눌 수 있다.
		- 위임이란, 특정 일의 책임을 다른 클래스나 메소드에 맡기는 것이다.
		- 다른 클래스의 기능을 사용해야 하지만, 그 기능을 변경하고 싶지 않다면 상속 대신 위임을 사용한다.
1. 예시
	- Java Swing의 JTable
		- JTable 클래스에는 많은 메소드들이 있다.
		- 컬럼 추가, 셀 에디터 리스너 부착 등 여러 역할이 하나의 클래스 안에 있다.
		- JTable의 입장에서 본다면 모두 제공해야 하는 역할이다.
		- JTable은 ISP가 제안하는 방식으로 모든 인터페이스 분리를 통해 특정 역할만 이용하는 것을 제공한다.
		- Accessible, CellEditorListener, ListSelectionListener, Scrollable, TableColumnModelListener, TableModelListener 등 여러 인터페이스 구현을 통해 서비스를 제공한다.
		- JTable은 자신을 이용하여 테이블을 만드는 객체(모든 서비스를 필요로 하는)에게는 기능 전부를 노출하지만, 이벤트 처리와 관련해서는 여러 리스너 인터페이스를 통해 해당 기능만 노출한다.
		
		```java
		public class SimpleTableDemo ... implements TableModelListener {
			...
			public SimpleTableDemo() {
				table.getModel().addTableModelListener(this);
			}
	
			@Override
			public void tableChanged(TableModelEvent e) {
				....
			}
		}
		```  
		
1. 적용 이슈
	- 구현된 클라이언트에 변경을 주지 말아야 한다.
	- 두 개 이상의 인터페이스가 공유하는 부분의 재사용을 극대화 한다.
	- 서로 다른 성격의 인터페이스를 명확히 분리한다.
	

---
## Reference
[객체지향 개발 5대 원리: SOLID](http://www.nextree.co.kr/p6960/)  