--- 
layout: single
classes: wide
title: "[Spring 실습] Aspect Pointcut 재사용 하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AOP 에서 Pointcut 을 재사용 해보자'
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
    - A-Pointcut
    - spring-core
    - execution
---  

# 목표
- 같은 Pointcut 표현식을 여러 번 사용해야 할 경우, 재사용을 해보자

# 방법
- @Pointcut Annotation 을 이용하면 Pointcut 만 따로 정의해 여러 Advice 에서 재사용 할 수 있다.

# 예제
- Aspect 에서 Pointcut 은 @Pointcut 을 붙인 단순 메서드로 선언할 수 있다.
- Pointcut 과 애플리케이션 로직이 혼재 되어 있는 것은 좋지 않으므로, 메서드의 바디는 보통 비워두고 Pointcut Visibility(가시성) 은 (public, protected, private 등) 메서드의 수정자로 조정 한다.
- 위와 같은 방법으로 선언한 Pointcut 은 다른 Advice 가 메서드명으로 참조하여 사용한다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Log log = LogFactory.getLog(this.getClass());

    @Pointcut("execution(* *.*(..))")
    private void loggingOperation() {
    }

    @Before("CalculatorLoggingAspect.loggingOperation()")
    public void logBefore(JoinPoint joinPoint) {
    	// ...
    }

    @After("CalculatorLoggingAspect.loggingOperation()")
    public void logAfter(JoinPoint joinPoint) {
    	// ...
    }

    @AfterReturning(
            pointcut = "CalculatorLoggingAspect.loggingOperation()",
            returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
    	// ...
    }

    @AfterThrowing(
            pointcut = "CalculatorLoggingAspect.loggingOperation()",
            throwing = "e")
    public void logAfterThrowing(JoinPoint joinPoint, IllegalArgumentException e) {
    	// ...
    }

    @Around("CalculatorLoggingAspect.loggingOperation()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
    	// ...
    }

}
```  

- 여러 Aspect 가 Pointcut 을 공유하는 경우에는 공통 클래스에서 Pointcut 을 모아두는 방식이 좋다.
- 위와 같을 경우 Pointcut 메서드의 수정자는 public 으로 선언하는 것이 좋다.

```java
@Aspect
public class CalculatorPointcuts {

    @Pointcut("execution(* *.*(..))")
    public void loggingOperation() {
    }

}
```  

- Pointcut 을 참조할 때는 클래스명도 함께 기입힌다. (CalculatorPointcuts.loggingOperation())
- 참조할 Pointcut 이 헌재 Aspect 와는 다른 패키지에 있으면 패키지까지 기입한다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Log log = LogFactory.getLog(this.getClass());

    @Before("CalculatorPointcuts.loggingOperation()")
    public void logBefore(JoinPoint joinPoint) {
    	// ...
    }

    @After("CalculatorPointcuts.loggingOperation()")
    public void logAfter(JoinPoint joinPoint) {
    	// ...
    }

    @AfterReturning(
            pointcut = "CalculatorPointcuts.loggingOperation()",
            returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
    	// ...
    }

    @AfterThrowing(
            pointcut = "CalculatorPointcuts.loggingOperation()",
            throwing = "e")
    public void logAfterThrowing(JoinPoint joinPoint, IllegalArgumentException e) {
    	// ...
    }

    @Around("CalculatorPointcuts.loggingOperation()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
    	// ...
    }
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
