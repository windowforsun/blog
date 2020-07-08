--- 
layout: single
classes: wide
title: "[Spring 실습] Annotation 으로 AOP 프로그래밍 하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Annotation 을 사용해서 AOP 프로그래밍을 해보자'
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
---  

# 목표
- 스프링에서 Annotation 을 이용해서 AOP(Aspect Oriented Programming) 을 해보자.

# 방법
- Aspect(에스펙트) 를 정의하려면 일단 자바 클래스에 @Aspect 를 붙이고 메서드별로 적절한 Annotation 을 붙여 Advice(어드바이스) 로 만든다.
- Advice Annotation 은 @Before, @After, @AfterReturning, @AfterThrowing, @Around 5개 중 하나를 사용할 수 있다.
- IoC 컨테이너에서 Aspect 기능을 활성화 하려면 설정 클래스 중 하나에 @EnableAspectJAutoProxy 를 붙인다.
- 기본적으로 스프링은 인터페이스 기반의 JDK Dynamic Proxy(동적 프록시)를 생성하여 AOP 를 적용한다.
- 인터페이스를 사용할 수 없거나 애플리케이션 설계상 사용하지 않을 경우엔 CGLIB(씨즐립) 으로 프록시를 만들 수 있다.
- @EnableAspectJAutoProxy 에서 proxyTargetClass 속성을 true 로 설정하면 동적 프록시 대신 CGLIB 를 사용한다.


# 예제
- 스프링에서는 AspectJ 와 동일한 Annotation 으로 Annotation 기반의 AOP 를 구현한다.
- Pointcut 을 파싱, 매칭하는 AspectJ 라이브러리를 그대로 빌려온 것이다.
- AOP 런타임 자체는 순수 스프링 AOP 이기 때문에 AspectJ 컴파일러나 Weaver(위버) 와는 아무런 의존 관계가 없다.
- 두 계산기 인터페이스가 있다.

```java
public interface ArithmeticCalculator {

    public double add(double a, double b);

    public double sub(double a, double b);

    public double mul(double a, double b);

    public double div(double a, double b);
}
```  

```java
public interface UnitCalculator {

    public double kilogramToPound(double kilogram);

    public double kilometerToMile(double kilometer);
}
```  

- 두 인터페이스를 구현한 POJO 클래스를 하나씩 작성하고 어느 메서드가 실행 됐는지 쉽게 알수 있도록 출력문을 사용한다.

```java
@Component("arithmeticCalculator")
public class ArithmeticCalculatorImpl implements ArithmeticCalculator {

    @Override
    public double add(double a, double b) {
        double result = a + b;
        System.out.println(a + " + " + b + " = " + result);
        return result;
    }

    @Override
    public double sub(double a, double b) {
        double result = a - b;
        System.out.println(a + " - " + b + " = " + result);
        return result;
    }

    @Override
    public double mul(double a, double b) {
        double result = a * b;
        System.out.println(a + " * " + b + " = " + result);
        return result;
    }

    @Override
    public double div(double a, double b) {
        if (b == 0) {
            throw new IllegalArgumentException("Division by zero");
        }
        double result = a / b;
        System.out.println(a + " / " + b + " = " + result);
        return result;
    }
}
```  

```java
@Component("unitCalculator")
public class UnitCalculatorImpl implements UnitCalculator {

    @Override
    public double kilogramToPound(double kilogram) {
        double pound = kilogram * 2.2;
        System.out.println(kilogram + " kilogram = " + pound + " pound");
        return pound;
    }

    @Override
    public double kilometerToMile(double kilometer) {
        double mile = kilometer * 0.62;
        System.out.println(kilometer + " kilometer = " + mile + " mile");
        return mile;
    }
}
```  

- 각 구현체는 @Component 를 붙여 빈 인스턴스를 생성한다.

## Aspect, Advice, Pointcut 선언하기
- Aspect 는 여러 타입과 객체에 공통 관심사(Logging, Transaction)를 모듈화한 자바 클래스이다.
	- @Aspect 를 붙여 표시한다.
	- AOP 에서 말하는 Aspect 란 어디에서(Pointcut) 무엇을 할것인지(Advice)를 합쳐 놓은 개념이다.
- Advice 는 @Advice 를 붙인 단순 자바 메서드로, AspectJ 는 @Before, @After, @AfterReturn, @AfterThrowing, @Around 다섯 개 Advice Annotation 을 지원한다.
- Pointcut 은 Advice 에 적용할 타입 및 객체를 찾는 표현식이다.

## @Before Advice
- Before Advice 는 특정 프로그램 실행 지점 이전의 공통 관심사를처리하는 메서드이다.
- @Before 를 붙이고 Pointcut 표현식을 Annotation 값으로 지정한다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Before("execution(* ArithmeticCalculator.add(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("The method {}() begins with {} ", joinPoint.getSignature().getName(), Arrays.toString(joinPoint.getArgs()));
    }

}
```  

- 위 Pointcut 표현식은 ArithmeticCalculator 인터페이스의 add()  메서드 실행을 나타낸다.
- 앞부분의 와일드카드(*)는 모든 수정자(public, protected, private), 모든 반환형을 매치함을 의미한다.
- 인수 목록 부분에 쓴 두 점(..)은 인수 개수는 몇개라도 포함된다는 의미이다.
- Aspect 로직(메시지를 콘솔에 출력)을 실행하려면 다음과 같이 logback.xml 파일을 설정한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>%d [%15.15t] %-5p %30.30c - %m%n</Pattern>
        </layout>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>

</configuration>
```  

- @Aspect 만 붙여서는 스프링이 classpath 에서 자동 감지하지 않기 때문에 해당 POJO 마다 개별적으로 @Component를 붙여야 한다.
- 자바 설정 클래스

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class CalculatorConfiguration {
}
```  

- Main 클래스

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context =
                new AnnotationConfigApplicationContext(CalculatorConfiguration.class);

        ArithmeticCalculator arithmeticCalculator =
                context.getBean("arithmeticCalculator", ArithmeticCalculator.class);
        arithmeticCalculator.add(1, 2);
        arithmeticCalculator.sub(4, 3);
        arithmeticCalculator.mul(2, 3);
        arithmeticCalculator.div(4, 2);

        UnitCalculator unitCalculator = context.getBean("unitCalculator", UnitCalculator.class);
        unitCalculator.kilogramToPound(10);
        unitCalculator.kilometerToMile(5);
    }
}
```  

- Pointcut 으로 매치한 실행 지점을 Joinpoint 라고 한다.
- Pointcut 은 여러 Joinpoint 를 매치하기 위해 지정한 표현식이고 이렇게 매치된 Joinpoint 에서 해야 할 일이 바로 Advice 이다.
- Advice 가 현재 Joinpoint 의 세부적인 내용에 액세스 하려면 JoinPoint 형 인수를 Advice 메서드에 선언해야 한다.
	- 메서드명, 인수 값 등 자세한 Jointpoint 정보를 조회할 수 있다.
- 클래스명, 메서드명에 와일드카드를 써서 모든 메서드에 예외 없이 Pointcut 을 적용한 코드이다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Before("execution(* *.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("The method {}() begins with {} ", joinPoint.getSignature().getName(), Arrays.toString(joinPoint.getArgs()));
    }

}
```  

## @After Advice
- After Advice 는 Joinpoint 가 끝나면 실행되는 메서드이다.
- @After 를 붙여 사용한다.
- Joinpoint 가 정상 실행되든, 도중에 예외가 발생하든 산관없이 실행된다.
- 아래는 계산기 메서드가 끝날 때마다 로그를 남기는 After Advice 이다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {
	
    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...
	
    @After("execution(* *.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        log.info("The method {}() ends", joinPoint.getSignature().getName());
    }

}
```  

## @AfterReturning Advice
- After Advice 는 Joinpoint 실행의 성공 여부와 상관없이 작동한다.
- logAfter aptjemdptj Jointpoint 가 값일 반환할 경우에믄 로깅하고 싶다면, 다음과 같이 After Returning Advice 로 대체하면 된다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...

    @AfterReturning("execution(* *.*(..))")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("The method {}() ends with {}", joinPoint.getSignature().getName(), result);
    }
}
```  

- After Returning Advice 로 Joinpoint 가 반환한 결과값을 가져오려면 @AfterReturning 의 returning 속성으로 지정한 변수명을 Advice 메서드의 인수로 지정한다.
- 스프링 AOP 는 런타임에 조인 포인트의 반환값을 이 인수에 넣어 전달한다.
- Pointcut 표현식은 pointcut 속성으로 따로 지정해야 한다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...

    @AfterReturning(
            pointcut = "execution(* *.*(..))",
            returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("The method {}() ends with {}", joinPoint.getSignature().getName(), result);
    }
}
```  

## @AfterThrowing Advice
- After Throwing Advice 는 Jointpoint 실행 도중 예외가 발생했을 경우에만 실행 된다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...

    @AfterThrowing("execution(* *.*(..))")
    public void logAfterThrowing(JoinPoint joinPoint, IllegalArgumentException e) {
        log.error("Illegal argument {} in {}()", Arrays.toString(joinPoint.getArgs()), joinPoint.getSignature().getName());
    }

}
```  

- 발생한 예외는 @AfterThrowing 의 throwing 속성에 담아 전달할 수 있다.
- Throwable 은 모둔 에러/예외 클래스의 상위 타입이므로 아래와 같이 Advice 를 적용하면 Joinpoint 에서 발생한 모든 에러/예외에 사용할 수 있다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...

    @AfterThrowing(
            pointcut = "execution(* *.*(..))",
            throwing = "e")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable e) {
        log.error("Illegal argument {} in {}()", Arrays.toString(joinPoint.getArgs()), joinPoint.getSignature().getName());
    }

}
```  

- 특정한 예외만 관심이 있다면 그 타입을 인수에 선언한다.
- 선언된 타입과 호횐되는(해당 타입 및 그 하위 타입) 예외가 발생한 경우에만 Advice 가 실행된다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

	// ...

    @AfterThrowing(
            pointcut = "execution(* *.*(..))",
            throwing = "e")
    public void logAfterThrowing(JoinPoint joinPoint, IllegalArgumentException e) {
        log.error("Illegal argument {} in {}()", Arrays.toString(joinPoint.getArgs()), joinPoint.getSignature().getName());
    }

}
```  

## @Around Advice
- Around Advice 는 Joinpoint 의 모든 제어가 가능하기 때문에 앞서 살펴본 Advice 모두 Around Advice 로 조합할 수 있다.
- Joinpoint 를 언제 실행할지, 실행 자체를 할지 말지, 계속 실행할지 여부까지도 제어 할 수 있다.
- Before, After Returning, After Throwing Advice 를 Around Advice 로 조합한 코드이다.
- Around Advice 의 Joinpoint 인수형은 ProceedingJoinPoint 로 고정되어 있다.
- JoinPoint 하위 인터페이스인 ProceedJoinPoint 를 이용하면 원조 Joinpoint 를 언제 진행할지 그 시점을 제어할 수 있다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Around("execution(* *.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {

        log.info("The method {}() begins with {}", joinPoint.getSignature().getName(), Arrays.toString(joinPoint.getArgs()));

        try {
            Object result = joinPoint.proceed();
            log.info("The method {}() ends with ", joinPoint.getSignature().getName(), result);
            return result;
        } catch (IllegalArgumentException e) {
            log.error("Illegal argument {} in {}()", Arrays.toString(joinPoint.getArgs()) , joinPoint.getSignature().getName());
            throw e;
        }
    }

}
```  

- Around 는 매우 강력하고 유연한 Advice 라서 원본 인숫값을 바꾸거나 최종 반환 값을 변경하는 일도 가능하다.
	- 원본 Joinpoint 를 진행하는 호출을 잊어버리기 쉽기 때문에 사용에 주의가 필요하다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
