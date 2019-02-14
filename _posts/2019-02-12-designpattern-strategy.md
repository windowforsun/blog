--- 
layout: post
title: "디자인 패턴(Design Pattern) 스트래티지 패턴(Strategy Pattern)"
subtitle: '스트래티지 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
tags:
    - 디자인 패턴
    - Design Pattern
    - Strategy Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 스트래티지 패턴 이란
- 행위를 클래스로 캡슐화해 동적으로 행위를 자유롭게 바꿀 수 있게 해주는 패턴
    - 같은 문제를 해결하는 여러 알고리즘이 클래스 별로 캡슐화되어 있고 이들이 필요할 때 교체할 수 있도록 함으로써 동일한 문제를 다른 알고리즘으로 해결할 수 있게 하는 디자인 패턴
    - 행위(Behavioral) 패턴 중 하나
- 전략을 쉽게 바꿀 수 있도록 해주는 디자인 패턴
    - 전략이란 어떤 목적을 달성하기 위해 일을 수행하는 방식, 비지니스 규칙, 문제를 해결하는 알고리즘 등
- 특히 게임 프로그래밍에서 게임 캐릭터가 자신이 처한 상황에 따라 공격이나 행동하는 방식을 바꾸고 싶을 때 스트래지 패턴은 매우 유용하다.
- ![스트래티지 패턴 예시](/img/designpattern-strategy-ex-classdiagram.png)
- 역할이 수행하는 작업
    - Strategy
        - 인터페이스나 추상 클래스로 외부에서 동일한 방식으로 알고리즘을 호출하는 방법을 명시
    - ConcreteStrategy
        - 스트래티지 패턴을 이용하는 역할을 수행한다.
        - 필요에 따라 동적으로 구체적인 전략을 바꿀 수 있도록 setter 메서드(집약관계)를 제공한다.

#### 참고
- 집약 관계
    - 참조값을 인자로 받아 필드를 세팅하는 경우
    - 전체 객체의 라이프타임과 부분 객체의 라이프 타임은 독립적이다.
    - 전체 객체가 메모리에서 사라진다 해도 부분 객체는 사라지지 않는다.
    
### 예시
#### 로봇 만들기
- ![스트래티지 패턴 로봇 1](/../img/designpattern-strategy-robot-1-classdiagram.png)

```java
public abstract class Robot {
    private String name;
    
    public Robot(String name) {
        this.name = name;
    }
    
    public String getName() {
        return this.name;
    }
    
    // 추상 메서드
    public abstract void attack();
    public abstract void move();
}

public class TaekwonV extends Robot {
    public TaekwonV(String name) {
        super(name);
    }
    
    @Override
    public void attack() {
        System.out.println("I have Missile");
    }
    
    @Override
    public void move() {
        System.out.println("I can only walk");
    }
}

public class Atom extends Robot {
    public Atom(String name) {
        super(name);
    }
    
    @Override
    public void attack() {
        System.out.println("I have strong punch");
    }
    
    @Override
    public void move() {
        System.out.println("I can fly");
    }
}
```  

```java
public class Client {
    public static void main(String[] args){
        Robot taekwonV = new TaekwonV("TaekwonV");
        Robot atom = new Atom("Atom");
        
        System.out.println("My name is " + tasekwonV.getName());
        taekwonV.move();
        taekwonV.attack();
        
        System.out.println();
        
        System.out.println("My name is " + atom.getName());
        atom.move();
        atom.attack();
    }
}
```  

#### 문제점
1. 기존 로봇의 공격과 이동 방법을 수정하는 경우
    - Atom이 날 수 없고, 걷게 만들고 싶다면 ?
    - TaekwonV 를 날게 하려면 ?
    ```java
    public class Atom extends Robot {
        public Atom(String name) {
            super(name);
        }
        
        @Override
        public void attack() {
            System.out.println("I can strong punch");
        }
        
        @Override
        public void move() {
            System.out.println("I can only walk") // 수정 필요
        }        
    }
    ```  
    
    - 새로운 기능으로 변경하려고 기존 코드의 내용을 수정해야 하므로 OCP에 위배된다.
    - 또한 TaekwonV와 Atom의 move() 메서드의 내용이 중복된다. 이런 중복 상황은 많은 문제를 야기하는 원인이 된다.
    - 만약 걷는 방식에 문제가 있거나 새로운 걷는 방식으로 수정하려면 모든 중복 코드를 일관성 있게 변경해야만 한다.
    
1. 새로운 로봇을 만들어 기존 공격(attack) 또는 이동 방법(move)을 추가/수정하는 경우
    - 새로운 로봇으로 Sungard를 만들어 TaekwonV의 미사일 공격 기능을 추가하려면 ?
    ```java
    public class Sungard extends Robot {
        public Sungard(String name) {
            super(name);
        }
        
        @Override
        public void attack() {
            System.out.println("I have have Missile");
        }
        
        @Override
        public void move() {
            System.out.println("I can only walk");
        }
    }
    ```  
    
    - TaekwonV와 Sungard클래스의 attack() 메서드의 내용이 중복된다.
    - ![스트래티지 패턴 예시 문제 1](/../img/designpattern-strategy-robot-problem-1-classdiagram.png)
    - 현재 시스템의 캡슐화의 단위가 Robot 자체이므로 로봇을 추가하기는 매우 쉽다.
    - 그러나 새로운 로봇인 Sungard에 기존의 공격 또는 이동 방법을 추가하거나 변경하려고 하면 문제가 발생한다.
    
#### 해결 방법
문제를 해결하기 위해서는 무엇이 변화되었는지 찾은 후에 이를 클래스로 캡슐화해야 한다.

- 로봇 예제에서 변화되면서 문제를 발생시키는 요인은 로봇의 이동 방식과 공격 방식의 변화이다.
- 이를 캡슐화하려면 외부에서 구체적인 이동 방식과 공격방식을 담은 구체적인 크래스들을 은닉해야 한다.
    - 공격과 이동을 위한 인터페이스를 각각 만들고 이들을 실제 실현한 클래스를 만들어야 한다.
- ![스트래티지 패턴 로봇 해결 1](/../img/designpattern-strategy-robot-solution-1-classdiagram.png)
    - Robot클래스가 이동 기능과 공격 기능을 이용하는 클라이언트 역할을 수행
        - 구체적인 이동, 공격 방식이 MovingStrategy, AttackStrategy 인터페이스에 의해 캡슐화 됨
        - 인터페이스들이 일종의 방화벽 역할을 수행해 Robot 클래스의 변경을 차단해 준다.
    - 스트래티지 패턴을 이용하면 새로운 기능 추가(새로운 이동, 공격 기능)가 기존의 코드에 영항을 미치지 못하게 하므로 OCP를 만족 하는 설계가 된다.
        - 이렇게 변경된 새로운 구조에서는 외부에서 로봇 객체의 이동, 공격 방식을 임의대로 바꾸도록 해주는 setter 매서드가 필요하다. (setMovingStrategy, setAttackStrategy)
        - 변경이 가능한 이유는 상속 대신 집약 관계를 이용했기 때문이다.
- Robot 클래스  
```java
public abstract class Robot {
    private String name;
    private AttackStrategy attackStrategy;
    private MovingStrategy  movingStrategy;
    
    public Robot(String name) {
        this.name = name;
    }
    
    public String getName() {
        return this.name;
    }
    
    public void attack(){
        this.attackStrategy.attack();
    }
    
    public void moving() {
        this.movingStrategy.move();
    }
    
    // 집약 관계, 전체 객체가 메모리에서 사라진다 해도 부분 객체는 사라지지 않는다.
    // setter 메서드
    public void setAttackStrategy(AttackStrategy attackStrategy) {
        this.attackStrategy = attackStrategy;
    }
    
    public void setMovingStrategy(MovingStrategy movingStrategy){
        this.movingStrategy = movingStrategy;
    }
}
```  

- 구체적인 Robot 클래스

```java
public class TaekwonV extends Robot {
    public TaekwonV(String name) {
        super(name);
    }
}

public class Atom extends Robot { 
    public Atom(String name) {
        super(name);
    }
}

public class Sungard extends Robot {
    public Sungard(String name) {
        super(name);
    }
}
```  

- 공격, 이동 기능에 대한 인터페이스와 구체적인 클래스

```java  
interface AttackStrategy {
    public void attack();
}

interface MovingStrategy {
    public void move();
}

public class MissileStrategy implements AttackStrategy {
    public void attack() {
        System.out.println("missile attack");
    }
}

public class PunchStrategy implements AttackStrategy {
    public void attack() {
        System.out.println("punch attack");
    }
}

public class FlyingStrategy implements MovingStrategy {
    public void move() {
        System.out.println("fly move");
    }
}

public class WalkingStrategy implements MovingStrategy {
    public void move() {
        System.out.println("walk move");
    }
}
```  

- 클라이언트에서의 사용  
```java  
public class Client {
    public static void main(String[] args) {
        Robot taekwonV = new TaekwonV("TaekwonV");
        Robot atom = new Atom("Atom");
        Robot sungard = new Sungard("Sungard");
        
        // 초기 버전에서 수정된 부분
        // 전략(공격, 이동) 변경 방법
        taekwonV.setMovingStrategy(new WalkingStrategy());
        taekwonV.setAttackStrategy(new MissileStrategy());
        atom.setMovingStrategy(new FlyingStrategy());
        atom.setAttackStrategy(new PunchStrategy());
        sungard.setMovingStrategy(new WakingStrategy());
        sungard.setAttackStrategy(new PunchStrategy());
        
        // 초기 버전과 동일
        System.out.println("My name is " + taekwonV.getName());
        taekwonV.move();
        taekwonV.attack();
        
        System.out.println();
        
        System.out.println("My name is " + atom.getName());
        atom.move();
        atom.attack();
        
        System.out.println();
        
        System.out.println("My name is " + sungard.getName());
        sungard.move();
        sungard.attack();
    }
}
```


---
 
## Reference
[[Design Pattern] 스트래티지 패턴이란](https://gmlwjd9405.github.io/2018/07/06/strategy-pattern.html)
