--- 
layout: single
classes: wide
title: "[Spring 실습] AspectJ Aspect 를 로드 타임 Weaving 하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'AspectJ Aspect 를 로드 타임 Weaving 해서 다양하게 사용해 보자'
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
    - Weaving
    - call
    - A-EnableLoadTimeWeaving
---  

# 목표
- 스프링 AOP 프레임워크는 제한된 타입의 AspectJ Pointcut 만 지원하며 IoC 컨테이너에 서언한 빈에 한하여 Aspect 를 적용할 수 있다.
- Pointcut 타입을 추가하거나, IoC 컨테이너 외부 객체에 Aspect 를 적용하려면 스프링 애플리케이션에서 AspectJ 프레임워크를 직업 사용해야 한다.

# 방법
- Weaving(엮어넣기) 는 Aspect 를 대상 객체에 적용하는 과정이다.
- 스프링 AOP 는 런타임에 동적 프록시를 활용해 Weaving 을 한다.
- AspectJ 프레임워크는 Compile-time(컴파일 시점) Weaving, Load-time(로드 시점) Weaving 을 모두 지원한다.
	- AspectJ Compile-time Weaving
		- Compile-time Weaving 은 ajc 라는 전용 컴파일러가 담당한다.
		- AspectJ 를 자바 소스 파일에 엮고 Weaving 된 바이너리 클래스 파일을 결과물로 만든다.
		- 컴파일된 클래스 파일이나 JAR 파일 안에도 Aspect 를 추가할 수 있는데 이를 Post-compile-time Weaving 이라고 한다.
		- Compile-time, Post-compile-time Weaving 모두 클래스를 IoC 컨테이너레 선언하기 이전에 수행할 수 있으며, 스프링은 Weaving 과정에 전혀 관여하지 않는다.
	- AspectJ Load-time Weaving(LTW)
		- JVM 이 클래스 로더를 이용해 대상 클래스를 로드하는 시점에 일어난다.
		- 바이트코드에 코드를 넣어 클래스를 Weaving 하려면 특수한 클래스로더가 필요하다.
		- AspectJ, 스프링 모두 클래스 로더에 Weaving 기능을 부여한 Load-time Weaver 를 제공한다.
		- Load-time Weaving 은 간단한 설정으로 바로 사용할 수 있다.
- 보다 자세한 내용은 [AspectJ](https://www.eclipse.org/aspectj/) 에서 확인할 수 있다.

# 예제
- 스프링 애플리케이션에서 Load-time Weaving 이 어떻게 처리되는지 복소수 계산기 예제로 설명한다.
- 복소수를 나타내는 Complex 클래스를 작성한다.

```java
public class Complex {

    private int real;
    private int imaginary;

    public Complex(int real, int imaginary) {
        this.real = real;
        this.imaginary = imaginary;
    }
    
    // getter, setter
    
    public String toString() {
        return "(" + real + " + " + imaginary + "i)";
    }
}
```  

- 복소수를 (a+bi) 형식의 문자열로 변환해서 표시하도록 toString() 을 오버라이드 하였다.
- 복소수 계산기 인터페이스이다.

```java
public interface ComplexCalculator {
    public Complex add(Complex a, Complex b);
    public Complex sub(Complex a, Complex b);
}
```  

- 복소수 계산기 인터페이스 ComplexCalculator 의 구현 클래스 이다.

```java
@Component("complexCalculator")
public class ComplexCalculatorImpl implements ComplexCalculator {

    @Override
    public Complex add(Complex a, Complex b) {
        Complex result = new Complex(a.getReal() + b.getReal(), a.getImaginary() + b.getImaginary());
        System.out.println(a + " + " + b + " = " + result);
        return result;
    }

    @Override
    public Complex sub(Complex a, Complex b) {
        Complex result = new Complex(a.getReal() - b.getReal(), a.getImaginary() - b.getImaginary());
        System.out.println(a + " - " + b + " = " + result);
        return result;
    }
}
```  

- 구현된 코드 테스트를 위한 Main 클래스

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context = new AnnotationConfigApplicationContext(CalculatorConfiguration.class);

        ComplexCalculator complexCalculator = context.getBean("complexCalculator", ComplexCalculator.class);

        complexCalculator.add(new Complex(1, 2), new Complex(2, 3));
        complexCalculator.sub(new Complex(5, 8), new Complex(2, 3));
    }
}
```  

- 성능적 측면을 고려해서 복소수 객체를 캐시하는(공통기능) 기능을 추가하기 위해 Aspect 로 모듈화 한다.

```java
@Aspect
public class ComplexCachingAspect {

    private final Map<String, Complex> cache = new ConcurrentHashMap<>();

    @Around("call(public Complex.new(int, int)) && args(a,b)")
    public Object cacheAround(ProceedingJoinPoint joinPoint, int a, int b)
            throws Throwable {

        String key = a + "," + b;
        Complex complex = cache.get(key);

        if (complex == null) {
            System.out.println("Cache MISS for (" + key + ")");
            complex = (Complex) joinPoint.proceed();
            cache.put(key, complex);
        } else {
            System.out.println("Cache HIT for (" + key + ")");
        }

        return complex;
    }
}
```  

- Aspect 코드에서 실수화 허수를 조합한 키를 복수소 객체 맵에 캐시한다.
- 생성자를 호출해서 복소수 객체를 생성할 때 이미 캐시된 값이 있는지 확인한다.
- AspectJ Pointcut 표현 call 안에 Complex(int, int) 로 생성자를 호출하는 Joinpoint 에서 실행한다.
- Call Pointcut 은 스프링 AOP 가 지원하지 않으므로 스프링에서 이 Annotation 을 스캐닝 할때 지원하지 않는 
Pointcut 호출을 의미하는 `unsupported pointcut primitive call.` 에러가 발생한다.
- 스프링 AOP 에서 지원되지 않는 Pointcut 을 쓴 Aspect 를 적용하려면 AspectJ 프레임워크를 직접 사용해야 한다.
- AspectJ 프레임워크는 classpath 루트의 META-INF 디렉토리에 있는 aop.xml 파일에 설정한다.

```xml
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "http://www.eclipse.org/aspectj/dtd/aspectj.dtd">

<aspectj>
    <weaver>
        <include within="com.apress.springrecipes.calculator.*"/>
    </weaver>

    <aspects>
        <aspect name="com.apress.springrecipes.calculator.ComplexCachingAspect"/>
    </aspects>
</aspectj>
```  

- AspectJ 설정 파일에는 Aspect 를 Weaving 할 대상 클래스를 지정한다.
- 위 설정은 ComplexCacheAspect Aspect 를 com.apress.springrecipes.calculator 패키지의 모든 클래스 안으로 
Weaving 하도록 설정 했다.
- 로드 타임 시점 Weaving 을 동작을 위해서는 아래의 두 가지 방법 중 하나로 애플리케이션을 실행해야 한다.
	- AspectJ 위버로 로드 타임에 위빙하기
	- 스프링 로드 타임 위버로 로드 타임에 위빙하기
	
## AspectJ 위버로 로드 타임에 위빙하기
- AspectJ 에서는 Load-time Weaving 용 Agent 를 사용한다.
- 애플리케이션 실행 명령어에 VM 인수를 추가하면 클래스가 JVM 에 로드죄는 시점에 Weaving 을 한다.

```
java -javaagent:lib/aspectjweaver-1.9.0.jar -jar 적용하는프로젝트.jar
```  

- 애플리케이션을 실행하면 다음과 같이 캐시 상태가 출력된다.
- AspectJ Agent 는 `Complex(int, int)` 생성자를 호출 할때마다 Advice 를 적용한다.
- 출력결과

```
Cache MISS for (1,2)
Cache MISS for (2,3)
Cache MISS for (3,5)
(1 + 2i) + (2 + 3i) = (3 + 5i)
Cache MISS for (5,8)
Cache HIT for (2,3)
Cache HIT for (3,5)
(5 + 8i) - (2 + 3i) = (3 + 5i)
```  

## 스프링 로드 타임 위버로 로드 타임에 위빙하기
- 스프링은 여러 런타임 환경용 Load-time Weaver 를 제공한다.
- 스프링 애플리케이션에 Load-time Weaver 를 적용하려면 설정 클래스에 @EnableLoadTimeWeaving 을 붙인다.
- 스프링은 자신의 런타임 환경에 가장 알맞는 Load-time Weaver 를 감지한다.
- 자바 EE 애플리케이션 서버는 스프링 Load-time Weaver 메커니즘을 지원하는 클래스 로더를 이미 내장한 경우가 많아, 
애플리케이션 구동 명령어에 자바 Agent 를 지정하지 않아도 된다.
- 단순 자바 애플리케이션에서 스프링으로 Load-time Weaving 을 하려면 Weaving Agent 가 반드시 필요하므로,
애플리케이션 구동 명령어에 스프링 Agent 를 지정해야 한다.

```
java -javaagent:lib/spring-instrument-5.0.0.jar -jar 적용하는프로젝트.jar
```  

- 출력결과

```
Cache MISS for (5.3)
(1 + 2i) + (2 + 3i) = (3 + 5i)
Cache HIT for (3,5)
(5 + 8i) - (2 + 3i) = (3 + 5i)
```  

- IoC 컨테이너에 선언한 빈이 `Complex(int, int)` 생성자를 호출할 경우 스프링 Agent 가 Advice 를 적용하기 때문에 위와같은 결과가 출력된다.
- 복소수 객체를 생성한 건 Main 클래스이므로 스프링 Agent 가 생성자를 호출해도 Advice 가 적용되지 않는다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
