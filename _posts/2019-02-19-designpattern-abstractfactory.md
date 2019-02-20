--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 추상 팩토리 패턴(Abstract Factory Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
subtitle: '추상 팩토리 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Abstract Factory
---  

'추상 팩토리 패턴이 무엇이고, 어떠한 특징을 가지는지'

# 추상 팩토리 패턴이란
- 구체적인 클래스에 의존하지 않고 서로 연관되거나 의존적인 객체들의 조합을 만드는 인터페이스를 제공하는 패턴
	- 관련성 있는 여러 종류의 객체를 일관된 방식으로 생성하는 경우에 유용하다.
	- [싱글턴 패턴]({{site.baseurl}}{% link _posts/2019-02-11-designpattern-singleton.md %}), [팩토리 메서드 패턴]({{site.baseurl}}{% link _posts/2019-02-17-designpattern-factorymethod.md %})을 사용한다.
	- [생성 패턴 중 하나]({{site.baseurl}}{% link _posts/2019-02-08-designpattern-intro.md %})
- ![추상 팩토리 패턴 예시1]({{site.baseurl}}/img/designpattern-abstractfactory-ex-1-classdiagram.png)
- 역할이 수행하는 작업
	- AbstractFactory
		- 실제 팩토리 클래스의 공통 인터페이스
	- ConcreteFactory
		- 구체적인 팩토리 클래스로 AbstractFactory 클래스의 추상 메서드를 오버라이드함으로써 구체적인 제품을 생성한다.
	- AbstractProduct
		- 제품의 공통 인터페이스
	- ConcreteProduct
		- 구체적인 팩토리 클래스에서 생성되는 구체적인 제품
		
# 예시
## 엘레베이터 부품 업체 변경하기
- ![추상 팩토리 패턴 엘레베이터1]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-1-classdiagram.png)
- 여러 제조 업체의 부품을 사용하더라도 같은 동작을 지원하게 하는 것이 바람직하다.
- 엘레베이터 프로그램의 변경을 최소화해야 한다.
	- 추상 클래스 Motor, Door 정의
		- 제조 업체가 다른 경우 구체적인 제어 방식은 다르지만 엘레베이터 입장에서는 모터를 구동해 엘레베이터를 이동시킨다는 면에서는 동일하다.
		- 추상 클래스로 Motor를 정의하고 여러 제조 업체의 모터를 하위 클래스로 정의할 수 있다.
	- Motor 클래스 -연관관계-> Door
		- Motor 클래스는 이동하기 전에 문을 닫아야 한다.
		- Door 객체의 getDoorStatus() 메서드를 호출하기 위해서 연관관계가 필요하다.
	- Motor 클래스의 핵심 기능인 이동은 move() 메서드로 정의
	
	```java
	public void move(Direction direction) {
		// 1. 이미 이동 중이면 무시한다.
		// 2. 만약 문이 열려 있으면 문을 닫는다.
		// 3. 모터를 구동해서 이동 시킨다 -> 제조 업체에 따라 다르다.
		// 4. 모터의 상태를 이동 중으로 설정한다.
	}
	```  
	
	- 4단계의 동작 모두 LGMotor, HyundaiMotor 클래스의 공통 기능이고, 3. 단계 부문만 달라진다.
	- 템플릿 메서드 패턴 이용
		- 일반적인 흐름에서는 동일하지만 특정 부분만 다른 동작을 하는 경우에는 일반적인 기능을 상위클래스에 템플릿 메서드로서 설계할 수 있다.
		- 특정 부분에 대해서는 하위 클래서에서 오버라이드 할 수 있도록 한다.
	- Door 클래스의 open(), close() 메서드 정의
	
	```java
	public void open() {
		// 1. 이미 문이 열려 있으면 무시한다.
		// 2. 문을 닫는다. -> 제조 업체에 따라 다르다.
		// 문의 상태를 닫힘으로 설정한다.
	}
	```  
	
	- 나머지 동작은 공통으로 필요한 기능이고, 2. 단계 부분만 달라진다.
	- move() 메서드와 같이 템플릿 메서드 패턴을 적용할 수 있다.
- 템플릿 메서드 패턴을 적용한 예제 코드
	- 전체적으로는 동일하면서 부분적으로는 다른 구문으로 구성된 메서드의 코드 중복을 최소화 할 수 있다.
	
```java
public abstract class Door {
	private DoorStatus doorStatus;
	
	public Door() {
		this.doorStatus = doorStatus;
	}
	
	public DoorStatus getDoorStatus() {
		return this.doorStatus;
	}
	
	// primitive, hook 메서드
	protected abstract void doClose();
	protected abstract void doOpen();
	
	// 템플릿 메서드 : 문을 닫는 기능 수행
	public void close() {
		// 이미 문이 닫혀 있으면 아무 동작 하지 않음
		if(this.doorStatus == DoorStatus.CLOSED) {
			return;
		}
		
		// primitive, hook 메서드 하위 클래스에서 오버라이드
		// 실제로 문을 닫는 동작 수행
		this.doClose();
		// 문의 상태를 닫힘으로 변경
		this.doorStatus = DoorStatus.CLOSED;
	}
	
	// 템플릿 메서드 : 문을 여는 기능 수행
	public void open() {
		// 이미 문이 열려 있으면 아무 동작 하지 않음
		if(this.doorStatus == DoorStatus.OPENED) {
			return;
		}
		
		// primitive, hook 메서드 하위 클래스에서 오버라이드
		// 실제로 문을 여는 동작 수행
		this.doOpen();
		// 문의 상태 열림으로 수정
		this.doorStatus = DoorStatus.OPENED;
	}
}

```  

```java
public class LGDoor extends Door {
	@Override
	protected void doClose() {
		System.out.println("close LG Door");
	}
	
	@Override
	protected void doOpen() {
		System.out.println("open LG Door");
	}
}

public class HyundaiDoor extends Door {
	@Override
	protected void doClose() {
		System.out.println("close Hyundai Door");
	}
	
	@Override
	protected void doOpen() {
		System.out.println("open Hyundai Door");
	}
}
```  

- 엘레베이터 입장에서는 특정 제조 업체의 모터와 문을 제어하는 클래스가 필요하다.
	- LGMotor, LGDoor 객체가 필요하다.
	- 팩토리 메서드 패턴 이용
	
- 팩토리 메서드 패턴을 적용한 예제 코드
	- 객체 생성 처리를 서브 클래스로 분리하여 캡슐화함으로써 객체 생성의 변화에 대비할 수 있다.
	- ![추상 팩토리 패턴 엘레베이터2]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-2-classdiagram.png)

```java
public enum VendorId {
	LG, HYUNDAI
}
```  

```java
public class DoorFactory {
	// VendorId에 따라 LGDoor, HyundaiDoor 객체를 생성함
	public static Door createDoor(VendorId vendorId) {
		Door door = null;
		
		switch(vendorId) {
			case LG:
				door = new LGDoor();
				break;
			case HYUNDAI:
				door = new HyundaiDoor();
				break;
		}
		
		return door;
	}
}
```  

- DoorFactory 클래스의 createDoor() 메서드는 인자로 주어진 VendorId에 따라 Door 객체를 생성한다.

```java
public class MotorFactory {
	public static Motor createMotor(VnedorId vendorId) {
		Motor motor = null;
		
		switch(vendorId){
			case LG:
				motor = new LGMotor();
				break;
			case HYUNDAI:
				motor = new HyundaiMotor();
				break;
		}
		
		return motor;
	}
}
```  

- MotorFactory 클래스의 createMotor() 메서드는 인자로 주어진 VendorId에 따라 Motor 객체를 생성한다.

```java
public class Client {
	public static void main(String[] args) {
		Door lgDoor = DoorFactory.createDoor(VendorId.LG);
		Motor lgMotor = MotorFactory.createMotor(VendorId.LG);
		
		lgMotor.setDoor(lgDoor);
		
		lgDoor.open();
		lgMotor.move(Direction.UP);
	}
}
```  

- 제조 업체에 따라 모터와 문에 해당하는 구체적인 클래스를 생성해야 한다.
- LGDoor와 LGMotor 객체를 활용하므로 move() 메서드를 호출하기 전 open() 메서드에 의해 문이 열려 있는 상태이다.
- 그러므로 move() 메서드는 문을 먼저 닫고 이동을 시작한다.
- 이때 LGDoor와 LGMotor 객체가 모두 사용된다.

## 문제점
1. 다른 제조 업체의 부품을 사용해야 하는 경우
	- LG 부품 대신 Hyundai 부품을 사용해야 한다면 ?
	
	```java
	public class Client {
		public static void main(String[] args) {
			Door hyundaiDoor = DoorFactory.createDoor(VendorId.HYUNDAI);
			Motor hyundaiMotor = MotorFactory.createMotor(VendorId.MOTOR);
			
			hyundaiMotor.setDoor(hyundaiDoor);
			
			hyundaiDoor.open();
			hyundaiMotor.move(Direction.UP);				
		}
	}
	```  
	
	- 총 10개의 부품을 사용해야 한다면 각 Factory 클래스를 구현하고 이들의 Factory 객체를 각각 생성해야 한다.
	- 부품의 수가 많아지면 특정 업체별 부품을 생성하는 코드의 길이가 길어지고 복잡해 진다.
2. 새로운 제조업체의 부품을 지원해야 하는 경우
	- Samsung 부품을 지원해야 한다면 ?
	
	```java
	public class DoorFactory {
		// VendorId에 따라 LGDoor, HyundaiDoor 객체를 생성함
		public static Door createDoor(VendorId vendorId) {
			Door door = null;
			
			switch(vendorId) {
				case LG:
					door = new LGDoor();
					break;
				case HYUNDAI:
					door = new HyundaiDoor();
					break;
				// 삼성 부품 추가
				case SAMSUNG:	
					door = new SamsungDoor();
					break;
			}
			
			return door;
		}
	}
	```  
	
	- DoorFactory 클래스 뿐만 아니라 나머지 9개의 부품과 연관된 Factory 클래스에도 마찬가지로 삼성의 부품을 생성하도록 변경해야 한다.
	- 위 코드와 마찬가지로 특정 업체별 부품을 생성하는 코드에서 삼성 부품을 생성하도록 모두 변경해야 한다.
- 결과적으로 기존의 팩토리 메서드 패턴을 이용한 객체 생성은 관련 있는 여러 개의 객체를 일관성 있는 방식으로 생성하는 경우에 많은 코드 변경이 발생 하게 된다는 것이다.

## 해결 방법
여러 종류의 객체를 생성할 때 객체들 사이의 관련성이 있는 경우 라면 각 종류별로 별도의 Factory 클래스를 사용하는 대신 관련 객체들을 일관성 있게 생성하는 Factory 클래스를 사용하는 것이 편리할 수 있다.
- ![추상 팩토리 패턴 엘레베이터 해결1]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-solution-1-classdiagram.png)
- MotorFactory, DoorFactory 클래스와 같이 부품별로 Factory 클래스를 만드는 대신 LGElevatorFactory나 HyundaiElevatorFactory클래스와 같이 제조 업체별로 Factory 클래스를 만들 수도 있다.
- LGElevatorFactory : LGMotor, LGDoor 객체를 생성하는 팩토리 클래스
- HyundaiElevatorFactory : HyundaiMotor, HyundaiDoor 객체를 생성하는 팩토리 클래스

```java
// 추상 부품을 생성하는 추상 팩토리 클래스
public abstract class ElevatorFactory {
	public abstract Motor createMotor();
	public abstract Door createDoor();
}
```  

```java
// LG부품을 생성하는 팩토리 클래스
public class LGElevatorFactory extends ElevatorFactory {
	@Override
	public Motor createMotor() {
		return new LGMotor();
	}
	
	@Override
	public Door createDoor() {
		return new LGDoor();
	}
}

// Hyundai 부품을 생성하는 팩토리 클래스
public class HyundaiElevatorFactory extends ElevatorFactory {
	@Override
	public Motor createMotor() {
		return new HyundaiMotor();
	}
	
	@Override
	public Door createDoor() {
		return new HyundaiDoor();
	}
}
```  

```java
// 주어진 업체의 이름에 다라 부품을 생성하는 Client 클래스
public class Client {
	public static void main(String[] args) {
		ElevatorFactory factory = null;
		VendorId vendorId = VendorId.LG;
		
		if(vendorId == VendorId.LG) {
			factory = new LGElevatorFactory();
		} else {
			factory = new HyundaiElevatorFactory();
		}
		
		Door door = factory.createDoor();
		Motor motor = factory.createMotor();
		motor.setDoor(door);
		
		door.open();
		motor.move(Direction.UP);
	}
}
```  

- VendorId에 맞춰 적절한 부품 객체를 생성한다.
	- 문제점 1과 같이 다른 제조 업체의 부품으로 변경하는 경우에도 Client코드를 변경할 필요가 없다.
- 제조 업체별로 Factory 클래스를 정의했으므로 제조 업체별 부품 객체를 간단히 생성할 수 있다.
	- 문제점 2와 같이 새로운 제조 업체의 부품을 지원하는 경우에도 해당 제조 업체의 부품을 생성하는 Factory 클래스만 새롭게 만들면 된다.
- ![추상 팩토리 패턴 엘레베이터 해결2]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-solution-2-classdiagram.png)

```java
// Samsung 부품을 생성하는 팩토리 클래스
public class SaumsungElevatorFactory extends ElevatorFactory {
	@Override
	public Motor createMotor() {
		return new SaumsungMotor();
	}
	
	@Override
	public Door createDoor() {
		return new SamsungDoor();
	}
}

// Samsung Door 클래스
public class SamsungDoor extends Door {
	@Override
	protected void doClose() {
		System.out.println("close Samsung Door");
	}
	
	@Override
	protected void doOpen() {
		System.out.println("open Samsung Door");
	}
}

// Samsung Motor 클래스
public class SamsungMotor extends Motor {
	@Override
	protected void moveMotor() {
		// ....
	}
}
```  

```java
// 주어진 업체의 이름에 다라 부품을 생성하는 Client 클래스
public class Client {
	public static void main(String[] args) {
		ElevatorFactory factory = null;
		VendorId vendorId = VendorId.LG;
		
		if(vendorId == VendorId.LG) {
			factory = new LGElevatorFactory();
		} else if(vendorId == Vendor.HYUNDAI) {
			factory = new HyundaiElevatorFactory();
		} else {
			// Samsung 추가
			factory = new SamsungElevatorFactory();
		}
		
		Door door = factory.createDoor();
		Motor motor = factory.createMotor();
		motor.setDoor(door);
		
		door.open();
		motor.move(Direction.UP);
	}
}
```  

### 추가 보완 해결책(정확한 추상 팩토리 패턴 적용)
![추상 팩토리 패턴 엘레베이터 해결3]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-solution-3-classdiagram.png)
1. 과정 1
팩토리 메서드 패턴 : 제조 업체별 Factory 객체를 생성하는 방식을 캡슐화 한다.
- ElevatorFactoryFactory 클래스 : vendorId에 따라 해당 제조 업체의 Factory 객체를 생성
- ElevatorFactoryFactory 클래스의 getFactory() 메서드 : 팩토리 메서드

```java
// 팩토리 클래스에 팩토리 메서드 패턴을 적용
public class ElevatorFactoryFactory {
	public static ElevatorFactory getFactory(VendorId vendorId) {
		ElevatorFactory factory = null;
		
		switch(vendorId) {
			case LG:
				factory = LGElevatorFactory.getInstance();
				break;
			case HYUNDAI:
				factory = HyundaiElevatorFactory.getInstance();
				break;
			case SAMSUNG:
				factory = SamsungElevatorFactory.getInstance();
				break;
		}
		
		return factory;
	}
}
```  

1. 과정 2
싱글턴 패턴 : 제조 업체별 Factory 객체는 각각 1개만 있으면 된다.
- 3개의 제조 업체별 Factory 클래스를 싱글턴으로 설계

```java
// 싱글턴 패턴을 적용한 LG 팩토리
public class LGElevatorFactory extends ElevatorFactory {
	private static ElevatorFactory factory = null;
	
	private LGElevatorFactory() { }
	
	public static void getInstance() {
		if(factory == null) {
			factory = new LGElevatorFactory();
		}
		
		return factory;
	}
	
	@Override
	public Motor createMotor() {
		return new LGMotor();
	}
	
	@Override
	public Door createDoor() {
		return new LGDoor();
	}
}

// HyndaiElevatorFactory, SamsungElevatorFactory 모두 동일한 방식으로 싱글턴 적용
```  

- 추상 팩토리 패턴을 이용한 방법을 사용하는 Client

```java
// 주어진 업체의 이름에 다라 부품을 생성하는 Client 클래스
public class Client {
	public static void main(String[] args) {
		ElevatorFactory factory = null;
		VendorId vendorId = VendorId.LG;
		
		factory = ElevatorFactoryFactory.getFacotry(vendorId);
		
		Door door = factory.createDoor();
		Motor motor = factory.createMotor();
		motor.setDoor(door);
		
		door.open();
		motor.move(Direction.UP);
	}
}
```  

# Summary
![추상 팩토리 패턴 엘레베이터 요약1]({{site.baseurl}}/img/designpattern-abstractfactory-elevator-summary-1-classdiagram.png)
- AbstractFactory : ElevatorFactory 클래스
- ConcreteFactory : LGElevatorFactory, HyundaiElevatorFactory 클래스
- AbstractProductA : Motor 클래스
- ConcreteProductA : LGMotor, HyundaiMotor 클래스
- AbstractProductB : Door 클래스
- ConcreteProductB : LGDoor, HyundaiDoor 클래스


---
 
## Reference
[[Design Pattern] 추상 팩토리 패턴이란](https://gmlwjd9405.github.io/2018/08/08/abstract-factory-pattern.html)
