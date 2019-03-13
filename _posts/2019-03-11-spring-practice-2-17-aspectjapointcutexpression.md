--- 
layout: single
classes: wide
title: "[Spring 실습] AspectJ 로 Pointcut 표현"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AspectJ 를 사용해서 다양한 Pointcut 을 표현해 보자.'
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
    - AspectJ
    - within
    - execution
---  

# 목표
- 공통 관심사(Aspect) 는 프로그램이 실행되는 지점, 즉 여러 Joinpoint 에 걸쳐 분포한다.
- Aspect 를 적용하고 싶은 Joinpoint 에 적용하기 위해서는 다양한 위치, 조건을 만족하는 표현언어가 필요하다.

# 방법
- AspectJ 는 다양한 종류의 Joinpoint 를 매치할 수 있는 강력한 표현언어이다.
- 스프링 AOP 가 지원하는 Joinpoint 대상은 IoC 컨터에너 안에 언선된 Bean 들에 국한된다.
- 조금 더 자세한 AspectJ Pointcut 언어는 [AspectJ](www.eclipse.org/aspectj/) 에서 확인할 수 있다.
- 스프링 AOP 에서는 AspectJ Pointcut 언어를 활용해 Pointcut 을 정의하며 런타임에 AspectJ 라이브러리를 이용해 Pointcut 표현식을 해석한다.
- 스프링 AOP 에서 AspectJ Pointcut 표현식을 작성할 경우 스프링 AOP 가 IoC 컨테이너 안에 있는 빈에만 Joinpoint 를 지원한다.
- 위 범위(IoC 컨테이너 외)를 벗어난 Pointcut 표현식을 쓰면 IllegalArgumentsException 예외가 발생한다.

# 예제
- 스프링에 구현된 Pointcut 표현식 패턴은 다음과 같다.
	- 메시지 시그니처 패턴
	- 타입 패턴에 따라 Pointcut 을 작성하는 방법
	- 메서드 인수의 사용법
	
## 메서드 시그니처 패턴
- Pointcut 표현식의 가장 일반적인 모습은 시그니처를 기준으로 여러 메서드를 매치하는 것이다.
- 다음과 같은 표현식은 ArithmeticCalculator 인터페이스에 선언한 메서드를 전부 매치한다.
	- `execution(* com.apress.springrecipes.calculator.ArithmeticCalculator.*(..))`
		- 앞쪽의 와일드카드는 수정자(public, protected, private)와 반환형에 상관없이, 뒤쪽 두 점(..) 은 인수 개수에 상관없이 매치한다는 의미이다.
	- `execution(* ArithmeticCalculator.*(..))`
		- 대상 클래스나 인터페이스가 Aspect 와 같은 패키지에 있으면 패키지명은 생략 가능하다.
- 다음은 ArithmeticCaculator 인터페이스에 선언된 모든 public 메서드를 매치하는 Pointcut 표현식이다.
	- `execution(public * ArithmeticCalculator.*(..))`
- 메서드 반환형을 double 형과 같이 특정하는 것도 가능하다.
	- `execution(public double ArithmeticCalculator.*(..))`
- 메서드 인수 목록에 제약을 두는 것도 가능하다.
	- `execution(public double ArithmeticCalculator.*(double, ..))`
		- 첫 번째 인수가 double 형인 메서드만 메치하며, 두 점(..) 을 통해 두 번째 이후 인수는 몇개라도 상관없다는 Pointcut 표현식이다.
- 메서드 인수형과 개수가 정확히 매치 되도록 하는 것도 가능하다.
	- `execution(public double ArithmeticCalculator.*(double, double))`
	
- AspectJ 에 탑재된 Pointcut 언어는 다양한 Joinpoint 를 매치할 수 있다.
- 매치 하고 싶은 메서드들 사이에 공통 특징(수정자, 반환형, 메서드명 패턴, 인수)이 없는 경우에는 메서드/타입 레벨에 커스텀 Annotation 을 만들어 붙이면 해결 가능하다.

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LoggingRequired {
}
```  

- 로깅이 필요한 메서드에 커스텀 Annotation @LoggingRequired 를 붙이면된다.
	- 클레스 레벨에 붙이게 될경우 포함된 메서드에 모두 적용된다.
	- Annotation 은 상속되지 않으므로 인터페이스나 상위클래스가 아닌, 구현 클래스나 하위 클래스에 붙여야 한다.

```java
@Component("arithmeticCalculator")
@LoggingRequired
public class ArithmeticCalculatorImpl implements ArithmeticCalculator {

    @Override
    public double add(double a, double b) {
    	// ...
    }

    @Override
    public double sub(double a, double b) {
    	// ...
    }

    @Override
    public double mul(double a, double b) {
    	// ...
    }

    @Override
    public double div(double a, double b) {
    	// ...
    }
}
```  

- @LoggingRequired 가 선언된 클레스/메서드를 스캐닝 하기 위해서는 @Pointcut 의 Annotation 안에 표현식을 작성한다.

```java
@Pointcut("annotation(com.apress.springrecipes.calculator.LoggingRequired)")
public void loggingOperation() {
}
```  

## 타입 시그니처 패턴
- 특정한 타입 내부에 모든 Joinpoint 를 매치하는 Pointcut 표현식도 있다.
- 스프링 AOP 에 적용하면 그 타입 안에 구현된 메서드를 실행할 때만 Advice 가 적용 되도록 Pointcut 적용 범위를 좁힐 수 있다.
- 다음 Pointcut 은 com.apress.springredipes.calculator 패키지의 전체 메서드 실행 Joinpoint 를 매치한다.
	- `within(com.apress.springrecipes.calculator.*)`
	- 하위 패키지도 함께 매치하려면 와일드카드 앞에 점 하나를 더 붙인다.
		- `within(com.apress.springrecipes.calculator..*)`
	- 어느 한 클래스 내부에 구현된 메서드 실행 Joinpoint 를 매치하는 Pointcut 은 아래와 같다.
		- `within(com.apress.springrecipes.calculator.ArithmeticCalculatorImpl)`
		- 해당 클래스의 패키지가 Aspect 와 같으면 패키지명은 안 써도 된다.
			- `within(ArithmeticCalculatorImpl)`
	- ArithmeticCalculator 인터페이스를 구현한 모든 클래스의 메서드 실행 Joinpoint 를 매치하려면 뒤에 플러스(+) 를 붙인다.
		- `within(ArithmeticCalculator+)`
	- @LoggingRequired 같은 커스텀 Annotation 은 클래스, 메서드 레벨에 적용 가능하다.
	
	```java
	@Component("arithmeticCalculator")
    @LoggingRequired
    public class ArithmeticCalculatorImpl implements ArithmeticCalculator {
    	// ...
    }
	```  
	- @Pointcut 에 within 키워드를 써서 @LoggingRequired 를 붙인 모든 클래스/메서드의 Joinpoint를 매치하는 Pointcut 은 아래와 같다.
	
		``` java
		@Aspect
        public class CalculatorPointcuts {
            @Pointcut("within(com.apress.springrecipes.calculator.LoggingRequired)")
            public void loggingOperation() {
            }
        }
		```  

## Pointcut 표현식 조합하기
- AspectJ Pointcut 표현식은 `&&(and), ||(or), !(not)` 등의 연산자로 조합할 수 있다.
- ArithmeticCalculator 또는 UnitCalculator 인터페이스를 구현한 클래스의 Joinpoint 와 매치하는 Pointcut 은 아래와 같다.
	- `within(ArithmeticCalculator+) || within(UnitCalculator+)`
- Pointcut 표현식이나 다른 Pointcut 을 가라키는 레퍼런스 모두 이런 연산자로 묶을 수 있다.

```java
@Aspect
public class CalculatorPointcuts {

    @Pointcut("within(ArithmeticCalculator+)")
    public void arithmeticOperation() {
    }

    @Pointcut("within(UnitCalculator+)")
    public void unitOperation() {
    }

    @Pointcut("arithmeticOperation() || unitOperation()")
    public void loggingOperation() {
    }
}
```  

## Pointcut 매개변수 선언하기
- Joinpoint 정보는(Advice 메서드에서 org.aspectj.lang.JoinPoint 형 인수를 통해) 리플렉션으로 액세스 가능하다.
- 이외에도 Declarative Way(선언적인 방법) 으로 Joinpoint 정보를 얻을 수 있다.
- AspectJ 표현식 에서 target(), args() 로 현재 Joinpoint 의 대상 객체 및 인수값을 Pointcut 매개변수로 액세스 할수 있다.
- 액세스된 매개변수는 이름이 똑같은 Advice 메서드의 인수로 전달된다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

    private Log log = LogFactory.getLog(this.getClass());

	// ...
	
    @Before("execution(* *.*(..)) && target(target) && args(a,b)")
    public void logParameter(Object target, double a, double b) {
        log.info("Target class : " + target.getClass().getName());
        log.info("Arguments : " + a + ", " + b);
    }
}
```  

- 독립적인 Pointcut 을 선언해 사용할 경우 Pointcut 메서드의 인수 목록에도 함께 추가해야 한다.

```java
@Aspect
public class CalculatorPointcuts {
	
	// ...
	
    @Pointcut("execution(* *.*(..)) && target(target) && args(a,b)")
    public void parameterPointcut(Object target, double a, double b) {
    }
}
```  

- Parameterized(매개변수화) 한 Pointcut 을 참조하는 모든 Advice 는 같은 이름의 메서드 인수를 선언해서 Pointcut 매개변수를 참조 할 수 있다.

```java
@Aspect
@Component
public class CalculatorLoggingAspect {

	// ...

    @Before("CalculatorPointcuts.parameterPointcut(target, a, b)")
    public void logParameter(Object target, double a, double b) {
        log.info("Target class : " + target.getClass().getName());
        log.info("Arguments : " + a + ", " + b);
    }
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
