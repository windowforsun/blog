--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 데코레이터 패턴(Decorator Pattern)"
subtitle: '데코레이터 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Decorator Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 데코레이터 패턴이란
- 객체의 결합을 통해 기능을 동적으로 유연하게 확장 할 수 있게 해주는 패턴
    - 기본 기능에 추가할 수 있는 기능의 종류가 많은 경우에 각 추가 기능을 Decorator클래스로 정의 한 후 필요한 Decorator객체를 조합함으로써 추가 기능의 조합을 설계하는 방식이다.
    - ex) 기본 도로 표시 기능에 차선 표시, 고통량 표시, 교차로 표시, 단속 카메라 표시의 4가지 추가 기능이 있을 때 추가 기능의 모든 조합은 15가지가 된다.
    - 데코레이터 패턴을 이용하여 필요 추가 기능의 조합을 동적으로 생성 할 수 있다.
    - 구조 패턴 중 하나
- ![데코레이터 패턴 예시1](/../img/designpattern-decorator-ex-1-classdiagram.png)
- 역할이 수행하는 작업
    - Component
        - 기본 기능을 뜻하는 ConcreteComponent와 추기 기능을 뜻하는 Decorator의 공통 기능을 정의
        - 클라이언트는 Component를 통해 실제 객체를 사용함
    - ConcreteComponent
        - 기본 기능을 구현하는 클래스
    - Decorator
        - 많은 수가 존재하는 구체적인 Decorator의 공통 기능을 제공
    - ConcreteDecoratorA, ConcreteDecoratorB
        - Decorator의 하위 클래스로 기본 기능에 추가되는 개별적인 기능을 뜻함
        - ConcreteDecorator 클래스는 ConcreteComponent 객체에 대한 참조가 필요한데, 이는 Decorator클래스에서 Component클래스로의 합성(Composition) 관계를 통해 표현됨
    
#### 참고
- 합성(Composition) 관계
    - 생성자에서 필드에 대한 객체를 생성하는 경우
    - 전체 객체의 라이프타임과 부분 객체의 라이프 타임은 의존적이다.
    - 전체 객체가 없어지면 부분 객체도 없어진다.
    
### 예시
#### 도로 표시 방법 조합하기
- ![데코레이터 패턴 도로 1](/../img/designpattern-decorator-road-1-classdiagram.png)
- 내비게이션 SW에서 도로를 표시하는 기능
  - 도로를 간단한 선으로 펴시하는 기능(기본 기능)
  - 내비게이션 SW에 따라 도로의 차선을표시하는 기능(추가 기능)
  
  ```java
  // 기본 도로 표시 클래스
  public class RoadDisplay {
    public void draw() {  
      System.out.println("기본 도로 표시");
    }
  }

  // 기본 도로 표시 + 차선 표시 클래스
  public class RoadDisplayWithLane extends RoadDisplay {
    @Override
    public void draw() {
      super.draw();
      this.drawLane();
    }
  
    private void drawLane() {
      System.out.println("차선 표시");
    }
  }
  ```  
  
  ```java
  public class Client {
    public static void main(String[] args){
      RoadDisplay roadDisplay = new RoadDisplay();
      // 기본 도로만 표시
      roadDisplay.draw();
      
      RoadDisplay roadDisplayWithLane = new RoadDisplayWithLane();
      // 기본 도로 표시 + 차선 표시
      roadDisplayWithLane.draw();
    }
  }
  ```  
  
- RoadDisplay 클래스에는 기본 도로 표시 기능을 시리행하기 위한 draw 메서드를 구한다.
- RoadDisplayWithLane클래스에는 차선 표시 기능을 추가하기 위해 상속받은 draw메서드를 오버라이드 한다.
  - 기본 도로 표시 기능 : 상위 클래스(RoadDisplay)의 draw메서드 호출
  - 차선 표시 기능 : 자신의 drawLane 메서드 호출
  
#### 문제점
1. 또 다른 도로 표시 기능을 추가로 구현하는 경우
  - 기본 도로 표시에 교통량을 표시하고 싶다면?
  - ![데코레이터 패턴 도로 문제 1](/../img/designpattern-decorator-problem-1-classdiagram.png)
  
  ```java
  // 기본 도로 표시 + 교통량 표시 클래스
  public class RoadDisplayWithTraffic extends RoadDisplay {
    public void draw() {
      // 기본 도로 표시
      super.draw();
      // 추가적으로 교통량 표시
      this.drawTraffic(); 
    }
    
    private void drawTraffic() {
      System.out.println("교통량 표시");
    }
  }
  ```  
  
1. 여러 가지 추가 기능을 조합해야 하는 경우
  - 기본 도로 표시에 차선 표시 기능과 교통량 포시 기능을 함께 제공하고 싶다면?
   - | 경우 | 기본기능 | 차선표시 | 교통량표시 | 교차로표시 | 클래스 이름 |
      |:---:|:---:|:---:|:---:|:---:|:---:|
      | 1 | O | | | | RoadDisplay 
      | 2 | O | O | | | RoadDisplayWithLane 
      | 3 | O | | O | | RoadDisplayWithTraffic 
      | 4 | O | | | O | RoadDisplayWithCrossing 
      | 5 | O | O | O | | RoadDisplayWithLaneTraffic 
      | 6 | O | O | | O | RoadDisplayWithLaneCrossing 
      | 7 | O | | O | O | RoadDisplayWithTrafficCrossing 
      | 8 | O | O | O | O | RoadDisplayWithLaneTrafficCrossing 
      
    - ![데코레이터 패턴 도로 문제 2](/../img/designpattern-decorator-problem-2-classdiagram.png)
    - 위와 같이 상속을 통해 조합의 가가 경우를 설계한다면 각 조합별로 하위 클래스(7개)를 구현해야 한다.
    - 다양한 기능의 조합을 고려해야 하는 경우 상속을 통한 기능 확장은 각 기능별 클래스를 추가해야 한다는 단점이 있다.
    
#### 해결 방법
문제를 해결하기 위해서는 각 추가 기능별로 개별적인 클래스를 설계하고 기능을 조합할 때 각 클래스의 객체 조합을 이용하면 된다.
- ![데코레이터 패턴 도로 해결 1](/../img/designpattern-decorator-solution-1-classdiagram.png)
  - 도로를 표시하는 기본 기능만 필요한 경우 RoadDisplay 객체를 이용한다.
  - 차선을 표시하는 추가 기능도 필요한 경우 RoadDisplay 객체와 LaneDecorator객체를 이용한다.
    1. LaneDecorator에서는 차선 표시 기능마나 지기접 제공: drawLane()
    1. 도로 표시 기능은 RoadDisplay클래스의 draw메서드를 호출: super.draw()
      - DisplayDecorator 클래스에서 Display클래스로 합성(Composition)관계를 통해 RoadDisplay객체에 대한 참조
      
- Display 추상 클래스

```java
public abstract class Display {
  public abstract void draw();
}
```  

- RoadDisplay 클래스

```java
// 기본 도로 표시 클래스
public class RoadDisplay extends Display {
  @Override
  public void draw() {
    System.out.println("기본 도로 표시");
  }
}
```  

- DisplayDecorator클래스

```java
// 다양한 추가 기능에 대한 공통 클래스
public abstract class DisplayDecorator extends Display {
  private Display decoratorDisplay;
  
  // 합성(Composition) 관계를 통해 RoadDisplay 객체에 대한 참조
  public DisplayDecorator(Display decoratorDisplay) {
    this.decoratorDisplay = decoratorDisplay;
  }
  
  @Override
  public void draw() {
    this.decoratorDisplay.draw();
  }
}
```  

- LaneDecorator, TrafficDecorator 클래스

```java
// 차선 표시를 추가하는 클래스
public class LaneDecorator extends DisplayDecorator {
  // 기본 표시 클래스의 설정
  public LaneDecorator(Display decoratorDisplay) {
    super(decoratorDisplay);
  }
  
  @Override
  public void draw() {
    // 설정된 기본 표시 기능
    super.draw();
    // 추가적으로 차선 표시
    drawLane();
  }
  
  private void drawLane() {
    System.out.println("차선 표시");
  }
}

// 교통량 표시를 추가하는 클래스
public class TrafficDecorator extends DisplayDecorator {
  // 기존 표시 클래스의 설정
  public TrafficDecorator(Display decoratorDisplay) {
    super(decoratorDisplay);
  }
  
  @Override
  public void draw() {
    // 설정된 기본 기능 표시
    super.draw();
    // 추가적으로 교통량 표시
    drawTraffic();
  }
  
  private void drawTraffic() {
    System.out.println("교툥량 표시");
  }
}
```  

- 클라이언트에서의 사용

```java
public class Client {
  public static void main(String[] args){
    Display display = new RoadDisplay();
    // 기본 도로 표시
    road.draw();
    
    Display roadWithLane = new LaneDecorator(new RoadDisplay());
    // 기본 도로 + 차선 표시
    roadWithLane.draw();
    Display roadWithTraffic = new TrafficDecorator(new RoadDisplay());
    // 기본 도로 + 교통량 포시
    roadWithTraffic.draw();
  }
}
```  

- 각 road, roadWithLane, roadWithTaffic 객체의 접근이 모두 Display 클래스를 통해 이루어 진다.
- 어떤 기능을 추가하냐느에 관계없이 Client클래스는 동일한 Display클래스만을 통해 일관성있는 방식으로 도로 정보를 표시할 수 있다.
- 이렇게 Decorator 패턴을 이용하면 추가 기능 조합별로 별도의 클래스를 구현하는 대신 각 추가 기능에 해당하는 클래스의 객체를 조합해 추가 기능의 조합을 구현할 수 있개 된다.
  - 이 설계는 추가 기능의 수가 많을수록 효과가 크다.
- ![데코레이터 패턴 정리 1](/../img/designpattern-decorator-conclusion-1-classdiagram.png)

#### 추가 예시
- 기본 도로 표시 + 차선 표시 + 교통량 표시

```java
public class Client {
  public static void main(String[] args){
    // 기본 도로 + 차선 + 교통량
    Display roadWithLaneAndTraffic = new TrafficDecorator(new LaneDecorator(new RoadDisplay()));
    roadWithLaneAndTraffic.draw();
  }
}
```  

1. 가장 먼저 생성된 RoadDisplay객체의 draw메서드가 실행
1. 첫 번째 추가 기능인 LaneDecorator 클래스의 drawLane메서드가 실행
1. 두 번째 추가 기능인 TrafficDecorator 클래스의 drawTraffic메서드가 실행

- 교차로를 표시하는 추가 기능을 지원하면서 기존의 다른 추가기능(차선, 교통량)과의 조합을 지원
  - CorssingDecorator 클래스
  
  ```java
  // 교차로 표시를 추가하는 클래스
  public class CrossingDecorator extends DisplayDecorator {
    // 기존 클래스의 설정
    public CrossingDecorator(Display decoratorDisplay) {
      super(decoratorDisplay);
    }
  
    @Override
    public void draw() {
      // 설정된 기존 표시 기능
      super.draw();
      // 추가적으로 교차로를 표시
      this.drawCrossing();
    }
  
    private void drawCrossing() {
      System.out.println("교차로 표시");
    }
  }
  ```  
  
  ```java
  public class Client {
    public static void main(String[] args){
      // 기본 도로 + 차선 + 교통량 + 교차로 표시
      Display roadWithLaneTrafficCrossing = new CrossingDecorator(new TrafficDecorator(new LaneDecorator(new RoadDisplay())));
      roadWithLaneTrafficCrossing.draw();
    }
  }
  ```  
  
  - CrossingDecorator를 DisplayDecorator의 하위 클래스로 설계한다.
    - CrossingDecorator의 draw메서드가 호출되면 우선 상위 클래스(DisplayDecorator)의 draw 메서드를 호출한 후 CrossingDecorator의 drawCrossing메서드를 호출한다.
  - roadWithLaneTrafficCrossing객체의 draw메서드를 호출하면
    1. 가장 먼저 생성된 RoadDisplay 객체의 draw 메서드 실행
    1. 첫 번째 추가 기능인 LaneDecorator 클래스의 drawLane 메서드 실행
    1. 두 번째 추가 기능인 TrafficDecorator 클래스의 drawTraffic 메서드 실행
    1. 세 번째 추가 기능인 CrossingDecorator 클래스의 drawCrossing 메서드 실행


---
 
## Reference
[[Design Pattern] 데코레이터 패턴이란](https://gmlwjd9405.github.io/2018/07/09/decorator-pattern.html)
