--- 
<!--layout: post-->
title: "디자인 패턴(Design Pattern) 템플릿 메서드 패턴(Template Method Pattern)"
subtitle: '템플릿 메서드 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Template Method Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 템플릿 메서드 패턴이란
- 어떤 작업을 처리하는 일부분을 서브 클래스로 캡슐화해 전체 일을 수행하는 구조는 바꾸지 않으면서 특정 단계에서 수행하는 내역을 바꾸는 패턴
    - 전체적으로는 동일하면서 부분적으로는 다른 구문으로 구성된 메서드의 코드 중복을 최소화 할 때 유용하다.
    - 다른 관점에서 보면 동일한 기능을 상위 클래스에서 정의하면서 확장/변화가 필요한 부분만 서브 클래스에서 구현할 수 있도록 한다.
    - 예를 들어, 전체적인 알고리즘은 상위 클래스에서 구현하면서 다른 부분은 하위 클래스에서 구현할 수 있도록 함으로써 전체적인 알고리즘 코드를 재사용하는 데 유용하도록 한다.
- 행위(Behavioral) 패턴 중 하나
- ![템플릿 메서드 클래스 다이어그램 예시](/img/designpattern-templatemethod-ex-classdiagram.png)
- 역할이 수행하는 작업
    - AbstractClass
        - 템플릿 메서드를 정의하는 클래스
        - 하위 클래스에 공통 알고리즘을 정의하고 하위 클래스에서 구현될 기능을 primitive 메서드 또는 hook 메서드로 정의하는 클래스
    - ConcreteClass
        - 물려받은 primitive 메서드 또는 hook 메서드를 구현하는 클래스
        - 상위 클래스에 구현된 템플릿 메서드의 일반적인 알고리즘에서 하위 클레스에 적합하게 primitive 메서드나 hook 메서드를 오버라이드하는 클래스
        
### 예시
#### 여러 회사의 모터 지원하기
- ![모터 클래스 다이어그램](/img/designpattern-templatemethod-motor-1-classdiagram.png)
- 엘리베이터 제어 시스템에서 모터를 구동시키는 기능
    - 예를 들어 현대 모터를 이용하는 제어 시스템이라면 HyundaiMotor 클래스에 move 메서드를 정의할 수 있다.
    - move 메서드를 실행할 때 안전을 위해 Door가 닫혀 있는지 확인하기 위해 연관 관계를 정의한다.
    - Enumeration Interface
        - 모터의 상태(정지, 이동)
        - 문의 상태(닫힘, 열림)
        - 이동 상태(위, 아래)
        
```java
public enum DoorStatus {
    CLOSED, OPENED
}
public enum MotorStatus {
    MOVING, STOPPED
}
public enum Direction {
    UP, DOWN
}
```   
```java
public class Door {
    private DoorStatus doorStatus;
    
    public Door() {
        this.doorStatus = DoorStatus.CLOSED;
    }
    
    public DoorStatus getDoorStatus() {
        return this.doorStatus;
    }
    
    public void close() {
        this.doorStatus = DoorStatus.CLOSED;
    }
    
    public void open() {
        this.doorStatus = DoorStatus.OPENED;
    }
}

public class HyundaiMotor {
    private Door door;
    private MotorStatus motorStatus;
    
    public HyundaiMotor(Door door) {
        this.door = door;
        this.motorStatus = MotorStatus.STOPPED;
    }
    
    private void moveHyundaiMotor(Direction direction) {
        // HyundaiMotor를 구동 시킴 
    }
    
    public MotorStatus getMotorStatus() {
        return this.motorStatus;
    }
    
    private void setMotorStatus(MotorStatus motorStatus) {
        this.motorStatus = motorStatus;
    }
    
    // 엘레베이터 제어
    public void move(Direction direction) {
        MotorStatus motorStatus = this.getMotorStatus();
        
        // 이미 이동 중이면 아무 작업 하지 않음
        if(this.motorStatus == MotorStatus.MOVING) {
            return;
        }
        
        DoorStatus doorStatus = this.door.getDoorStatus();
        
        // 만약 문이 열려 있으면 우선 문을 닫음
        if(doorStatus == DoorStatus.OPENED) {
            this.door.close();
        }
        
        // Hyundai 모터를 주어진 방향으로 이동 시킴
        this.moveHyundaiMotor(direction);
        // 모터 상태를 이동 중으로 변경함
        this.setMotorStatus(MotorStatus.MOVING);
        
    }
}
```  
```java
public class Client {
    public static void main(String[] args){
        Door door = new Door();
        HyundaiMotor hyundaiMotor = new HyundaiMotor(door);
        
        // 위로 올라가도록 엘레베이터 제어
        hyundaiMotor.move(Direction.UP);
    }
}
```  

- HyundaiMotor 클래스의 move 메서드는 우선 getMotorStatus 메서드를 호출해 모터 상태를 조회한다.
    - 모터가 이미 동작 중이면 move 메서드의 실행을 종료한다.
- Door 클래스의 getDoorStatus 메서드를 호출해 문의 상태를 조회한다.
    1. 문이 열려 있는 상태면 Door 클래스의 close 메서드를 호출해 문을 닫는다.
    1. 그리고 moveHyundaiMotor 메서드를 호출해 모터를 구동시킨다.
    1. setMotorStatus를 호출해 모터의 상태를 MOVING으로 기록한다.

#### 문제점
- 다른 회사의 모터를 제어해야 하는 경우
    - HyundaiMotor 클래스는 현대 모터를 구동시킨다. 만약 LG 모터를 구동시키려면 ?
    
```java
public class LGMotor {
    private Door door;
    private MotorStatus motorStatus;
    
    public LGMotor(Door door) {
        this.door = door;
        this.motorStatus = MotorStatus.STOPPED;
    }
    
    private void moveLGMotor(Direction direction) {
        // LG Motor를 구동시킴
    }
    
    public MotorStatus getMotorStatus() {
        return this.motorStatus;
    }
    
    private void setMotorStatus(MotorStatus motorStatus) {
        this.motorStatus = motorStatus;
    }
    
    // 엘레베이터 제어
    public void move(Direction direction) {
        MotorStatus motorStatus = this.getMotorStatus();
        
        // 이미 작동 중이면 아무 작업을 하지 않는다.
        if(motorStatus == MotorStatus.MOVING) {
            return;
        }
        
        DoorStatus doorStatus = this.door.getDoorStatus();
        
        // 만약 문이 열려 있으면 우선 문을 닫는다.
        if(doorStatus == DoorStatus.OPENED) {
            this.door.close();
        }
        
        // LG 모터를 주어진 방향으로 이동시킨다.
        this.moveLGMotor(direction);
        // 모터 상태를 이동 중으로 변경함
        this.setMotorStatus(MotorStatus.MOVING);
    }
}
```  

- HyundaiMotor 와 LGMotor 에는 여러 개의 메서드가 동일하게 구현되어 있다.
    - Door 클래스와의 연관관계
    - motorStatus 필드
    - getMotorStatus, setMotorStatus 메서드
- 중복 코드는 유지보수성을 악화 시키므로 바람직하지 않다.

#### 해결방법
##### 방법 1
2 개 이상의 클래스가 유사한 기능을 제공하면서 중복된 코드가 있는 경우에는 상속을 이용해서 코드 중복 문제를 피할 수 있다.
- ![템플릿 메서드 패턴 방법 1](/../img/designpattern-templatemethod-motor-solution-1-classdiagram.png) 

```java
// HyundaiMotor 와 LGMotor 의 공통적인 기능을 구현하는 클래스
public abstract class Motor {
    protected Door door;
    protected MotorStatus motorStatus;
    
    public Motor(Door door){
        this.door = door;
        this.motorStatus = MotorStatus.STOPPED;
    }

    public MotorStatus getMotorStatus() {
        return this.motorStatus;
    }
    
    protected void setMotorStatus(MotorStatus motorStatus) {
        this.motorStatus = motorStatus;
    }
}
```  

```java
// Motor 를 상속받아 HyundaiMotor 클래스 구현
public class HyundaiMotor extends Motor {
    public HyundaiMotor(Door door) {
        super(door);
    }
    
    private void moveHyundaiMotor(Direction direction) {
        // HyundaiMotor 를 구동 시킴
    }
    
    public void move(Direction direction) {
        // 동일
    }
}
```  

```java
// Motor 를 상속받아 LGMotor 클래스를 구현
public class LGMotor extends Motor {
    public LGMotor(Door door) {
        super(door);
    }
    
    private void moveLGMotor(Direction direction) {
        // LG Motor를 구동 시킴
    }
    
    public void move(Direction direction) {
        // 동일
    }
}
```  

- Motor 클래스를 상위 클래스로 정의함으로써 중복 코드를 제거할 수 있다.
    1. Door 클래스와의 연관관계
    1. motorStatus 필드
    1. getMotorStatus, setMotorStatus 메서드
- 그러나 HyundaiMotor, LGMotor 의 move 메서드는 대부분이 비슷하다.
    - 아직 코드 중복 문제가 있다.
    
##### 방법 2
위의 move 메서드와 같이 부분적으로 중복되는 경우에도 상속을 활용해 코드 중복을 피할 수 있다.
- move 메서드에서 moveHyundaiMotor 메서드와 moveLGMotor 메서드를 호출하는 부문만 다르다.
- moveHyundaiMotor, moveLGMotor 메서드는 기능(모터 구동을 실제로 구현)면에서는 동일하다.
- ![템플릿 메서드 방법 2](/../img/designpattern-templatemethod-motor-solution-2-classdiagram.png)
    1. move 메서드를 상위 Motor 클래스로 이동시킨다.
    1. moveHyundaiMotor 메서드와 moveLGMotor 메서드의 호출 부분을 하위 클래스에서 오버라이드한다.
    
```java
// HyundaiMotor, LGMotor 의 공통적인 기능을 구현한 클래스
public abstract class Motor {
    // 클래스 멤버 변수, 생성자, getter, setter 은 동일
    
    protected abstract void moveMotor(Direction direction);
    
    // HyundaiMotor, LGMotor 의 move 메서드에서 공통되는 부분만을 가짐
    public void move(Direction direction){
        MotorStatus motorStatus = this.getMotorStatus();
        
        if(motorStatus == MotorStatus.MOVING) {
            return;
        }
        
        DoorStatus doorStatus = this.door.getDoorStatus;
        
        if(doorStatus == DoorStatus.OPENED) {
            this.door.close();
        }
        
        moveMotor(direction);
        setMotorStatus(MotorStatus.MOVING);
    }
}
```  

```java
// Motor 클래스를 상속받아 HyundaiMotor 클래스를 구현
public class HyundaiMotor extends Motor {
    public HyundaiMotor(door door) {
        super(door);
    }
    
    @Override
    protected void moveMotor(Direction direction) {
        // Hyundai Motor 를 구동시킴
    }
}
```  

```java
// Motor 를 상속받아 LGMotor 클래스를 구현
public class LGMotor extends Motor {
    public LGMotor(Door door) {
        super(door);
    }
    
    @Override
    protected void moveMotor(Direction direction) {
        // LG Motor 를 구동시킴
    }
}
```  

- Motor 클래스의 move 머세드는 HyundaiMotor, LGMotor 에서 동일한 기능을 구현하면서 각 하위 클래스에서 구체적으로 정의할 필요가 있는 부분, moveMotor 메서드 부분만 각 하위 클래스에서 오버라이드 되도록 한다.
- 이렇게 Template Method 패턴을 이용하면 전체적으로는 동일하면서 부분적으로는 다른 구문으로 구성된 메서드의 코드 중복을 최소화할 수 있다.


### 정리
![템플릿 메서드 정리](/../img/designpattern-templatemethod-motor-conclusion-classdiagram.png)
- AbstractClass : Motor 클래스
- ConcreteClass : HyundaiMotor, LGMotor 클래스
- TemplateMethod : Motor 클래스의 move 메서드
- Primitive(Hook)Operation : move 메서드에서 호출되면서 하위 클래스에서 오버라이드 될 필요가 있는 moveMotor 메서드

---
 
## Reference
[[Design Pattern] 템플릿 메서드 패턴이란](https://gmlwjd9405.github.io/2018/07/13/template-method-pattern.html)
