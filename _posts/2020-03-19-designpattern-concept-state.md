--- 
layout: single
classes: wide
title: "[DesignPattern 개념] State Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '객체의 상태를 개별 클래스로 표현하고 관리하는 State 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - State
use_math : true
---  

## State 패턴이란
- `State` 는 상태라는 의미와 같이, 상태를 클래스로 표현해서 상태에 따른 동작을 보다 유연하게 구성할 수 있는 패턴이다.

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_1.png)

- `State` : 상태를 나타내는 역할로, 각 상태에서 수행해야하는 동작인 메소드를 정의하는 추상 클래스나 인터페이스이다.
- `ConcreteState` : `State` 의 구현체로 하나의 상태가 수행해야하는 동작을 구현한 클래스이다.
- `Context` : 전체적인 상태를 가지는 역할로 `ContextState` 를 가지고, 상태를 컨트롤 할 수 있는 메소드를 정의하거나 구현한다.

### 분할 통지
- `State` 패턴은 분할 통치기법은 `divide and conquer` 방식을 사용한다.
- 하나의 동작에 대해 모든 상태를 고려해서 개발하는 방식이 아닌, 각 상태에 따른 동작으로 분할해 전체적인 구현을 해나간다.
- 다수의 상태가 존재할 경우, 복잡하게 분기마다 구현된 처리를 클래스로 분할해 처리하게 된다.

### 상태에 의존한 처리
- `State` 패턴은 상태를 추상 클래스, 인터페이스를 사용해서 메소드를 선언하고, 각 하위 클래스에서 이를 구현한다.
- 상태를 컨트롤 할때 추상 클래스의 메소드를 호출하는 방식으로 실제 동작을 위임한다.

### 상태전환 관리
- `State` 패턴에서 상태를 관리하는 방법은 아래와 같다.
- 각 상태 클래스에서 자신의 상태를 관리한다.
	- 각 상태가 다른 상태 클래스에 의존하게 된다는 단점이 있다.
- `Mediator` 패턴과 같은 다른 패턴을 적용해 관리한다.
- 상태 테이블을 구성해 관리한다.

### 새로운 상태 추가
- `State` 패턴에서 새로운 상태를 추가하는 것은 간단하다.
- `State` 의 하위로 새로운 상태 클래스를 만들고, 해당 상태에서 수행해야하는 동작을 구현한다.
- 상태의 전의를 위해 다른 상태 클래스에 새로운 상태 클래스를 추가해준다.

### 기타
- 다양한 변수값을 통해 판별되는 상태를 하나의 클래스로 지칭한다.

## 컴퓨터 배터리 상태 관리
- 컴퓨터 배터리의 잔량에 따라 상태를 나눠 관리한다고 가정한다.
- 배터리 잔량에 따른 상태는 아래와 같다.
	- 100 ~ 81 : 고
	- 80 ~ 31 : 중
	- 30 ~ 0 : 저
- 고, 중, 고 배터리 상태에 따라 CPU 성능과 RAM 의 성능을 조절한다.
	- 고 : 100%
	- 중 : 70%
	- 저 : 40%
- 성능조절은 CPU 는 클럭수를 제한하고, RAM 은 대역폭을 제한한다고 가정한다.
- `State` 패턴을 사용하지 않는다면 아래와 같이 구현할 수 있다는 점을 참고한다.

	```java
    class Computer {
	    public void setMaxCpuClock(double battery) {
	        if(battery > 80 && battery <= 100) {
	            // 배터리 고
	            // 성능 조절
	        } else if(battery > 30 && battery <= 80) {
	            // 배터리 중
	            // 성능 조절
	        } else {
	            // 배터리 저
	            // 성능 조절
	        }
	    }
	    
	    public void setMaxRamBandwidth(double battery) {
	        if(battery > 80 && battery <= 100) {
	            // 배터리 고
	            // 성능 조절
	        } else if(battery > 30 && battery <= 80) {
	            // 배터리 중
	            // 성능 조절
	        } else {
	            // 배터리 저
	            // 성능 조절
	        }
	    }
	    
	    // ...
    }
    ```  
    
    - 배터리관리를 보다 세부적으로 하기위해 상태를 세분화하게되면, 각 동작을 수행하는 메소드의 분기가 복잡해 질 수 있다.


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_2.png)

### State

```java
public interface State {
    void checkState(Context context, double battery);
    double getMaxCpuClock(double cpuClock);
    double getMaxRamBandwidth(double ramBandwidth);
    boolean isState(double battery);
}
```  

- `State` 배터리 상태를 나타내면서 가져야할 기능적인 메소드를 정의하는 인터페이스이다.
- `State` 패턴에서 `State` 역할을 수행한다.
- 하나의 배터리 상태는 현재 상태 체크, CPU, RAM 조절, 자신의 상태정보 판별 기능이 있다.
- 상태에 대한 기능들은 하위 클래스에서 각 상태에 맞춰 구현한다.

### HighState, MiddleState, LowState

```java
public class HighState implements State {
    private static HighState instance = new HighState();

    private HighState() {}

    public static HighState getInstance() {
        return instance;
    }

    @Override
    public void checkState(Context context, double battery) {
        if(MiddleState.getInstance().isState(battery)) {
            context.changeState(MiddleState.getInstance());
        } else if(LowState.getInstance().isState(battery)){
            context.changeState(LowState.getInstance());
        }
    }

    @Override
    public double getMaxCpuClock(double cpuClock) {
        return cpuClock * 1;
    }

    @Override
    public double getMaxRamBandwidth(double ramBandwidth) {
        return ramBandwidth * 1;
    }

    @Override
    public boolean isState(double battery) {
        return battery > 80 && battery <= 100;
    }
}
```  

```java
public class MiddleState implements State{
    private static MiddleState instance = new MiddleState();

    private MiddleState() {}

    public static MiddleState getInstance() {
        return instance;
    }

    @Override
    public void checkState(Context context, double battery) {
        if(HighState.getInstance().isState(battery)) {
            context.changeState(HighState.getInstance());
        } else if (LowState.getInstance().isState(battery)){
            context.changeState(LowState.getInstance());
        }
    }

    @Override
    public double getMaxCpuClock(double cpuClock) {
        return cpuClock * 0.7;
    }

    @Override
    public double getMaxRamBandwidth(double ramBandwidth) {
        return ramBandwidth * 0.7;
    }

    @Override
    public boolean isState(double battery) {
        return battery > 30 && battery <= 80;
    }
}
```  

```java
public class LowState implements State {
    private static LowState instance = new LowState();

    private LowState(){}

    public static LowState getInstance() {
        return instance;
    }

    @Override
    public void checkState(Context context, double battery) {
        if(HighState.getInstance().isState(battery)) {
            context.changeState(HighState.getInstance());
        } else if(MiddleState.getInstance().isState(battery)){
            context.changeState(MiddleState.getInstance());
        }
    }

    @Override
    public double getMaxCpuClock(double cpuClock) {
        return cpuClock * 0.4;
    }

    @Override
    public double getMaxRamBandwidth(double ramBandwidth) {
        return ramBandwidth * 0.4;
    }

    @Override
    public boolean isState(double battery) {
        return battery >= 0 && battery <= 30;
    }
}
```  

- `HighState`, `MiddleState`, `LowState` 는 `State` 의 구현체로 각 상태를 나타내는 클래스이다.
- `State` 패턴에서 `ConcreteState` 역할을 수행한다.
- 구현 클래스(각 상태)는 [Singleton 패턴]({{site.baseurl}}{% link _posts/2020-01-11-designpattern-concept-singleton.md %})
으로 구현 돼있다.
- `checkState()` 메소드는 `Context` 의 정보를 기반으로 현재 상태가 자신의 상태에서 전환을 해야 할지 판별하고 상태 전환을 수행한다.
	- 현재 구조에서 `State` 패턴을 사용하지 않을 때와 다른 점 중 하나는 상태를 판별하고 상태를 결정하는 부분이 각 상태를 나타내는 구현 클래스에 분산돼 있다는 것이다.
- `getMaxCpuClock()`, `getMaxRamBandwidth()` 메소드는 현재의 상태에 따라 인자값으로 받은 성능적인 값을 퍼센트로 계산해 리턴해준다.
	- 상태에 대한 분기 없이 구현 클래스에서 구현하는 상태에 맞게 메소드를 구현하면 된다.
- `isState()` 는 `Context` 의 정보를 기반으로 현재 상태가 자신의 상태인지 판별해 결과를 리턴한다.

### Context 

```java
public interface Context {
    void setBattery(double battery);
    void changeState(State state);
    void setMaxCpuClock();
    void setMaxRamBandwidth();
    double getPerformanceScore();
}
```  

- `Context` 는 배터리 상태를 포함하는 객체의 동작을 나타내는 인터페이스이다.
- `State` 패턴에서 `Context` 역할을 수행한다.

### Computer

```java
public class Computer implements Context {
    private State state;
    private double battery;
    private double cpuClock;
    private double ramBandwidth;
    private double maxCpuClock;
    private double maxRamBandwidth;

    public Computer(double battery, double cpuClock, double ramBandwidth) {
        this.cpuClock = cpuClock;
        this.ramBandwidth = ramBandwidth;
        this.changeState(HighState.getInstance());
        this.setBattery(battery);
    }

    @Override
    public void setBattery(double battery) {
        this.battery = battery;
        this.state.checkState(this, battery);
    }

    @Override
    public void changeState(State state) {
        this.state = state;
        this.setMaxCpuClock();
        this.setMaxRamBandwidth();
    }

    @Override
    public void setMaxCpuClock() {
        this.maxCpuClock = this.state.getMaxCpuClock(this.cpuClock);
    }

    @Override
    public void setMaxRamBandwidth() {
        this.maxRamBandwidth = this.state.getMaxRamBandwidth(this.ramBandwidth);
    }

    @Override
    public double getPerformanceScore() {
        return this.maxCpuClock * this.maxRamBandwidth;
    }
}
```  

- `Computer` 는 `Context` 의 구현체로 배터리의 상태를 컨트롤하는 컴퓨터를 나타내는 클래스이다.
- `State` 패턴에서 `Context` 역할을 수행한다.
- `state` 필드를 통해 현재 자신의 상태를 저장하고 관리한다.
- `battery` 필드는 는 현재 컴퓨터의 배터리를 저장하고 괸리한다.
- `cpuClock`, `ramBandwidth` 필드는 CPU, RAM 이 가지는 스펙을 의미한다.
- `maxCpuClock`, `maxRamBandwidth` 필드는 상태에 따라 조절하는 CPU, RAM 의 스펙의 최대치이다.
- 컴퓨터가 생성자를 통해 만들어지면 처음에는 `HighState`(100) 으로 설정하고, 인자값으로 받은 `battery` 에 따라 다시 상태를 설정한다.
- `setBattery()` 메소드는 배터리 필드를 설정하는 역할로, 인자값의 배터리 수치를 설정하고, `state` 필드(현재 상태)의 `checkState()` 메소드를 호출해 상태를 검사하고 젼환한다.
- `changeState()` 메소드는 현재 배터리수치에 따라 컴퓨터의 상태를 전환하는 역할로, `state` 필드에 새로운 상태 인스턴스로 전환하고 `setMaxCpuClock()`, `setMaxRamBandwidth()` 메소드르 통해 상태에 따른 성능을 조절한다.
- `setMaxCpuClock()`, `setMaxRamBandwidth()` 메소드는 설정된 현재 상태(`state`) 에게 상태에 따른 성능 수치를 설정한다.
- `getPerformance()` 메소드는 `max` 값을 바탕으로 현재 컴퓨터의 성능 점수를 계산해서 리턴한다.

### State 의 처리과정
- `Computer` 인스턴스를 생성할 때 생성자의 `battery` 값은 100이다.
- `setBattery()` 메소드를 통해 `Computer` 의 배터리를 70, 20 으로 설정 했을 때 상태 변화의 과정이다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_2.png)

### 테스트

```java
public class StateTest {
    @Test
    public void Computer_Battery_100_HighStatePerformance() {
        // given
        double battery = 100;
        double cpuClock = 100;
        double ramBandwidth = 100;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * ramBandwidth));
    }

    @Test
    public void Computer_Battery_81_HighStatePerformance() {
        // given
        double battery = 81;
        double cpuClock = 100;
        double ramBandwidth = 100;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * ramBandwidth));
    }

    @Test
    public void Computer_Battery_80_MiddleStatePerformance() {
        // given
        double battery = 80;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.7;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_31_MiddleStatePerformance() {
        // given
        double battery = 31;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.7;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_30_LowStatePerformance() {
        // given
        double battery = 30;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.4;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_0_LowStatePerformance() {
        // given
        double battery = 0;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.4;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_HighToMiddle_MiddleStatePerformance() {
        // given
        double battery = 81;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.7;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);
        computer.setBattery(80);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_MiddleToHigh_HighStatePerformance() {
        // given
        double battery = 80;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 1;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);
        computer.setBattery(81);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_MiddleToLow_LowStatePerformance() {
        // given
        double battery = 31;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.4;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);
        computer.setBattery(30);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }

    @Test
    public void Computer_Battery_LowToMiddle_MiddleStatePerformance() {
        // given
        double battery = 30;
        double cpuClock = 100;
        double ramBandwidth = 100;
        double stateRate = 0.7;
        Computer computer = new Computer(battery, cpuClock, ramBandwidth);
        computer.setBattery(31);

        // when
        double actual = computer.getPerformanceScore();

        // then
        assertThat(actual, is(cpuClock * stateRate * ramBandwidth * stateRate));
    }
}
```  


---
## Reference

	