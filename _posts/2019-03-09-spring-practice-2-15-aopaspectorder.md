--- 
layout: single
classes: wide
title: "[Spring 실습] AOP @Order 로 Aspect 우선순위 설정하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Aspect 간 우선순위를 @Order 를 사용해서 설정하자'
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
    - A-Order
    - spring-core
    - execution
---  

# 목표
- 같은 Joinpoint 에 Aspect 를 여러 개 적용했을 때에 Aspect 간 우선순위를 설정한다.

# 방법
- Aspect 간 우선순위는 Ordered 인터페이스를 구현하거나 @Order Annotation 을 이용해서 지정 가능하다.

# 예제
- 아래와 같은 2개의 Aspect 가 있다.

```java
@Aspect
@Component
public class CalculatorValidationAspect {

    @Before("execution(* *.*(double, double))")
    public void validateBefore(JoinPoint joinPoint) {
        for (Object arg : joinPoint.getArgs()) {
            validate((Double) arg);
        }
    }

    private void validate(double a) {
        if (a < 0) {
            throw new IllegalArgumentException("Positive numbers only");
        }
    }
}

@Aspect
@Component
public class CalculatorLoggingAspect {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Before("execution(* *.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("The method {}() begins with {}", joinPoint.getSignature().getName() ,Arrays.toString(joinPoint.getArgs()));
    }

}
```  

- 로깅(CalculateLoggingAspect) 와 검증(CalculateValidationAspect) 가 모두 적용되는 Joinpoint 에서 둘 중 하나의 Aspect 를 먼저 실행하기 위해서는 우선순위를 정해야 한다.
- 두 Aspect 모두 Ordered 인터페이스를 구현하거나 @Order Annotation 을 사용하면 된다.
- Order 인터페이스를 구현할 경우 getOrder() 메서드가 반환하는 값이 작을 수록 우선순위가 높게 된다.
- 로깅, 검증 중에서 검증 Aspect 의 우선순위를 높게 했을 경우 아래와 같다.

```java
@Aspect
@Component
public class CalculatorValidationAspect implements Ordered {
	
	// ...
	
	@Override
    public int getOrder() {
        return 0;
    }
}

@Aspect
@Component
public class CalculatorLoggingAspect implements Ordered {
	
	// ...

	@Override
    public int getOrder() {
        return 1;
    }
}
```  

- 같은 우선순위로 @Order Annotation 을 사용하면 아래와 같다.

```java
@Aspect
@Component
@Order(0)
public class CalculatorValidationAspect {
	
	// ...	
}

@Aspect
@Component
@Order(1)
public class CalculatorLoggingAspect {
	
	// ...
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
