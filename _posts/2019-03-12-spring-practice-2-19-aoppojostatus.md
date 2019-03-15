--- 
layout: single
classes: wide
title: "[Spring 실습] AOP 로 POJO 인스턴스 변수 추가하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AOP 를 이용해 POJO 인스턴스 변수를 추가하자'
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
- 기존 객체에 새로운 인스턴스 변수를 추가해서 호출 횟수, 최종 수정 일자 등 사용 내역을 파악한다.
- 모든 객체가 동일한 Base Class 를 상속하는 방법으로는 불가능하다.
- 레이어 구조가 다른 여러 클래스에 인스턴스 변수를 추가 해보자.

# 방법
- 인스턴스 변수가 위치한 구현 클래스의 인터페이스를 기존 객체에 가져온다.
- 특정 조건에 따라 인스턴스 변수의 값을 변경하는(getter, setter) Advice 를 작성한다.


# 예제
- Calculator 객체의 호출 횟수를 기록한다.
- 기존 Calculator 클래스에는 호출 횟수 변수가 존재하지 않기 때문에 스프링 AOP Introduction 을 사용한다.
- 카운터 인터페이스

```java
public interface Counter {
    void increase();
    int getCount();
}
```  

- Counter 인터페이스의 구현 클래스

```java
public class CounterImpl implements Counter {

    private int count;

    public void increase() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```  

- CounterImpl 의 인스턴스 변수를 Calculator 객체에서 사용하기 위해 타입 매치 표현식을 이용해 Introduction 을 작성한다.

```java
@Aspect
@Component
public class CalculatorIntroduction {

	// ...
	
    @DeclareParents(
            value = "com.apress.springrecipes.calculator.*CalculatorImpl",
            defaultImpl = CounterImpl.class)
    public Counter counter;

}
```

- 계산기 메서드를 한번씩 호출할 때마다 counter 값을 하나씩 증가 시키기 위해서는 After Advice 적용이 필요하다.
- Counter 인터페이스를 구현한 객체는 프록시가 유일하므로 반드시 target 이아닌, this 객체를 가져와 사용해야 한다.

```java
@Aspect
@Component
public class CalculatorIntroduction {

	// ...

    @DeclareParents(
            value = "com.apress.springrecipes.calculator.*CalculatorImpl",
            defaultImpl = CounterImpl.class)
    public Counter counter;

    @After("execution(* com.apress.springrecipes.calculator.*Calculator.*(..))"
            + " && this(counter)")
    public void increaseCount(Counter counter) {
        counter.increase();
    }
}

```  

- Main 클래스

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context = new GenericXmlApplicationContext("appContext.xml");

        ArithmeticCalculator arithmeticCalculator = (ArithmeticCalculator) context.getBean("arithmeticCalculator");
        arithmeticCalculator.add(1, 2);
        arithmeticCalculator.sub(4, 3);
        arithmeticCalculator.mul(2, 3);
        arithmeticCalculator.div(4, 2);

        UnitCalculator unitCalculator = (UnitCalculator) context.getBean("unitCalculator");
        unitCalculator.kilogramToPound(10);
        unitCalculator.kilometerToMile(5);

        MaxCalculator maxCalculator = (MaxCalculator) arithmeticCalculator;
        maxCalculator.max(1, 2);

        MinCalculator minCalculator = (MinCalculator) arithmeticCalculator;
        minCalculator.min(1, 2);

        Counter arithmeticCounter = (Counter) arithmeticCalculator;
        System.out.println("arithmeticCounter : " + arithmeticCounter.getCount());

        Counter unitCounter = (Counter) unitCalculator; 
        System.out.println("unitCounter : " + unitCounter.getCount());
    }
}
```  

- arithmeticCounter, uniCounter 출력결과

```
arithmeticCounter : 4
unitCounter : 2
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
