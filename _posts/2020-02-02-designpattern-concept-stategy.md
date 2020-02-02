--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Strategy Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '알고리즘을 용이하게 교체할 수 있도록 구성하는 Strategy 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Strategy
use_math : true
---  

## Strategy 패턴이란
- `Strategy` 는 전략을 의미를 가진 것처럼 무언가를 구현하기 위한, 문제를 해결하기 위한 전략(알고리즘)을 구성하는 패턴이다.
- `Strategy` 패턴을 통해 구성된 전략(알고리즘)은 교환 및 교체가 손쉽게 가능하다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_strategy_1.png)

- 패턴의 구성요소
	- `Strategy` : 전략(알고리즘)을 구성하기 위한 메소드를 선언하는 추상 클래스 또는 인터페이스이다.
	- `ConcreteStrategy` : `Strategy` 에 선언된 추상 메소드를 실제로 구현하는 클래스로, 각기 다른 방식으로 구체적인 알고리즘의 구현이 작성된다.
	- `Context` : `Stategy` 의 전략(알고리즘)을 사용해서 만들고자 하는 동작을 구현한다.
- `Strategy` 패턴은 위임(delegation) 이라는 느슨한 연결을 사용해서 구성하기 때문에, 알고리즘을 용이하게 교환할 수 있다는 장점이 있다.
	- 다양한 방식으로 구현되는 알고리즘의 경우 보다 손쉬운 확장이 가능하다.
	- 구현된 알고리즘을 개선해야 하는 경우에도 손쉽게 새로운 알고리즘으로 교환 할 수 있다.
	- 프로그램이 실행 도중 입력 값이나, 상황에 맞춰 알고리즘을 교환하며 사용이 할 수도 있다.
	
## 만기 금액 계산기
- 만기 금액을 계산해주는 계산기를 예시로 만들어 본다. 예금, 적금상품은 단리, 복리 상품으로 나눠지고 특징은 아래와 같다.
	- 단리 : 이자를 계산할 때 원금에 대해서만 일정한 시기에 약정한 이율을 적용하여 계산하는 방법
	- 복리 : 일정기간의 기말마다 이자를 원금에 가산하여 그 합계액을 다음 기간의 원금으로 계산하는 방법
- 단리와 복리의 계산 방식은 아래와 같이 다르기 때문에 이를 각각의 클래스로 구성한다.
	- S는 합계, A는 원금, r은 이율, n은 기간을 뜻한다.
	- 단리 : $S = A(1 + rn)$
	- 복리 : $S = A(1 + r)^n$

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_strategy_2.png)

### Strategy

```java
public interface Strategy {
    double calculateMaturityPrice(int initPrice, int month, double interestRate);
}
```  

- `Strategy` 는 만기 금액 계산 전략을 구성하기 위한 추상 메소드가 선언된 인터페이스이다.
- `Strategy` 패턴에서 `Strategy` 역할을 수행한다.
- `calculateMaturityPrice()` 메소드에 필요한 인자값을 넘겨 주면, 계산 전략에 따라 만기금액을 계산해서 리턴한다.

### SimpleRateStrategy

```java
public class SimpleRateStrategy implements Strategy {

    @Override
    public double calculateMaturityPrice(int initPrice, int month, double interestRate) {
        return initPrice * (1 + (interestRate / month * month));
    }
}
```  

- `SimpleRateStrategy` 는 `Strategy` 를 단리 방식의 계산 전략이 구현된 클래스이다. 
- `Strategy` 패턴에서 `ConcreteStrategy` 역할을 수행한다.

### CompoundRateStrategy

```java
public class CompoundRateStrategy implements Strategy {

    @Override
    public double calculateMaturityPrice(int initPrice, int month, double interestRate) {
        return initPrice * Math.pow(1 + interestRate / month, month);
    }
}
```  

- `CompoundRateStrategy` 는 `Strategy` 를 복리 방식의 계산 전략이 구현된 클래스이다.
- `Strategy` 패턴에서 `ConcreteStrategy` 역할을 수행한다.

### SavingCalculate

```java
public class SavingCalculator {
    private int initPrice;
    private int month;
    private double interestRate;
    private Strategy strategy;

    public SavingCalculator(int initPrice, int month, double interestRate, Strategy strategy) {
        this.initPrice = initPrice;
        this.month = month;
        this.interestRate = interestRate;
        this.strategy = strategy;
    }

    public double calculatePrice() {
        return this.strategy.calculateMaturityPrice(this.initPrice, this.month, this.interestRate);
    }
}
```  

- `SavingCalculate` 는 만기 금액 계산기를 나태내는 클래스이다.
- `Strategy` 패턴에서 `Context` 역할을 수행한다.
- 초기 금액, 기간, 이율, 전략을 생성자에서 인자값을 받아 계산기 객체를 구성한다.
- `calculatePrice()` 메소드에서 구성된 객체를 바탕으로 만기금액을 `Strategy` 에 위임해 계산한다.

### 테스트

```java
public class StrategyTest {
    @Test
    public void SimpleRateStrategy() {
        // given
        int initPrice = 10000000;
        int month = 12;
        double interestRate = 0.05;
        SavingCalculator savingCalculator = new SavingCalculator(initPrice, month, interestRate, new SimpleRateStrategy());

        // when
        double actual = savingCalculator.calculatePrice();

        // then
        assertThat(actual, is(10500000d));
    }

    @Test
    public void CompoundRateStrategy() {
        // given;
        int initPrice = 10000000;
        int month = 12;
        double interestRate = 0.05;
        SavingCalculator savingCalculator = new SavingCalculator(initPrice, month, interestRate, new CompoundRateStrategy());

        // when
        double actual = savingCalculator.calculatePrice();
        actual = Math.round(actual);

        // then
        assertThat(actual, is(10511619d));
    }
}
```  

---
## Reference

	