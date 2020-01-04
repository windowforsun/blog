--- 
layout: single
classes: wide
title: "[DesignPattern 개념] UML"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '디자인 패턴을 표현하고 이해하는 데에 도움이 되는 UML 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - DesignPattern
tags:
  - DesignPattern
  - UML
---  


## UML 이란
- Unified Modeling Language 의 약자이다.
- 시스템을 시각화 하거나 시스템의 사양이나 설계를 문서화하기 위한 표현 방법이다.

## UML 의 종류
- Class Diagram (클래스 다이어그램)
	- 클래스 명세와 클래스 간의 관계를 표현
- Composite Structure Diagram (복합 구조 다이어그램)
	- 전체-부분 구조를 가진 클래스를 실행할 때의 구조를 표현
- Component Diagram (컴포넌트 다이어그램)
	- 파일과 데이터베이스, 프로세스, 스레드 등의 소프트웨어 구조를 표현
- Deployment Diagram (디플로이먼트 다이어그램)
	- 하드웨어와 네트워크 등 시스템의 물리 구조를 표현
- Object Diagram (객체 다이어그램)
	- 인스턴스 간의 연관 관계를 표현
- Package Diagram (패키지 다이어그램)
	- 패키지 간의 연관 관계를 표현
- Activity Diagram (액티비티 다이어그램)
	- 일련의 처리에 있어 제어의 흐름을 표현
- Sequence Diagram (시퀀스 다이어그램)
	- 인스턴스 간의 상호 작용을 시계열로 표현
- Communication Diagram (커뮤니케이션 다이어그램)
	- 인스턴스 간의 상호 작용을 구조 중심으로 표현
- Interaction Overview Diagram (인터액션 오버뷰 다이어그램)
	- 조건에 따라 다르게 동작을 하는 시퀀스 다이어그램을 액티비티 다이어그램 안에 포함하여 표현
- Timing Diagram (타이밍 다이어그램)
	- 인스턴스 간의 상태 전이와 상호 작용을 시간 제약으로 표현
- UseCase Diagram (유스케이스 다이어그램)
	- 시스템이 제공하는 기능과 이용자의 관계를 표현
- State Machine Diagram (스테이트 머신 다이어그램)
	- 인스턴스의 상태 변화를 표현
	
## Class Diagram
- 클래스 다이어그램은 클래스나 인스턴스, 인터페이스 등의 정적인 관계를 표현한다.
- 클래스 다이어그램이라고 해서 클래스만 등장하는 것은 아니다.

### 화살표 방향
- 클래스 다이어그램에서 화살표 방향은 혼돈하기 쉽다.
- 간단하게 화살표가 나아가는 클래스가 화살표가 향하는 클래스(인터페이스)를 `알고있다.` 라고 이해하면 쉽다.
- 클래스(인터페이스)를 알고 있어야 지칭(화살표)할 수 있기 때문이라고 이해한다.

### 상속관계 (extends)

```java
abstract class AbstractParent {
	int generalField;
	static char staticField;
	
	abstract void abstractMethod();
	int method() {
		// stuff
	}
}

class Child extends AbstractParent {
	@Override
	void abstractMethod() {
		// stuff
	}
	
	static void staticMethod() {
		// stuff
	}
}
```  

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_1.png)

- 위 코드와 그림은 `AbstarctParent` 와 `Child` 두 클래스의 상속관계를 보여주고 있다.
- 빈 삼각형에 실선의 화살표는 클래스의 상속관계를 표현한다.
- 상속관계에서 화살표는 하위(자식) 클래스에서 상위(부모) 클래스로 향한다.
- 하나의 클래스 박스는 `클래스 이름`, `필드 이름`, `메소드 이름` 으로 구성되어 있다.
- 이름 뿐만 아니라 타입, 리턴, 접근 제어자 등의 다양한 정보를 나타낼 수도 있다.
- `abstract` 클래스, 메서드는 이름을 이탤릭체로 표기한다.
- `static` 필드와 메서드는 밑줄로 표기한다.

### 구현관계 (implements)

```java
interface Able {
	void is();
}

class DoAble implements Able {
	@Override
	public void is() {
		// stuff
	}
}
```  

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_2.png)

- 위 코드와 그림은 `Able` 인터페이스와 `DoAble` 클래스의 구현관계를 보여주고 있다.
- 빈 삼각형에 점신의 화살표는 인터페이스와 클래스의 구현관계를 표현한다.
- 구현관계에서 화살표는 클래스에서 인터페이스로 향한다.
- `interface` 의 이름은 이텔릭테로 표기하고, 추가적으로 `<<interface>>` 를 통해 명시하기도 한다.
- `interface` 의 메서드 또한 이텔릭체로 표기한다.

### 집약 (Aggregation)

```java
class Lens {
	// stuff
}

class Camera {
	Lens lens;
	// stuff
}

class Phone {
	Camera[] cameras;
	// stuff
}
```  

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_3.png)

- 위 코드와 그림은 `Lens`, `Camera`, `Phone` 클래스들의 집약관계를 보여주고 있다.
- 화살표화 빈 마름모는 클래스간의 집약관계를 표현한다.
- 집약이란 클래스가 다른 클래스를 포함하는 관계를 뜻한다.
- `Phone` 은 여러개의 `Camera` 를 가질 수 있고, `Camera` 는 하나의 `Lens` 를 가지게 된다.
- 집약관계에서 빈 마름모는 포함하는 클래스에 위치해서 포함되는 클래스로 화살표가 향한다.

### 접근제어

```java
class SomeClass {
	private int privateField;
	protected int protectedField;
	public int publicField;
	int packageField;
	
	private void privateMethod() {
		// stuff
	}
	
	protected void protectedMethod() {
		// stuff
	}
	
	public void publicMethod() {
		// stuff
	}
	
	void packageMethod() {
		// stuff
	}
}
```  

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_4.png)


- 위 코드와 그림은 하나의 클래스에서 접근제어자에 대해 보여 주고 있다.
- 클래스 다이어그램에서 접근제어자를 표현하기 위해서는 아래와 같은 기호를 사용한다.
	- `-` : private 접근제어자로 필드, 메서드 앞에 붙인다.
	- `#` : protected 접근제어자로 필드, 메서드 앞에 붙인다.
	- `+` : public 접근제어자로 필드, 메서드 앞에 붙인다.
	- `~` : package 접근제어자로 필드, 메서드 앞에 붙인다.
	
### 클래스 관계

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_5.png)

- 위 그림은 클래스 관계를 보여주고 있다. 관계를 표시할 때는 관계의 역할이 해당하는 클래스 방향으로 채워진 삼각형을 표시한다.
- `People` 클래스는 `Device` 클래스를 사용한다.
- `Factory` 클래스는 `Device` 클래스를 생성한다.
- `Subject` 클래스는 `Observer` 클래스에게 통지한다.


## Sequence Diagram

```java
class People {
	Phone phone;
	
	public void usePhone() {
		this.phone.useCamera();
		this.phone.savePicture();
	}
}

class Phone {
	Camera camera;
	
	public void useCamera() {
		this.camera.doPicture();
	}
	
	public void savePicture() {
		// stuff
	}	
}

class Camera {
	public void doPicture() {
		// stuff
	}
}
```  

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_uml_6.png)

- 위 코드와 그림은 `People`, `Phone`, `Camera` 를 사용해서 사진을 찍고 저장을 하는 과정을 표현하고 있다.
- 각 인스턴스에서 아래 방향으로 향하는 점섬을 라이프 라인(생존선) 이라고 하고 시간의 흐름을 뜻한다.
- 각 인스턴스 점선에 있는 길죽한 네모는 객체가 활동 중인 것을 뜻한다.
- 꽉찬 세모 실선 화살표는 메서드 호출을 뜻한다.
- 점선 화살표는 메서드의 리턴을 뜻한다.
- 시퀀스 다이어그램은 라이프 라인을 따라가며 위에서 부터 아래로 화살표를 따라가며 읽으면서, 각 인스턴스간의 동작 과정을 익힐 수 있다.


---
## Reference
