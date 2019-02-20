--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 팩토리 메서드 패턴(Factory Method Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
subtitle: '팩토리 메서드 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Factory Method
---  

'팩토리 메서드 패턴이 무엇이고, 어떠한 특징을 가지는지'

# 팩토리 메서드 패턴이란
- 객체, 생성 처리를 서브 클래스로 분리 해 처리하도록 캡슐화하는 패턴
	- 객체의 생성 코드를 별도의 클래스/메서드로 분리함으로써 객체 생성의 변화에 대비하는 데 유용하다.
	- 특정 기능의 구현은 개별 클래스를 통해 제공되는 것이 바람직한 설계다.
		- 기능의 변경이나 상황에 따른 기능의 선택은 해당 객체를 생성하는 코드의 변경을 초래한다.
		- 상황에 따라 적절한 객체를 생성하는 코드는 자주 중복될 수 있다.
		- 객체 생성 방식의 변화는 해당되는 모든 코드 부분을 변경해야 하는 문제가 발생한다.
	- [스크래치지 패턴]({{site.baseurl}}{% link _posts/2019-02-12-designpattern-strategy.md %}), [싱글턴 패턴]({{site.baseurl}}{% link _posts/2019-02-11-designpattern-singleton.md %}), [템플릿 메서드 패턴]({{site.baseurl}}{% link _posts/2019-02-11-designpattern-templatemethod.md %}) 를 사용한다.
	- [생성 패턴 중 하나]({{site.baseurl}}{% link _posts/2019-02-08-designpattern-intro.md %})
- ![팩토리 메서드 패턴 예시1]({{site.baseurl}}/img/designpattern-factorymethod-ex-1-classdiagram.png)
- 역할이 수행하는 작업
	- Product
		- 팩토리 메서드로 생성될 객체의 공통 인터페이스
	- ConcreteProduct
		- 구체적으로 객체가 생성되는 클래스
	- Creator
		- 팩토리 메서드를 갖는 클래스
	- ConcreteCreator
		- 팩토리 메서드를 구현하는 클래스로 CreateProduct 객체를 생성
- 팩토리 메서드 패턴의 개념과 적용 방법
	1. 객체 생성을 전담하는 별도의 Factory 클래스 이용
		- 스트래티지 패턴과 싱글턴 패턴을 이용한다.
	1. 상속 이용을 이용하여 하위 클래스에서 적합한 클래스의 객체 생성
		- 스트래티지 패턴, 싱글턴 패턴, 템플릿 메서드 패턴을 이용한다.
		- 다른 방법으로 팩토리 메서드 패턴 적용하기 참조
		
# 예시
## 여러 가지 방식의 엘레베이터 스케쥴링 방법 지원하기
- ![팩토리 메서드 패턴 엘레베이터1]({{site.baseurl}}/img/designpattern-factorymethod-elevator-1-classdiagram.png)
- 작업 처리량(Throughput)을 기준으로 한 스케쥴링에 따른 엘레베이터 관리
- 스케쥴링은 주어진 요청(목적지 층과 방향)을 받았을 때 여러 대의 엘레베이터 중 하나를 선택하는 것을 말한다.
	- 엘레베이터 내부에서 버튼(Elevator Button)을 눌렀을 때는 해당 사용자가 탄 엘레베이터를 이동시킨다.
	- 사용자가 엘레베이터 외부, 건물 내부의 층에서 버튼(Floor Button)을 누른 경우에는 여러 대의 엘레베이터 중 하나를 선택해 이동시켜야 한다.
	
```java
public class ElevatorManager {
	private List<ElevatorController> controllers;
	private ThroughputSheduler scheduler;
	
	// 주어진 수 만큼의 ElevatorController를 생성함
	public ElevatorManager(int controllerCount) {
		// 엘레베이터 이동을 책임지는 ElevatorController 객체 생성
		this.controllers = new ArrayList<>(controllerCount);
		
		for(int i = 0; i < controllerCount; i++){
			ElevatorController controller = new ElevatorController(i);
			this.controllers.add(controller);
		}
		
		// 엘레베이터 스케쥴링(엘레베이터 선택)하기 위한 ThroughputScheduler 객체 생성
		this.scheduler = new ThroughputSheculer();
	}
	
	public void requestElevator(int destination, Direction direction) {
		// ThroughputScheduler를 이용해 엘레베이터를 선택함
		int selectedElevator = this.scheduler.selectElevator(this, destination, direction);
		// 선택된 엘레베이터 이동
		this.controllers.get(selectedElevator).gotoFloor(destination);
	}
}
```  

```java
public class ElevatorController {
	private int id;
	private int curFloor;
	
	public ElevatorController(int id) {
		this.id = id;
		this.curFloor = 1;
	}
	
	public void gotoFloor(int destination) {
		System.out.println("Elevator " + this.id + " floor : " + this.curFloor);
		
		// 현재 층 경신, 주어진 목적지 층으로 엘레베이터 이동
		this.curFloor = destination;
		System.out.println("=> " + this.curFloor);
	}
}
```  

```java
// 엘레베이터 작업 처리량을 최대화 시키는 전략의 클래스
public class ThroughputScheduler {
	public int selectElevator(ElevatorManager elevatorManager, int destination, Direction direction) {
		int result = 0;
		// 처리 알고리즘
		return result;
	}
}
```  

- ElevatorManager 클래스
	- 이동 요청을 처리하는 클래스
	- 엘레베이터를 스케쥴링(엘레베이터 선택)하기 위한 ThroughputScheduler 객체를 갖는다.
	- 각 엘레베이터의 이동을 책임지는 ElevatorController 객체를 복수개 갖는다.
- requestElevator() 메서드
	1. 요청(목적지 층, 방향)을 받았을 때 우선 ThroughputScheduler 클래스의 selectedElevator() 메서드를 호출해 적정한 엘레베이터를 선택한다.
	1. 선택된 엘레베이터에 해당하는 ElevatorController 객체의 gotoFloor() 메서드를 호출해 엘레베이터를 이동시킨다.
	
## 문제점 1
1. 다른 스케쥴링 전략을 사용하는 경우
	- 엘레베이터 작업 처리량을 최대화(ThroughputScheduler 클래스)시키는 전략이 아닌 사용자의 대기 시간을 최소화하는 엘레베이터 선택 전략을 사용해야 한다면 ?
1. 프로그램 실행 중에 스케쥴링 전략을 변경, 동적 스케쥴링을 지원 해야 하는 경우
	- 오전에는 대기 시간 최소화 전략을 사용하고, 오후에는 처리량 최대화 전략을 사용해야 한다면 ?
	
- 문제점 1 해결 방법 - 스트레티지 패턴 이용
![팩토리 메서드 패턴 엘레베이터 스트레티지 패턴 해결]({{site.baseurl}}/img/designpattern-factorymethod-elevator-solution-strategy-classdiagram.png)

```java
public class ElevatorManager {
	private List<ElevatorController> controllers;
	
	// 주어진 수만큼 ElevatorController 생성
	public ElevatorManager(int controllerCount) {
		// 엘레베이터 이동을 책임지는 ElevatorController 객체 생성
		this.controllers = new ArrayList<>();
		
		for(int i = 0; i < controllerCount; i++) {
			ElevatorController controller = new ElevatorController(i);
			this.controllers.add(controller);
		}
	}
	
	// 요청에 따라 엘레베이터를 선택하고 이동
	public void requestElevator(int destination, Direction direction) {
		// 엘레베이터 스케쥴러 인터페이스
		ElevatorScheduler scheduler;
		
		int hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
		
		// 오전에는 ResponseTimeScheduler, 오후에는 ThroughputScheduler
		if(hour < 12) {
			scheduler = new ResponseTimeScheduler();
		} else {
			scheduler = new ThroughputScheduler();
		}
		
		// ElevatorScheduler 인터페이스를 이용해 엘레베이터를 선택함
		int selectedElevator = scheduler.selectElevator(this, destination, direction);
		// 선택된 엘레베이터 이동
		this.controllers.get(selectedElevator).gotoFloor(destination);
	}
}
```  

```java
// 엘레베이터 작업 처리량을 최대화 시키는 전략의 클래스
public class ThroughputScheduler {
	public int selectElevator(ElevatorManager elevatorManager, int destination, Direction direction) {
		int result = 0;
		// 처리 알고리즘
		return result;
	}
}

// 사용자 대기시간을 최소화시키는 전략의 클래스
public class ResponseTimeScheduler {
	public int selectedFloor(ElevatorManager elevatorManager, int destination, Direction direction) {
		int result = 1;
		// 처리 알고리즘
		return result;
	}
}
```  

- requestElevator() 메서드가 실행될 때마다 현재 시간에 따라 적절한 스케쥴링 객체를 생성해야 한다.
- ElevatorManager 클래스의 입장에서는 여러 스케쥴링 전략이 있기 때문에 ElevatorScheduler라는 인터페이스를 사용하여 여러 전략들을 캡슐화하여 동적으로 선택할 수 있게 한다.
- 하지만, 문제는 여전히 남아 있다.

## 문제점 2
- 엘레베이터 스케쥴링 전략이 추가되거나 동적 스케쥴링 방식으로 전략을 선택하도록 변경되면 ?
	1. 해당 스케쥴링 전략을 지원하는 구체적인 클래스를 생성해야 한다.
	1. ElevatorManager 클래스의 requestElevator() 메서드도 수정 해야 한다.
	1. 이처럼 엘레베이터를 선택하는 전략의 변경에 따라 requestElevator()가 변경되는 것은 바람직하지 않다.
		- 새로운 스케쥴링 전략이 추가 되는 경우
			- 엘레베이터 노후화 최소화 전략
		- 동적 스케쥴링 방식이 변경되는 경우
			- 오전 : 대기시간 최소화 전략, 오후 : 처리량 최대화 전략 -> 두 전략의 시간이 변경되는 경우
			
## 해결 방법 (Factory 클래스 이용)
### 과정1
주어진 기능을 실제로 제공하는 적절한 클래스 생성 작업을 별도의 클래스/메서드로 분리 시켜야 한다.
- ![팩토리 메서드 엘레베이터 패턴 해결2]({{site.baseurl}}/img/designpattern-factorymethod-elevator-solution-2-classdiagram.png)
- 엘레베이터 스케쥴링 전략에 일치하는 클래스를 생성하는 코드를 reqeustElevator()메서드에서 분리해 별도의 클래스/메서드를 정의한다.
	- 변경 전 : ElevatorManager 클래스가 직접 ThroughputScheduler 객체와 ResponsetimeScheduler 객체를 생성
	- 변경 후 : SchedulerFactory 클래스의 getScheduler() 메서드가 스케쥴링 전략에 맞는 객체를 생성
	
```java
public enum SchedulerStrategyId {
	RESPONSE_TIME, THROUGHPUT, DYNAMIC
}
```  

```java
public class SchedulerFactory {
	// 스케쥴링 전략에 맞는 객체를 생성
	public static ElevatorScheduler getScheduler(SchedulerStrategyId strategyId) {
		ElevatorScheduler scheduler = null;
		
		switch(strategyId) {
			// 대기 시간 최소화 전략
			case RESPONSE_TIME:
				scheduler = new ResponseTimeScheduler();		
				break;
			// 처리량 최대화 전략
			case THROUGHPUT:
				scheduler = new ThroughputScheduler();
				break;
			// 동적 스케쥴링
			case DYNAMIC:
				int hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
				
				if(hour <12) {
					scheduler = new ResponseTimeScheduler();
				} else {
					scheduler = new ThroughputScheduler();
				}
				break;
		}
		
		return scheduler;
	}
}
```  

- 이제 ElevatorManager 클래스의 requestElevator() 메서드에서는 SchedulerFactory 클래스의 getScheduler() 메서드를 호출하면 된다.

```java
public class ElevatorManager {
	private List<ElevatorController> controllers;
	private SchedulerStrategyId strategyId;
	
	// 주어진 수만큼 ElevatorController 생성
	public ElevatorManager(int controllerCount) {
		// 엘레베이터 이동을 책임지는 ElevatorController 객체 생성
		this.controllers = new ArrayList<>();
		
		for(int i = 0; i < controllerCount; i++) {
			ElevatorController controller = new ElevatorController(i);
			this.controllers.add(controller);
		}
	}
	
	// 실행 중에 다른 스케쥴링 전략으로 지정 가능
	public void setStrategyId(SchedulerStrategyId strategyId) {
		this.strategyId = strategyId;
	}
	
	// 요청에 따라 엘레베이터를 선택하고 이동
	public void requestElevator(int destination, Direction direction) {
		// 주어진 전략 ID에 해당되는 ElevatorScheduler를 사용
		ElevatorScheduler scheduler = SchedulerFactory.getScheduler(this.strategyId);
		
		// 주어진 전략에 따라 엘레베이터를 선택함
		int selectedElevator = scheduler.selectElevator(this, destination, direction);
		// 선택된 엘레베이터 이동
		this.controllers.get(selectedElevator).gotoFloor(destination);
	}
}
```  

```java
public class Client {
	public static void main(String[] args) {
		ElevatorManager emWithResponseTimeScheduler = new ElevatorManager(2, SchedulerStrategyId.RESPONSE_TIME);
		emWithResponseTimeScheduler.requestElevator(10, Direction.UP);
		
		ElevatorManager emWithThroughputScheduler = new ElevatorManager(2, SchedulerStrategy.THROUGHPUT);
		emWithThroughputScheduler.requestElevator(19, Direction,UP);
		
		ElevatorManager emWithDynamicScheduler = new ElevatorManager(2, SchedulerStrategy.DYNAMIC);
		emWithDynamicScheduler.requestElevator(10, Direction.UP);
	}
}
```  

- Client 클래스에서는 총 3개의 ElevatorManager 객체를 사용하는데, 세 객체 모두 10층으로 이동 요청을 하지만 서로 다른 엘레베이터가 선택될 수 있다.

### 과정2
동적 스케쥴링 방식(DynamicScheduler)이라고 하면 여러 번 스케쥴링 객체를 생성하지 않고 한 번 생성한 것을 계속해서 사용하는 것이 바람직할 수 있다.
- 싱글턴 패턴을 활용한 엘레베이터 스케쥴링 전략 설계
	- ![팩토리 메서드 패턴 엘레베이터 해결3]({{site.baseurl}}/img/designpattern-factorymethod-elevator-solution-3-classdiagram.png)
	- 스케쥴링 기능을 제공하는 ResponseTimeScheduler 클래스와 ThroughputShceudler 클래스는 오직 하나의 객체만 생성해서 사용하도록 한다.
	- 생성자를 통해 직접 객체를 생성하는 것이 허용되지 않아야 한다.
		- 이를 위해 각 생성자를 private으로 정의한다.
		- 대신 getInstance() 라는 정적 메서드로 객체 생성을 구현한다.
		
```java
public class SchedulerFactory {
	// 스케쥴링 전략에 맞는 객체를 생성
	public static ElevatorScheduler getScheduler(SchedulerStrategyId strategyId) {
		ElevatorScheduler scheduler = null;
		
		switch(strategyId) {
			// 대기 시간 최소화 전략
			case RESPONSE_TIME:
				scheduler = ResponseTimeScheduler.getInstance();
				break;
			case THROUGHPUT:
				scheduler = ThroughputScheduler.getInstance();
				break;
			case DYNAMIC:
				int hour = Calendar.getInstance().get(calendar.HOUR_OF_DAY);
				
				if(housr < 12) {
					scheduler = ResponseTimerScheduler.getInstance();
				} else {
					scheduler = ThroughputScheduler.getInstance();
				}
		}
		
		return scheduler;
	}
}
```  

```java
// 싱글턴 패턴으로 구현한 ThroughputScheduler 클래스
public class ThroughputScheduler extends ElevatorScheduler {
	private static ElevatorScheduler scheduler;
	
	private ThroughputScheduler() {}
	
	public static ElevatorScheduler getInstance() {
		if(this.scheduler == null){
			this.scheduler = new ThroughputScheduler();
		}
		
		return this.scheduler;
	}
	
	public int selectElevator(ElevatorManager elevatorManager, int destination, Direction direction) {
		int result = 0;
		// 처리 알고리즘
		return result;
	}
}

// 싱글턴 패턴으로 구현한 ResponsetimeScheduler 클래스
public class ResponseTimeScheduler extends ElevatorScheduler {
	private static ElevatorScheduler scheduler;
	
	private ThroughputScheduler() {}
	
	private static ElevatorScheduler getInstance() {
		if(this.schduler == null) {
			this.scheduler = new ResponseTimeScheduler();
		}
		
		return this.scheduler;
	}
	public int selectedFloor(ElevatorManager elevatorManager, int destination, Direction direction) {
		int result = 1;
		// 처리 알고리즘
		return result;
	}
}
```  

- 이제 단 1개의 ThroughputScheduler와 ResponseTimeScheduler 객체를 사용할 수 있다.
- 객체 생성을 전담하는 별도의 Factory 클래스를 분리하여 객체 생성의 변화에 대비할 수 있다.
- 이 방법은 스트래티지 패턴과 싱글턴 패턴을 이용하여 팩토리 메서드 패턴을 적용하였다.

## 해결 방법 (상속을 이용)
하위 클래스에서 적합한 클래스의 객체를 생성하여 객체의 생성 코드를 분리한다.
- 이 방법은 스트래티지 패턴, 싱글턴 패턴, 템플릿 메서드 패턴을 이용하여 팩토리 메서드 패턴을 적용한다.
- ![팩토리 메서드 패턴 엘레베이터 해결4]({{site.baseurl}}/img/designpattern-factorymethod-elevator-solution-4-classdiagram.png)

```java
// 템플릿 메서드를 정의하는 클래스 : 하위 클래스에서 구현될 기능을 primitive 메서드로 정의
public abstract class ElevatorManager {
	private List<ElevatorController> controllers;
	
	// 주어진 수만큼 ElevatorController 생성
	public ElevatorManager(int controllerCount) {
		// 엘레베이터 이동을 책임지는 ElevatorController 객체 생성
		this.controllers = new ArrayList<>();
		
		for(int i = 0; i < controllerCount; i++) {
			ElevatorController controller = new ElevatorController(i);
			this.controllers.add(controller);
		}
	}
	
	// 팩토리 메서드 : 스케쥴링 전략 객체를 생성하는 기능 제공
	protected abstract ElevatorScheduler getScheduler();
	
	// 템플릿 메서드 : 요청에 따라 엘레베이터를 선택하고 이동
	public void requestElevator(int destination, Direction direction) {
		// 하위 클래스에서 오버라이드 된 getScheduler() 메서드를 호출함
		// primitive, hook 메서드
		ElevatorScheduler scheduler = this.getScheduler();
		
		// 주어진 전략에 따라 엘레베이터를 선택함
		int selectedElevator = scheduler.selectElevator(this, destination, direction);
		// 선택된 엘레베이터 이동
		this.controllers.get(selectedElevator).gotoFloor(destination);
	}
}
```  

```java
// 처리량 최대화 전략 하위 클래스
public class ElevatorManagerWithThroughputScheduling extends ElevatorManager {
	public ElevatorManagerWithThroughputScheduling(int controllerCount) {
		// 상위 클래스 생성자 호출
		super(controllerCount);
	}
	
	// primitive, hook 메서드
	@Override
	protected ElevatorScheduler getScheduler() {
		ElevatorScheduler scheduler = ThroughputScheduler.getInstance();
		return scheduler;
	}
}

// 대기 시간 최소화 전략 하위 클래스
public class ElevatorManagerWithResponseTimeScheduling extends ElevatorManager {
	public ElevatorManagerWithResponseTimeScheduling(int controllerCount) {
		// 상위 클래스 생성자 호출
		super(controllerCount);
	}
	
	// primitive, hook 메서드
	@Override
	protected ElevatorScheduler getScheduler() {
		ElevatorScheduler scheduler = ResponseTimeScheduler.getInstance();
		return scheduler;
	}
}

// 동적 스케쥴링 전략 하위 클래스
public class ElevatorManagerWithDynamicScheduling extends ElevatorManager {
	public ElevatorManagerWithDynamicScheduling(int controllerCount) {
		// 상위 클래스 생성자 호출
		super(controllerCount);
	}
	
	// primitive, hook 메서드
	@Override
	protected ElevatorScheduler getShceduler() {
		ElevatorScheduler scheduler = null;
		
		int hour = Calendar.getInstance().gete(Calendar.HOUR_OF_DAY);
		
		if(hour < 12){
			scheduler = ResponseTimeScheduler.getInstance();
		} else {
			scheduler = ThroughputScheduler.getInstance();
		}
		
		return scheduler;
	}
}
```  

- 팩토리 메서드
	- ElevatorManager 클래스의 getScheduler() 메서드
	- 스케쥴링 전략 객체를 생성하는 기능 제공(객체 생성을 분리)
	- 템플릿 메서드 패턴의 개념에 따르면, 하위 클래스에서 오버라이드될 필요가 있는 메서드는 primitive, hook 메서드라고 부름
- 템플릿 메서드
	- ElevatorManager 클래스의 requestElevator() 메서드
	- 공통 기능(스케쥴링 전략 객체 생성, 엘레베이터 선택, 엘레베이터 이동)의 일반 로직 제공
	- 하위 클래스에서 구체적으로 정의할 필요가 있는 '스케쥴링 전략 객체 생성' 부분은 하위 클래스에서 오버라이드
	- 템플릿 메서드 패턴을 이용하면 전체적으로는 동일하면서 부분적으로는 다른 구문으로 구성된 메서드의 코드 중복을 최소화시킬 수 있다.
- 팩토리메서드를 호출하는 상위클래스의 메서드는 템플릿 메서드가 된다.  

![팩토리 메서드 패턴 엘레베이터 정리1]({{site.baseurl}}/img/designpattern-factorymethod-elevator-conclusion-1-classdiagram.png)
- Product : ElevatorScheduler 인터페이스
- ConcreteProduct : ThroughputScheduler 클래스, ResponseTimeScheduler 클래스
- Creator : ElevatorManager 클래스
- ConcreteCreator : ElevatorManagerWithThroughScheduling 클래스, ElevatorManagerWithResponseTimeScheduling 클래스, ElevatorManagerWithDynamicScheduling 클래스
	
---
 
## Reference
[[Design Pattern] 팩토리 메서드 패턴이란](https://gmlwjd9405.github.io/2018/08/07/factory-method-pattern.html)
