--- 
layout: single
classes: wide
title: "[Spring 개념] Spring AOP 실행 순서 및 우선순위"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring AOP 관련 실행 순서와 우선순위를 조정하는 방법에 대해 알아 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring AOP
    - Order
    - Aspect
    - Advice
toc: true
use_math: true
---  

## Spring AOP Execution Order
`Spring` 은 여러 `AOP` 들의 조합으로 구성돼 있다고 해도 과언이 아닐 만큼, 
우리가 모르는 많은 `AOP` 들이 사용되고 있다. 
직접 구현한 하나 이상의 `AOP` 의 구성이 어느 순서에 따라 동작 되느냐에 따라 결과가 달라질 수 있기 때문에, 
`AOP` 를 사용할 때 순서를 지정하는 방법과 순서에 대한 몇가지 테스를 수행해 본다.  

이후 진행하는 테스트는 아래 `SomeService` 에 정의된 `someMethod` 를 사용한다. 

```java
@Service
@Slf4j
public class SomeService {
    public void someMethod(boolean isThrow) throws Exception {
        if(isThrow) {
            log.info("call error someMethod");
            throw new Exception("test exception");
        } else {
            log.info("call no error someMethod");
        }
    }

    public List<String> someMethod(List<String> list) {
        log.info("call someMethod : {}", Arrays.toString(list.toArray()));
        list.add("someMethod");
        return list;
    }

}
```  

- `someMethod(boolean isThrow)` 는 `isThrow` 값에 따라 예외를 던질지 말지 결정 한다. 
- `someMethod(List<String> list)` 는 인자로 전달된 `list` 를 로그로 출력하고, `someMethod` 문자열을 추가한 후 리턴 한다.

### AOP 실행 우선순위
이후 테스트에서 사용할 `@Order` 어노테이션의 설명을 바탕으로 우선순위에 대해서 정리를 먼저 진행하려고 한다. 
[@Order](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/annotation/Order.html)
설명을 읽으면 `Lower values have higher priority` 라고 되어 있다. 
즉 낮은 값을 가질 수록 높은 우선순위를 갖는다는 의미이다.  

여기서 `AOP` 관점에서 높은 우선순위는 `AOP` 가 먼저 실행 될 수록, 즉 `target method` 와 멀수록 우선순위가 높다고 할 수 있다.   

1. A AOP
1. B AOP
1. Target method
1. B AOP
1  A AOP
   
위와 같이 실행된다면 `A` 가 `B` 보다 우선순위가 높다고 할 수 있다. 

### Bean 정의에 따른 우선순위
`AOP` 를 구현해서 빈으로 등록 할때 아래 2가지 방법이 있다. 

```java
// 1
@Aspect
@Component
public class FooAspect {
    
}



// 2
@Aspect
public class BooAspect {
    
}

@Configuration
public class Config {
    @Bean
    public BooAspect() {
        return new BooAspect();
    }
}
```  

1번의 경우 `Spring AutoConfigure` 가 로드하는 순서에 따라 다르기 때문에 순서를 예측하기 힘들다. 
하지만 2번의 경우 어느 순서로 `Aspect Bean` 을 정의 하느냐에 따라 `AOP` 실행 순서가 달라질 수 있다. 
테스트에서는 2번을 통해 빈이 정의된 순서에 따라 `AOP` 실행 순서가 어떻게 달라지는지 확인해 본다.  

아래와 같은 `AOP` 클래스가 있다고 가정하자. 

```java
@Aspect
@Slf4j
public class FirstAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("first around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param first");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("first");
        log.info("first around after");

        return result;
    }
}

@Aspect
@Slf4j
public class SecondAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("second around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param second");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("second");
        log.info("second around after");

        return result;
    }
}

@Aspect
@Slf4j
public class ThirdAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("third around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param third");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("third");
        log.info("third around after");

        return result;
    }
}
```  

`FirstAspect`, `SecondAspect`, `ThirdAspect` 총 2개의 `Aspect` 클래스가 있다. 
`Pointcut` 및 내부 수행 둥작 등은 거의 동일하고 로그와 인자 그리고 결과에 `first`, `second`, `third` 문자열을 각각 사용한다는 점에만 차이가 있다. 
`AOP` 에서 수행하는 동작은 아래와 같다.  

1. `around before` 로그 출력
1. `target method` 인자에(List) 파라미터 추가
1. `target method` 호출
1. `target method` 리턴 결과에(List) 결과 추가
1. `around after` 로그 출력

아래는 `FirstAspect`, `SecondAspect`, `ThirdAspect` 순으로 빈이 정의 된 경우에 대한 테스트이다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class AopBeanFirstSecondThirdOrderTest {
    @TestConfiguration
    public static class TestConfig {
        @Bean
        public FirstAspect firstAspect() {
            return new FirstAspect();
        }

        @Bean
        public SecondAspect secondAspect() {
            return new SecondAspect();
        }

        @Bean
        public ThirdAspect thirdAspect() {
            return new ThirdAspect();
        }
    }

    @Autowired
    private SomeService someService;

    @Test
    public void test_call_return() {
        List<String> actual = this.someService.someMethod(new ArrayList<>());

        assertThat(actual, contains(
                "param first",
                "param second",
                "param third",
                "someMethod",
                "third",
                "second",
                "first"
        ));
    }
}
```  

```
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.FirstAspect     : first around before
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.SecondAspect    : second around before
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.ThirdAspect     : third around before
INFO 32448 --- [           main] c.w.springexam.aoporder.SomeService      : call someMethod : [param first, param second, param third]
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.ThirdAspect     : third around after
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.SecondAspect    : second around after
INFO 32448 --- [           main] c.w.s.aoporder.beanorder.FirstAspect     : first around after
```  

검증 문과 출력된 로그를 보면 `Bean` 이 위에서 부터 정의된 순서대로 높은 우선순위를 갖는 것을 확인 할 수 있다. 
우선순위가 높은 순으로 나열 하면 아래와 같다.  

1. `FirstAspect`
1. `SecondAspect`
1. `ThirdAspect`

이번에는 `ThirdAspect`, `FirstAspect`, `SecondAspect` 순으로 빈이 정의 된 경우를 확인해 보자.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class AopBeanThirdFirstSecondOrderTest {
    @TestConfiguration
    public static class TestConfig {
        @Bean
        public ThirdAspect thirdAspect() {
            return new ThirdAspect();
        }

        @Bean
        public FirstAspect firstAspect() {
            return new FirstAspect();
        }

        @Bean
        public SecondAspect secondAspect() {
            return new SecondAspect();
        }

    }

    @Autowired
    private SomeService someService;

    @Test
    public void test_call_return() {
        List<String> actual = this.someService.someMethod(new ArrayList<>());

        assertThat(actual, contains(
                "param third",
                "param first",
                "param second",
                "someMethod",
                "second",
                "first",
                "third"
        ));
    }
}
```  

```
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.ThirdAspect     : third around before
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.FirstAspect     : first around before
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.SecondAspect    : second around before
INFO 72476 --- [           main] c.w.springexam.aoporder.SomeService      : call someMethod : [param third, param first, param second]
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.SecondAspect    : second around after
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.FirstAspect     : first around after
INFO 72476 --- [           main] c.w.s.aoporder.beanorder.ThirdAspect     : third around after
```  

이번에도 빈의 위에서 부터 정의된 순서대로 우선순위가 결정 된 것을 확인 할 수 있다. 
우선순위가 높은 순으로 나열하면 아래와 같다.  

1. `ThirdAspect`
1. `FirstAspect`
1. `SecondAspect`


### @Order 에 따른 우선순위
앞서 먼저 잠깐 `@Order` 에 대해서 언급하고 `Bean` 정의 순서에 따라 `AOP` 의 실행 순서가 달라질 수 있다고 했었다. 
이는 즉 `@Aspect` 클래스를 정의하고 또 별도로 `@Bean` 으로 직접 순서에 맞춰서 등록할 필요 없이 아래와 같이 사용해서도 실행 순서에 우선순위를 부여할 수 있다는 의미가 된다.  

> 더 정확하게는 `AOP` 클래스가(`@Aspect`) 빈으로 등록되는 우선순위를 조정하고 그에 따라 `AOP` 실행 순서가 결정 된다고 할 수 있다. 
> 그리고 `@Component` 와 같은 어노테이션을 사용해서 `Java Config` 기반 `Auto Configure` 환경에서 순서 지정이 가능하다.  

```java
@Aspect
@Component
@Order
public class MyAspect {
    
}
```  

사용할 `AOP` 클래스는 아래와 같다.  

```java
@Slf4j
@Aspect
// @Bean 메소드로 등록 예정
public class Default2OrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param default2");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("default2");
        log.info("order around after");

        return result;
    }
}


@Slf4j
@Aspect
@Component
public class DefaultOrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("Order() around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param default");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("default");
        log.info("Order() around after");

        return result;
    }
}


@Slf4j
@Aspect
@Order(-1)
@Component
public class MinusOneOrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("Order(-1) around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param -1");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("-1");
        log.info("Order(-1) around after");

        return result;
    }
}


@Slf4j
@Aspect
@Order(-2)
@Component
public class MinusTwoOrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("Order(-2) around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param -2");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("-2");
        log.info("Order(-2) around after");

        return result;
    }
}


@Slf4j
@Aspect
@Order(1)
@Component
public class OneOrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("Order(1) around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param 1");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("1");
        log.info("Order(1) around after");

        return result;
    }
}


@Slf4j
@Aspect
@Order(2)
@Component
public class TwoOrderAspect {
    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("Order(2) around before");
        Object[] args = joinPoint.getArgs();
        if(args.length > 0 && args[0] instanceof List) {
            ((List<String>) args[0]).add("param 2");
        }
        List<String> result = (List<String>) joinPoint.proceed(args);
        result.add("2");
        log.info("Order(2) around after");

        return result;
    }
}
```  

`AOP` 에서 수행하는 동작은 아래와 같다.

1. `around before` 로그 출력
1. `target method` 인자에(List) 파라미터 추가
1. `target method` 호출
1. `target method` 리턴 결과에(List) 결과 추가
1. `around after` 로그 출력

우선순위를 테스트할 테스트 코드는 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class AopOrderTest {

    @TestConfiguration
    public static class TestConfig {
        @Bean
        public Default2OrderAspect default2OrderAspect() {
            return new Default2OrderAspect();
        }
    }

    @Autowired
    private SomeService someService;

    @Test
    public void test_call_return() {
        List<String> actual = this.someService.someMethod(new ArrayList<>());

        assertThat(actual, contains(
                "param -2",
                "param -1",
                "param 1",
                "param 2",
                "param default",
                "param default2",
                "someMethod",
                "default2",
                "default",
                "2",
                "1",
                "-1",
                "-2"
        ));
    }
}
```  

```
INFO 2024 --- [           main] c.w.s.a.o.MinusTwoOrderAspect            : Order(-2) around before
INFO 2024 --- [           main] c.w.s.a.o.MinusOneOrderAspect            : Order(-1) around before
INFO 2024 --- [           main] c.w.s.a.orderannotation.OneOrderAspect   : Order(1) around before
INFO 2024 --- [           main] c.w.s.a.orderannotation.TwoOrderAspect   : Order(2) around before
INFO 2024 --- [           main] c.w.s.a.o.DefaultOrderAspect             : Order() around before
INFO 2024 --- [           main] c.w.s.a.o.Default2OrderAspect            : around before
INFO 2024 --- [           main] c.w.springexam.aoporder.SomeService      : call someMethod : [param -2, param -1, param 1, param 2, param default, param default2]
INFO 2024 --- [           main] c.w.s.a.o.Default2OrderAspect            : order around after
INFO 2024 --- [           main] c.w.s.a.o.DefaultOrderAspect             : Order() around after
INFO 2024 --- [           main] c.w.s.a.orderannotation.TwoOrderAspect   : Order(2) around after
INFO 2024 --- [           main] c.w.s.a.orderannotation.OneOrderAspect   : Order(1) around after
INFO 2024 --- [           main] c.w.s.a.o.MinusOneOrderAspect            : Order(-1) around after
INFO 2024 --- [           main] c.w.s.a.o.MinusTwoOrderAspect            : Order(-2) around after
```  

우선순위가 높은 순서대로 나열 하면 아래와 같다.  

1. `MinusTowOrderAspect`(`@Order(-2)`, `Auto Configurer`)
1. `MinusOneOrderAspect`(`@Order(-1)`, `Auto Configurer`)
1. `OneOrderAspect`(`@Order(1)`, `Auto Configurer`)
1. `TwoOrderAspect`(`@Order(2)`, `Auto Configurer`)
1. `DefaultOrderAspect`(`Auto Configurer`)
1. `Default2OrderAspect`(`@Bean`)


### Advice 우선순위
`Spring AOP` 에는 `Before`, `After`, `Around` 등과 같은 `Advice` 가 있다. 
이때 각 `Advice` 가 어떤 우선순위를 가지고 실행되는지 테스트 해보도록 한다.  

```java
@Aspect
@Slf4j
public class AllAdviceAspect {

    @Pointcut("execution(* com.windowforsun.springexam.aoporder.SomeService.* (..))")
    public void pointcut() {

    }

    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        log.info("before");
    }

    @After("pointcut()")
    public void after(JoinPoint joinPoint) {
        log.info("after");
    }

    @AfterReturning("pointcut()")
    public void afterReturning(JoinPoint joinPoint) {
        log.info("afterReturning");
    }

    @AfterThrowing("pointcut()")
    public void afterThrowing(JoinPoint joinPoint) {
        log.info("afterThrowing");
    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        log.info("around before");
        Object result = null;
        try {
            result = proceedingJoinPoint.proceed();
            log.info("around after");
        } catch (Exception e) {
            log.info("around after error");
        }
        return result;
    }
}
```  

`AllAdviceAspect` 에는 정의 할 수 있는 모든 `Advice` 가 정의 된 상태이다. 
테스트 클래스 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class AdviceOrderTest {
    @TestConfiguration
    public static class TestConfig {
        @Bean
        public AllAdviceAspect allAdviceAspect() {
            return new AllAdviceAspect();
        }
    }

    @Autowired
    private SomeService someService;

    @Test
    public void check_advice_order_no_throwing() throws Exception {
        this.someService.someMethod(false);
    }
    /**
     *  INFO 88324 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : around before
     *  INFO 88324 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : before
     *  INFO 88324 --- [           main] c.w.springexam.aoporder.SomeService      : call no error someMethod
     *  INFO 88324 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : afterReturning
     *  INFO 88324 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : after
     *  INFO 88324 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : around after
     */

    @Test
    public void check_advice_order_throwing() throws Exception {
        this.someService.someMethod(true);
    }
    /**
     * INFO 18576 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : around before
     * INFO 18576 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : before
     * INFO 18576 --- [           main] c.w.springexam.aoporder.SomeService      : call error someMethod
     * INFO 18576 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : afterThrowing
     * INFO 18576 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : after
     * INFO 18576 --- [           main] c.w.s.a.adviceorder.AllAdviceAspect      : around after error
     */
}
```  

예외가 발생하지 않고 정상 수행 됐을 때 순서는 아래와 같다. 

1. `@Around`
1. `@Before`
1. `Target method`
1. `@After`
1. `@AfterReturning`
1. `@Around`

예외가 발생하는 경우의 순서는 아래와 같다. 

1. `@Around`
1. `@Before`
1. `@AfterThrowing`
1. `@After`
1. `@Around` (`try-cache` 가 있는 경우)



---
## Reference
[Spring AOP @Before @Around @After and other advice execution order](https://www.programmerall.com/article/32351005013/)  