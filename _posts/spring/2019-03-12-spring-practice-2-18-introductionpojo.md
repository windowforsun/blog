--- 
layout: single
classes: wide
title: "[Spring 실습] AOP Introduction 으로 POJO 에 공통 기능 추가하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AOP Introduction 을 사용해서 공통 로직을 POJO 에 추가해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Annotation
    - AOP
    - spring-core
    - Introduction
    - A-DeclareParents
---  

# 목표
- 어떤 공통 로직을 공유하는 클래스가 여러 개 있을 경우, OOP 에서는 같은 Base Class(베이스 클래스) 를 상속하거나 같은 인터페이스를 구현하는 경태로 애플리케이션을 개발한다.
- AOP 의 관점에서는 충분히 모듈화 가능한 공통 관심사이지만, 자바와 같은 언어에서는 구조상 클래스를 한 개만 상속할 수 있기 때문에 동시에 여러 구현 클래스로부터 기능을 상속받아 사용하는 것이 불가능하다.

# 방법
- Introduction(끌어들임) 은 AOP Advice 의 특별한 타입이다.
	- 객체가 어떤 인터페이스의 구현 클래스를 공급받아 동적으로 인터페이스를 구현할 수 있다.
	- 객체가 런타임에 구현 클래스를 상속하는 것과 비슷한 동작을 보인다.
	- 여러 구현 클래스를 지닌 여러 인터페이스를 동시에 Introduce 할 수 있어, 사실상 다중 상속도 가능하다.

# 예제
- MaxCalculator, MinCalculator 두 인터페이스에 각각 max(), min() 메서드가 있다.

```java
public interface MaxCalculator {
    double max(double a, double b);
}

public interface MinCalculator {
    double min(double a, double b);
}
```  

- MaxCalculatorImpl, MinCalculatorImpl 은 위의 두 인터페이스의 구현 클래스로 출력문을 통해 출력 시점을파악한다.

```java
public class MaxCalculatorImpl implements MaxCalculator {
    @Override
    public double max(double a, double b) {
        double result = (a >= b) ? a : b;
        System.out.println("max(" + a + ", " + b + ") = " + result);
        return result;
    }
}

public class MinCalculatorImpl implements MinCalculator {
    @Override
    public double min(double a, double b) {
        double result = (a <= b) ? a : b;
        System.out.println("min(" + a + ", " + b + ") = " + result);
        return result;
    }
}
```  

- ArithmeticCalculatorImpl 클래스를 통해 max(), min() 메서드를 둘다 사용해야 할 경우라면 ?
	- MaxCalculatorImpl, MinCalculatorImpl 두 구현클래스를 다중상속하는 것이 편리하지만 자바에서는 불가능하다.
	- 구현 코드를 복사하거나, 실제 구현 클래스에게 처리를 맡겨야 한다.
	- 혹은 한쪽은 클래스로 상속, 다른 한쪽은 인터페이스로 구현 할 수도 있다.
- 위와 같은 경우에 Introduction 을 통해 ArithmeticCalculatorImpl 에서 MaxCalculator 및 MinCalculator 인터페이스를 구현한 MaxCalculatorImpl, MinCalculatorImpl 을 동적으로 구현한 것처럼 사용할 수 있다.
	- MaxCalculatorImpl, MinCalculatorImpl 을 다중 상속하는 것처럼 사용 가능하다.
- Introduction 은 ArithmeticCalculatorImpl 클래스를 수정해 새 메서드를 추가할 필요가 없다.
	- 기존 소스코드를 건드리지 않고 메서드를(공통기능) 사용할 수 있다.

> ### 스프링 AOP Introduction 동작 방식  
> - 동적 프록시를 사용하여 Introduction 의 동작 방식을 구현한다.
> - Introduction 은 동적 프록시에 인터페이스(MaxCalculator) 를 추가한다.  
> - 인터페이스에 선언된 메서드를 프록시 객체에서 호출하면 프록시는 백엔드 구현클래스(MaxCalculatorImpl)에 처리를 위임한다.  

- Introduction 연식 Advice 처럼 Aspect 안에서 필드에 @DeclareParents 를 붙여 선언한다.
- Aspect 를 새로 만들거나 용도가 비슷한 기존 Aspect 를 재사용 할 수도 있다.

```java
@Aspect
@Component
public class CalculatorIntroduction {

    @DeclareParents(
            value = "com.apress.springrecipes.calculator.ArithmeticCalculatorImpl",
            defaultImpl = MaxCalculatorImpl.class)
    public MaxCalculator maxCalculator;

    @DeclareParents(
            value = "com.apress.springrecipes.calculator.ArithmeticCalculatorImpl",
            defaultImpl = MinCalculatorImpl.class)
    public MinCalculator minCalculator;
}
```  

- Introduction 대상 클래스는 @DeclareParents 의 value 속성으로 지정한다.
- Annotation 이 선언된 필드의 타입에 따라 인터페이스가 결정된다.
- 인터페이스에서 사용할 구현 클래스는 defaultImpl 속성에 명시한다.
- 선언한 두 Introduction 을 사용해 ArithmeticCalculatorImpl 클래스에 Introduction 의 기능들을 추가할 수 있다.
- ArithmeticCalculatorImpl 에서 MaxCalculator, MinCalculator 처럼 해당 인터페이스로 캐스팅 후 max(), min() 계산을 수행한다.

```java
public class Main {

    public static void main(String[] args) {
        ApplicationContext context = new GenericXmlApplicationContext("appContext.xml");

        ArithmeticCalculator arithmeticCalculator = (ArithmeticCalculator) context.getBean("arithmeticCalculator");
     
        MaxCalculator maxCalculator = (MaxCalculator) arithmeticCalculator;
        maxCalculator.max(1, 2);

        MinCalculator minCalculator = (MinCalculator) arithmeticCalculator;
        minCalculator.min(1, 2);
    }
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
