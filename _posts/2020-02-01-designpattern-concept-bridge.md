--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Bridge Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '서로 다른(분리된) 클래스 계층을 이어주는 Bridge 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Bridge
---  

## Bridge 패턴이란
- `Bridge` 는 다리라는 의미를 가진 것처럼, 디자인 패턴에서도 어떤 양끝을 이어주는 역할을 한다.

### Bridge 패턴의 양끝
- `Bridge` 패턴이 이어주는 양끝은 `기능의 클래스 계층` 과 `구현의 클래스 계층` 이다.
- `Bridge` 패턴은 `기능의 클래스 계층` 과 `구현의 클래스 계층` 사이를 이어주는 다리 역할을 한다.

#### 기능의 클래스 계층
- `기능의 클래스 계층` 은 새로운 기능을 추가하고 싶은 경우이다.
- `Base` 라는 클래스가 있을 때, 이 클래스가 가지고 있던 기능 외에 다른 기능을 추가하고 싶을 때(메소드를 추가하고 싶을 떄) 상속을 통해 이를 구현한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_0_1.png)

- 이러한 계층을 `기능의 클래스 계층` 이라 하고 아래와 같은 특징을 가지고 있다.
	- 상위 클래스는 기본적인 기능을 가지고 있다.
	- 하위 클래스에서 새로운 기능을 추가한다.
- `BaseGood` 클래스에 새로운 기능을 추가하게 되면 아래와 같이 계층의 깊이가 더 깊어진다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_0_2.png)

- `기능의 클래스 계층` 에서 기능을 추가할 때 계층안에서 추가하는 기능의 목적과 가까운 클래스를 찾아 하위 클래스를 만든다.
- 계층을 구성할 때 계층의 깊이가 너무 깊어지지 않도록 주의해야 한다.

#### 구현의 클래스 계층
- `구현의 클래스 계층` 은 새로운 구현을 추가하고 싶은 경우이다.
- 추상 클래스나 인터페이스가 있을 때, 하위 클래스에서 실제 구현을 하기 때문에 선언과 구현을 분리해 보다 유연한 구조를 만들 수 있다.
- 추상 클래스인 `AbstractClass` 가 있고 이를 구현하는 `ConcreteClass` 가 있을 때 아래와 같은 클래스 계층이 만들어 진다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_0_3.png)

- `ConcreteClass` 에서는 새로운 기능, 메소드를 추가하는 것이 아닌 상위 클래스에 선언된 것을 실제로 구현하는 일을 한다.
- 이러한 계층을 `구현의 클래스 계층` 이라 하고 아래와 같은 특징을 가지고 있다.
	- 상위 클래스는 추상 메소드에 의히 인터페이스를 정의한다.
	- 하위 클래스는 추상 메소드를 실제로 구현한다.
- `AbstractClass` 의 다른 구현을 만들게 되면 아래와 같은 계층이 된다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_0_4.png)

- `구현의 클래스 계층` 에서 새로운 구현을 추가할 때는 추상 클래스의 새로운 하위 클래스를 만든다.

### 클래스 계층의 분리와 연결
- `기능의 클래스 계층` 과 `구현의 클래스 계층` 이 있는 것처럼 클래스 계층을 만들려고 할때, 자신의 목적과 의도롤 명확하게 해서 알맞는 선택을 해야 한다.
- 어떤 클래스 계층에 `기능의 클래스 계층` 과 `구현의 클래스 계층` 이 혼재하게 된다면 복잡한 클래스 계층이 되고, 클래스 계층을 파악하는 것과 확장에 어려움이 있다.
- `기능의 클래스 계층` 과 `구현의 클래스 계층` 을 독립된 클래스 계층으로 구성 했을 때, 이 두 계층을 연결 해주는 것이 바로 `Bridge` 패턴이다.

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_1.png)

- `Abstraction` : `기능의 클래스 계층` 의 최상위 클래스로 `Implementor` 의 메소드를 사용해서 기본적인 기능만 있는 클래스이다.
- `RefineAbstractClass` : `Abstraction` 클래스에 기능을 추가한 클래스이다.
- `Implementor` : `구현의 클래스 계층` 의 최상위 클래스로, `Abstraction` 클래스의 기능을 구현에 필요한 메소드가 선언돼 있다.
- `ConcreteImplementor` : `Implementor` 에 선언된 메소드를 실제로 구현하는 클래스이다.

### 클래스 계층 분리의 장점
- 클래스 계층을 분리하게 되면 계층을 독립적으로 확장 할 수 있다는 장점이 있다.
- 기능 추가가 필요할 때는 `기능의 클래스 계층` 에 클래스를 추가하고, 새로운 구현이 필요하다면 `구현의 클래스 계층에` 클래스를 추가하면 된다.
- 독립적으로 확장이 가능하기 때문에, 한 클래스 계층의 확장에 있어서 다른 클래스 계층에 영향을 주지 않는다.

### 상속(extends)과 위임(delegation)
- 상속은 손쉽게 클래스를 확장 할 수 있지만, 두 클래스를 강하게 연결시키게 된다.
- 클래스 간의 관계에 있어서 유동적인 상황이라면 상속은 강한 연결로 인해 부적합하다.
- 클래스 관계가 유동적이라면 클래스를 느슨하게 연결해주는 하는 위임을 사용해서 구성해야 한다.
- 위임은 말그대로 어떤 구현에 있어서 이를 떠넘기는식을 구현을 한다.
- `Bridge` 패턴 구성에서 `Abstraction` 클래스는 `Implementor` 에게 실제 기능을 위임하고, `RefineAbstraction` 클래스는 `Abstraction` 클래스를 상속해 기능을 구현한다.

## 다양한 계산기 만들기

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_2.png)

### Calculator

```java
public class Calculator {
    private CalculatorImpl impl;

    public Calculator(CalculatorImpl impl) {
        this.impl = impl;
    }

    public double plus(double a, double b) {
        return this.impl.plus(a, b);
    }

    public double minus(double a, double b) {
        return this.impl.minus(a, b);
    }

    public double multiply(double a, double b) {
        return this.impl.multiply(a, b);
    }
}
```  

- `Calculator` 는 실질적인 계산기 역할을 하는 클래스이다.
- `Bridge` 패턴에서 `Abstraction` 역할을 수행한다.
- `기능의 클래스 계층` 에서 최상위 클래스이다.
- 생성자로 `CalculatorImpl` 를 받고, 실제 기능은 `CalculatorImpl` 의 메소드를 사용해 위임 한다.

### AdvancedCalculator

```java
public class AdvancedCalculator extends Calculator {
    public AdvancedCalculator(CalculatorImpl impl) {
        super(impl);
    }

    public double advancedPlus(double a, double b, double c) {
        double result = this.plus(a, b);
        return this.plus(result, c);
    }

    public double advancedMinus(double a, double b, double c) {
        double result = this.minus(a, b);
        return this.minus(result, c);
    }

    public double advancedMultiply(double a, double b, double c) {
        double result = this.multiply(a, b);
        return this.multiply(result, c);
    }
}
```  

- `AdvancedCalculator` 는 `Calculator` 클래스에서 기능을 확장한 클래스이다.
- `Bridge` 패턴에서 `RefineAbstraction` 역할을 수행한다.
- `Calculator` 클래스를 상속해 기능을 확장 하고 있기 때문에, `기능의 클래스 계층` 에 속한다.

### CalculatorImpl

```java
public abstract class CalculatorImpl {
    public abstract double plus(double a, double b);
    public abstract double minus(double a, double b);
    public abstract double multiply(double a, double b);
}
```  

- `CalculatorImpl` 은 계산기의 실질적인 기능이 선언된 추상 클래스이다.
- `Brdige` 패턴에서 `Implementor` 역할을 수행한다.
- `구현의 클래스 계층` 에서 최상위 클래스이다.

### SimpleCalculatorImpl

```java
public class SimpleCalculatorImpl extends CalculatorImpl {
    @Override
    public double plus(double a, double b) {
        return a + b;
    }

    @Override
    public double minus(double a, double b) {
        return a - b;
    }

    @Override
    public double multiply(double a, double b) {
        return a * b;
    }
}
```  

- `SimpleCalculatorImpl` 은 `CalculatorImpl` 의 기능(계산기의 기능)을 실제로 구현하는 클래스이다.
- `Bridge` 패턴에서 `ConcreteImplementor` 역할을 수행한다.
- `CalculatorImpl` 에 선언된 추상 메소드를 실제로 구현하고 있기 때문에, `구현의 클래스 계층` 에 속한다.

### 테스트

```java
public class BridgeTest {
    @Test
    public void SimpleCalculator_plus() {
        // given
        Calculator calculator = new Calculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;

        // when
        double actual = calculator.plus(a, b);

        // then
        assertThat(actual, is(5d));
    }

    @Test
    public void SimpleCalculator_minus() {
        // given
        Calculator calculator = new Calculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;

        // when
        double actual = calculator.minus(a, b);

        // then
        assertThat(actual, is(-1d));
    }

    @Test
    public void SimpleCalculator_multiply() {
        // given
        Calculator calculator = new Calculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;

        // when
        double actual = calculator.multiply(a, b);

        // then
        assertThat(actual, is(6d));
    }

    @Test
    public void AdvancedSimpleCalculator_plus() {
        // given
        Calculator calculator = new AdvancedCalculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;

        // when
        double actual = calculator.plus(a, b);

        // then
        assertThat(actual, is(5d));
    }

    @Test
    public void AdvancedSimpleCalculator_advancedPlus() {
        // given
        AdvancedCalculator calculator = new AdvancedCalculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;
        double c = 4;

        // when
        double actual = calculator.advancedPlus(a, b, c);

        // then
        assertThat(actual, is(9d));
    }

    @Test
    public void AdvancedSimpleCalculator_advancedMinus() {
        // given
        AdvancedCalculator calculator = new AdvancedCalculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;
        double c = 4;

        // when
        double actual = calculator.advancedMinus(a, b, c);

        // then
        assertThat(actual, is(-5d));
    }

    @Test
    public void AdvancedSimpleCalculator_advancedMultiply() {
        // given
        AdvancedCalculator calculator = new AdvancedCalculator(new SimpleCalculatorImpl());
        double a = 2;
        double b = 3;
        double c = 4;

        // when
        double actual = calculator.advancedMultiply(a, b, c);

        // then
        assertThat(actual, is(24d));
    }
}
```  

## 계산기 확장하기
- 구현된 계산기 계층을 바탕으로 기능적인 확장과 구현적인 확장을 해본다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_bridge_3.png)

### MoreAdvancedCalculator

```java
public class MoreAdvancedCalculator extends AdvancedCalculator {
    public MoreAdvancedCalculator(CalculatorImpl impl) {
        super(impl);
    }

    public double moreAdvancedPlus(double a, double b, double c, double d) {
        double result = this.advancedPlus(a, b, c);
        result = this.plus(result, d);

        return result;
    }

    public double moreAdvancedMinus(double a, double b, double c, double d) {
        double result = this.advancedMinus(a, b, c);
        result = this.minus(result, d);

        return result;
    }

    public double moreAdvancedMultiply(double a, double b, double c, double d) {
        double result = this.advancedMultiply(a, b, c);
        result = this.multiply(result, d);

        return result;
    }
}
```  

- `MoreAdvancedCalculator` 는 `AdvancedCalculator` 에서 기능을 다시 확장한 클래스이다.
- `Bridge` 패턴에서 `RefineAbstraction` 역할을 수행한다.
- `기능의 클래스 계층` 에서 `AdvancedCalculator` 클래스를 상속 받아 새로운 기능을 추가했다.

### RoundCalculatorImpl

```java
public class RoundCalculatorImpl extends CalculatorImpl{
    @Override
    public double plus(double a, double b) {
        return Math.round(a + b);
    }

    @Override
    public double minus(double a, double b) {
        return Math.round(a - b);
    }

    @Override
    public double multiply(double a, double b) {
        return Math.round(a * b);
    }
}
```  

- `RoundCalculatorImpl` 은 `CalculatorImpl` 의 기능(계산기의 기능)을 실제로 구현하는 클래스이다.
- `Bridge` 패턴에서 `ConcreteImplementor` 역할을 수행한다.
- `구현의 클래스 계층` 에서 `CalculatorImpl` 에 대한 새로운 구현을 추가 했다.

### 테스트

```java
public class BridgeTest {
    @Test
    public void RoundCalculator_plus() {
        // given
        Calculator calculator = new Calculator(new RoundCalculatorImpl());
        double a = 2.2;
        double b = 3.3;

        // when
        double actual = calculator.plus(a, b);

        // then
        assertThat(actual, is(6d));
    }

    @Test
    public void AdvancedRoundCalculator_plus() {
        // given
        AdvancedCalculator calculator = new AdvancedCalculator(new RoundCalculatorImpl());
        double a = 2.2;
        double b = 3.3;
        double c = 4.4;

        // when
        double actual = calculator.advancedPlus(a, b, c);

        // then
        assertThat(actual, is(10d));
    }

    @Test
    public void MoreAdvancedRoundCalculator_plus() {
        // given
        MoreAdvancedCalculator calculator = new MoreAdvancedCalculator(new RoundCalculatorImpl());
        double a = 2.2;
        double b = 3.3;
        double c = 4.4;
        double d = 5.5;

        // when
        double actual = calculator.moreAdvancedPlus(a, b, c, d);

        // then
        assertThat(actual, is(16d));
    }

    @Test
    public void MoreAdvancedSimpleCalculator_plus() {
        // given
        MoreAdvancedCalculator calculator = new MoreAdvancedCalculator(new SimpleCalculatorImpl());
        double a = 2.2;
        double b = 3.3;
        double c = 4.4;
        double d = 5.5;

        // when
        double actual = calculator.moreAdvancedPlus(a, b, c, d);

        // then
        assertThat(actual, is(15.4));
    }
}
```  

---
## Reference

	