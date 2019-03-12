--- 
layout: single
classes: wide
title: "[Spring 실습] AOP JoinPoint 정보 가져오기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AOP JoinPoint 의 정보를 가져오자'
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
    - JoinPoint 
---  

# 목표
- AOP 에서 Advice 는 여러 Joinpoint, 프로그램 실행 지점 곳곳에 적용된다.
- Advice 가 정확하게 동작하려면 Joinpoint 에 관한 세부 정보가 필요한 경우가 많다.

# 방법
- Advice 메서드의 Sigature(서명부)에 org.aspectj.lang.JoinPoint 형 인수를 선언하면 여기서 Joinpoint 정보를 얻을 수 있다.

# 예제
- logJoinPoint Advice 에서 Joinpoint 정보를 액세스 한다 하자.
- 필요한 정보는 Joinpoint 유형(스프링 AOP 의 메서드 실행만 해당), 메서드 시그니쳐(선언 타입 및 메서드명), 인수값, 대상 객체와 프록시 객체이다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Before("execution(* *.*(..))")
    public void logJoinPoint(JoinPoint joinPoint) {
        log.info("Join point kind : {}", joinPoint.getKind());
        log.info("Signature declaring type : {}", joinPoint.getSignature().getDeclaringTypeName());
        log.info("Signature name : {}", joinPoint.getSignature().getName());
        log.info("Arguments : {}", Arrays.toString(joinPoint.getArgs()));
        log.info("Target class : {}", joinPoint.getTarget().getClass().getName());
        log.info("This class : {}", joinPoint.getThis().getClass().getName());
    }
}
```  

- 프록시로 감싼 원본 빈은 대상 객체(target object) 라고 하며 프록시 객체는 this 로 참조 한다.
- 대상 객체와 프록시 객체는 각각 조인인트에서 getTarget(), getThis() 메서드로 가져올 수 있다.
- 실행결과를 보면 두 객체가 다른다는 것을 알 수 있다.

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context =
                new AnnotationConfigApplicationContext(CalculatorConfiguration.class);

        ArithmeticCalculator arithmeticCalculator =
                context.getBean("arithmeticCalculator", ArithmeticCalculator.class);
        arithmeticCalculator.add(1, 2);

        UnitCalculator unitCalculator = context.getBean("unitCalculator", UnitCalculator.class);
        unitCalculator.kilogramToPound(10);
    }
}
```  

```
// arithmeticCalculator.add(1, 2) 호출 시
Join point kind : method-execution
Signature declaring type : com.apress.springrecipes.calculator.ArithmeticCalculator
Signature name : add
Arguments : [1.0, 2.0]
Target class : com.apress.springrecipes.calculator.ArithmeticCalculatorImpl
This class : com.sun.proxy.$Proxy18

// unitCalculator.kilogramToPound(10) 호출 시
Join point kind : method-execution
Signature declaring type : com.apress.springrecipes.calculator.UnitCalculator
Signature name : kilogramToPound
Arguments : [10.0]
Target class : com.apress.springrecipes.calculator.UnitCalculatorImpl
This class : com.sun.proxy.$Proxy19
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
