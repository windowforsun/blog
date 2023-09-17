--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
toc: true
use_math: true
---  

## Spring Retry
[Spring Retry](https://github.com/spring-projects/spring-retry)
는 실패한 동작에(예외 발생) 대해서 설정된 내용에 맞게 재시도를 수행 할 수 있도록 기능을 제공한다. 
`Spring` 대부분 기능들의 공통적인 특정처럼 기존 비지니스 로직을 크게 건들지 않고 몇가지 설정 등 추가로 적용 할 수 있다.  

어느 서비스를 개발하다 보면 단발적인 예외가 밸상하는 경우가 있다. 
이러한 예외 중 몇가지 케이스는 다시 시도를 통해 해결되는 경우가 있는데, 
이때 사용할 수 있는게 바로 `Spring Retry` 이다.  

`keep-alive` 를 통해 내부적으로 통신을 하는 두 애플리케이션이 있다고 가정한다. 
요청을 받는 서버 애플리케이션이 배포된다면 기존 `keep-alive connection` 은 서버로 인해 끊기게 된다.
(일반적으로 커넥션은 클라이언트에서 끊어야 정상적이다.) 
이런 경우 요청을 하는 클라이언트 애플리케이션에서는 `connection reset` 과 같은 예외가 발생하게 되는데, 
이 경우는 잠깐 대기 후 동일한 요청을 재시도 해주면 정상 응답이 가능한 경우가 대부분이다.  


`Spring Retry` 에서 제공하는 기능과 주요 특징을 나열 하면 아래와 같다. 

- `@EnableRetry`, `@Retryable` : `Annotation` 으로 재시작 동작을 정의하고 적용한다. `@EnableRetry` 로 애플리케이션에 선언하면 적용되고, `@Retryable` 로 재시작 타겟이 되는 메서드에 선언하며 재시도 동작을 정의 할 수 있다.  
- `RetryTemplate` : `Spring` 에서 제공하는 `RetryOperation` 의 간단한 구현체로 콜백에 직접 비지니스 로직을 넣어 재시작 동작을 정의하거나 커스텀 할 수 있다. 
- `Listeners` : `Retry Annotation` 이나 `RetryTemplate` 에 정의해서 재시도 시작, 재시도 콜백 호출, 재시도 종료 등의 콜백을 받을 수 있다. 
- `RetryContext` : `RetryCallback` 메서드가 받는 파라미터로, 실행을 반복하는 동안 필요한 데이터 속성을 저장해서 사용 할 수 있다. 
- `RecoveryCallback` : 재시도를 완료한 상태에서도 해결되지 않는 경우 `RecoveryCallback` 으로 제어권을 넘겨 최종적으로 실패한 경우 `rocevery` 로직이 실행 될 수 있도록 한다. 
- `Stateless Retry` : `RetryContext` 를 전역적인 사용하지 않고 콜 스택에서만 사용하는 경우로, 상태가 없는 재시작이라고 하며 이는 항상 동일 스레드에서 수행된다. 
- `Stateful Retry` : 트랜잭션과 역인 경우 등으로, `RetryCallback` 이 이미 콜스택에 나와 `Stateless Retry` 로는 해결이 불가한 경우 `RetryContext` 를 `RetryContextCache` 로 전역에 저장해서 사용하는 경우이다. 
- `Retry Policies` : 재사작 동작에 대한 정책을 정의 할 수 있다. 횟수, 소요시간, 적용할 예외, 제외할 예외 등이 있고, 대표적으로 `SimpleRetryPolicy` 와 `TimeoutRetryPolcy`(둘다 `Stateless Retry`) 가 있다. 
- `Backoff Policies` : 실패가 일시적인 경우 잠시 기다렸다가 하는 경우에 대한 정책을 정의한다. 대기 주기나, 시작 주기 등을 정의 할 수 있다.

### buidl.gradle
`Spring Retry` 를 프로젝트에 적용하기 위해서는 아래와 같은 의존성이 필요하다.  

```groovy
dependencies {
    // ...
    implementation 'org.springframework.retry:spring-retry'
    implementation 'org.springframework:spring-aspects'
    // ...
}
```  

### Retry Annotation
`Annotation` 방식으로 `Spring Retry` 적용은 `AOP` 방식으로 동작이 수행된다. 
그리고 이러한 방식을 활성화 하기 위해서는 `@EnableRetry` 를 설정 혹은 애플리케이션 클래스에 선언해야 한다.  

```java
@EnableRetry
@SpringBootApplication
public class ExamRetryApplication {
    public static void main(String... args) {
        SpringApplication.run(ExamRetryApplication.class, args);
    }
}
```  

`@EnableRetry` 가 프로젝트에 선언된 상태에서 
재시작 동작을 적용하고 싶은 메소드에 `@Retryable` 어노테이션을 선언하면 해당 메소드가 예외를 던졌을 때 설정된 값에 따라서 재시작을 수행한다. 
`@Retryable` 어노테이션에는 다양한 설정 값들이 있는데 그 원형을 살펴보면 아래와 같다.  

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Retryable {

    // 적용 할 recover 수행 메소드, 동일한 클래스에 존재해야 한다. 
	String recover() default "";

    // 적용 할 interceptor 빈 이름
	String interceptor() default "";

    // 재시작을 수행 할 예외
	Class<? extends Throwable>[] value() default {};

    // 재사작에 포함 시킬 예외
	Class<? extends Throwable>[] include() default {};

    // 재시작에 제외 할 예외
	Class<? extends Throwable>[] exclude() default {};

    // 재시작 통계에 사용할 라벨
	String label() default "";

    // stateless, stateful 설정
	boolean stateful() default false;

    // 최대 시도 횟수, 타겟 메소드 첫 수행도 횟수에 포함
	int maxAttempts() default 3;

    // 최대 시도 횟수 표현식 e.g. ${max.attempts}
	String maxAttemptsExpression() default "";

    // backoff 정책 e.g. @Backoff(delay = 100)
	Backoff backoff() default @Backoff();

    // 재시작 적용 판별 표현식 e.g. @retryCheckerService.isNeedRetry(#root)
	String exceptionExpression() default "";

    // 적용 할 listeners 빈 이름
	String[] listeners() default {};

}
```  

아무런 설정을 하지 않고 기본 값 그대로 `@Retryable` 을 적용하고 수행해 주면 아래 같은 결과가 출력된다. 

```java
public static AtomicInteger COUNTER = new AtomicInteger(1);

@Retryable
public int getCount() {
    int result = COUNTER.getAndIncrement();
    log.info("result : {}", result);

    if (result % 3 == 0) {
        return result;
    } else {
        throw new IllegalArgumentException("test exception");
    }
}
```  

아무런 설정없이 선언한 경우 타겟 메소드에서 어떤 예외라도 발생한다면, 1초 주기로 최대 3회 수행하게 된다.

```java
@Test
public void retryAnnotation() {
    Assertions.assertEquals(3, this.retryAnnotationCountService.getCount());
}
/*
14:48:57.548  INFO 14787 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 1
14:48:58.554  INFO 14787 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 2
14:48:59.559  INFO 14787 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 3
 */
```

그리고 아래와 같이 `@Retryable` 설정을 커스텀한 예시는 아래와 같다.  

```java
@Retryable(maxAttempts = 3,
        value = IllegalArgumentException.class,
        backoff = @Backoff(delay = 100))
public int getCountRetryAnnotationIllegalArgumentException() {
    int result = COUNTER.getAndIncrement();
    log.info("result : {}", result);

    if (result % 3 == 0) {
        return result;
    } else {
        throw new IllegalArgumentException("test exception");
    }
}

@Retryable(maxAttempts = 3,
        value = IllegalArgumentException.class,
        backoff = @Backoff(delay = 100))
public int getCountRetryAnnotationRuntimeException() {
    int result = COUNTER.getAndIncrement();
    log.info("result : {}", result);

    if (result % 3 == 0) {
        return result;
    } else {
        throw new RuntimeException("test exception");
    }
}

@Retryable(maxAttempts = 3,
        value = IllegalArgumentException.class,
        backoff = @Backoff(delay = 100))
public int getCountRetryAnnotationNumberFormatException() {
    int result = COUNTER.getAndIncrement();
    log.info("result : {}", result);

    if (result % 3 == 0) {
        return result;
    } else {
        throw new NumberFormatException("test exception");
    }
}

@Retryable(maxAttempts = 3,
        value = IllegalArgumentException.class,
        exclude = {MissingFormatArgumentException.class},
        backoff = @Backoff(delay = 100))
public int getCountRetryAnnotationMissingFormatArgumentException() {
    int result = COUNTER.getAndIncrement();
    log.info("result : {}", result);

    if (result % 3 == 0) {
        return result;
    } else {
        throw new MissingFormatArgumentException("test exception");
    }
}
```  

위 예제에 대응되는 테스트와 그 결과는 아래와 같다.  

```java
@Test
public void retryAnnotation_match_exception() {
    Assertions.assertEquals(3, 
            this.retryAnnotationCountService.getCountRetryAnnotationIllegalArgumentException());
}
/*
15:13:37.734  INFO 17007 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 1
15:13:37.839  INFO 17007 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 2
15:13:37.942  INFO 17007 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 3
 */

@Test
public void retryAnnotation_child_exception() {
    Assertions.assertEquals(3, 
            this.retryAnnotationCountService.getCountRetryAnnotationNumberFormatException());
}
/*
15:14:25.014  INFO 17056 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 1
15:14:25.117  INFO 17056 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 2
15:14:25.223  INFO 17056 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 3
 */

@Test
public void retryAnnotation_parent_exception() {
    Assertions.assertThrows(RuntimeException.class, 
            () -> this.retryAnnotationCountService.getCountRetryAnnotationRuntimeException());
}
/*
15:14:37.611  INFO 17091 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 1
 */

@Test
public void retryAnnotation_exclude_exception() {
    Assertions.assertThrows(MissingFormatArgumentException.class, 
            () -> this.retryAnnotationCountService.getCountRetryAnnotationMissingFormatArgumentException());
}
/*
15:14:52.019  INFO 17100 --- [           main] c.w.s.retry.RetryAnnotationCountService  : result : 1
 */
```  

재사작 타겟이 되는 예외는 `IllegalArgumentException` 을 지정했고, 최대 시도 횟수는 3회, 재시도 간격은 `100ms` 로 설정했다. 
설정과 일치하거나 하위 예외인 `IllegalArgumentException` 과 `NumberFormatException` 은 모두 재시작이 정상적으로 수행되지만, 
상위 예외이거나 `exclude` 에 포함된 `RuntimeException` 과 `MissingFormatArgumentException` 은 재시작이 수행되지 않고 그대로 예외를 던지게 된다.  


### RetryTemplate
`RetryTemplate` 은 `Retry Annotation` 방식 보다 더 상세하고 커스텀한 설정이 가능하다. 
재시도 과정에서 `RetryOperation` 에 있는 `RetryCallback` 을 직접 제어할 수 있으므로, 
좀 더 다양하고 커스텀한 작업을 수행 할 수 있다.  

`RetryTemplate` 에서 사용할 수 있는 `RetryOperation` 의 원형은 아래와 같다.  

```java
public interface RetryOperations {

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback) throws E;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback) throws E;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RetryState retryState) throws E, ExhaustedRetryException;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback, RetryState retryState) throws E;

}
```  

그리고 실제 재시도 동작의 로직이 담기게 되는 `RetryCallback` 원형은 아래와 같다.  

```java
public interface RetryCallback<T, E extends Throwable> {

	T doWithRetry(RetryContext context) throws E;

}
```  

`RetryTemplate` 을 사용하기 위해서는 먼저 객체를 생성하는 과정이 필요하다. 
아래 처럼 생성자와 빌더를 사용하는 2가지 방법이 가능하다.  

```java
RetryTemplate constructorRetryTemplate = new RetryTemplate();
FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
backOffPolicy.setBackOffPeriod(100);
constructorRetryTemplate.setBackOffPolicy(backOffPolicy);
    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3,
        Map.of(IllegalArgumentException.class, true,
            MissingFormatArgumentException.class, false));
constructorRetryTemplate.setRetryPolicy(retryPolicy);

// or
        
RetryTemplate builderRetryTemplate = RetryTemplate.builder()
        .maxAttempts(3)
        .fixedBackoff(100)
        .retryOn(IllegalArgumentException.class)
        .notRetryOn(MissingFormatArgumentException.class)
        .build();
```  

생성한 `RetryTemplate` 은 범용적으로 사용 할 수 있도록 빈으로 등록하면 된다.  

```java
@Configuration
@RequiredArgsConstructor
public class RetryTemplateConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        return RetryTemplate.builder()
                .maxAttempts(3)
                .fixedBackoff(100)
                .retryOn(IllegalArgumentException.class)
                .notRetryOn(MissingFormatArgumentException.class)
                .build();
    }
}
```  


`RetryTemplate` 을 사용해서 재시도를 적용한 서비스의 예시는 아래와 같다.  

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class RetryTemplateCountService {
    public static AtomicInteger COUNTER = new AtomicInteger(1);
    private final RetryTemplate retryTemplate;

    public int getCountRetryTemplateIllegalArgumentException() {
        return this.retryTemplate.execute(context -> {
            int result = COUNTER.getAndIncrement();
            log.info("result : {}", result);

            if (result % 3 == 0) {
                return result;
            } else {
                throw new IllegalArgumentException("test exception");
            }
        });
    }

    public int getCountRetryTemplateRuntimeException() {
        return this.retryTemplate.execute(context -> {
            int result = COUNTER.getAndIncrement();
            log.info("result : {}", result);

            if (result % 3 == 0) {
                return result;
            } else {
                throw new RuntimeException("test exception");
            }
        });
    }

    public int getCountRetryTemplateNumberFormatException() {
        return this.retryTemplate.execute(context -> {
            int result = COUNTER.getAndIncrement();
            log.info("result : {}", result);

            if (result % 3 == 0) {
                return result;
            } else {
                throw new NumberFormatException("test exception");
            }
        });
    }

    public int getCountRetryTemplateMissingFormatArgumentException() {
        return this.retryTemplate.execute(context -> {
            int result = COUNTER.getAndIncrement();
            log.info("result : {}", result);

            if (result % 3 == 0) {
                return result;
            } else {
                throw new MissingFormatArgumentException("test exception");
            }
        });
    }
}
```  

위 예제에 대응되는 테스트와 그 결과는 아래와 같다.  

```java
@Test
public void retryTemplate_match_exception() {
    Assertions.assertEquals(3,
            this.retryTemplateCountService.getCountRetryTemplateIllegalArgumentException());
}
/*
16:53:19.573  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 1
16:53:19.678  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 2
16:53:19.779  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 3
 */

@Test
public void retryTemplate_child_exception() {
    Assertions.assertEquals(3,
            this.retryTemplateCountService.getCountRetryTemplateNumberFormatException());
}
/*
16:53:19.796  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 1
16:53:19.901  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 2
16:53:20.004  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 3
 */

@Test
public void retryTemplate_parent_exception() {
    Assertions.assertThrows(RuntimeException.class,
            () -> this.retryTemplateCountService.getCountRetryTemplateRuntimeException());
}
/*
16:53:19.791  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 1
 */

@Test
public void retryTemplate_exclude_exception() {
    Assertions.assertThrows(MissingFormatArgumentException.class,
            () -> this.retryTemplateCountService.getCountRetryTemplateMissingFormatArgumentException());
}
/*
16:53:19.788  INFO 75528 --- [           main] c.w.s.retry.RetryTemplateCountService    : result : 1
 */
```  

`RetryTemplate` 설정은 앞서 진행한 `Retry Annotation` 과 동일하다. 
설정과 일치하거나 하위 예외인 `IllegalArgumentException` 과 `NumberFormatException` 은 모두 재시작이 정상적으로 수행되지만,
상위 예외이거나 `exclude` 에 포함된 `RuntimeException` 과 `MissingFormatArgumentException` 은 재시작이 수행되지 않고 그대로 예외를 던지게 된다.  


### Recover
`Recover` 는 설정된 재시도 동작을 모두 수행 했음에도 실패한 경우 후처리를 담당하는 로직으로, 
`Retry Annotation` 방식을 사용하는 경우 `@Recover` 어노테이션을 통해 메서드에 선언해 지정 할 수 있는데, 
필요한 조건은 아래와 같다. 

- `@Retryable` 이 정의된 메소드와 동일한 클래스에 위치
- `@Recover` 메소드의 맨 첫 파라미터는 `@Retryable` 가 수행되는 예외 클래스 이거나 부모 클래스 
- `@Recover` 메소드는 `@Retryable` 메소드와 반환 타입 및 파라미터가 동일 해야함
- `@Recover` 어노테이션이 해당 클래스에 한 개만 존재한다면, 별도 선언 없이 `recover` 동작 수행
- 동일 클래스에 2개 이상의 동일하게 적용 가능한 `@Recover` 메소드가 있다면, `@Retryable` 의 `recover` 필드로 메소드 이름으로 지정 가능

`RetryTemplate` 을 사용하는 경우에는 `execute()` 메소드에 적용할 `recover` 동작을 정의할 수 있다.  

아래는 `Retry Annotation` 과 `RetryTemplate` 을 사용해서 `Recover` 를 적용한 예시이다.  

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class RecoverCounterService {
    public static AtomicInteger COUNTER = new AtomicInteger(1);
    private final RetryTemplate retryTemplate;
    
    @Retryable(maxAttempts = 3,
            value = IllegalArgumentException.class,
            backoff = @Backoff(delay = 100))
    public int getCountRetryAnnotationIllegalArgumentException() {
        int result = COUNTER.getAndIncrement();
        log.info("result : {}", result);

        if (result % 4 == 0) {
            return result;
        } else {
            throw new IllegalArgumentException("test exception");
        }
    }
    
    @Retryable(maxAttempts = 3,
            value = IndexOutOfBoundsException.class,
            backoff = @Backoff(delay = 100),
            recover = "recoverGetCountRetryAnnotationIndexOutOfBoundsException2")
    public int getCountRetryAnnotationIndexOutOfBoundsException(String param1) {
        int result = COUNTER.getAndIncrement();
        log.info("result : {}", result);

        if (result % 4 == 0) {
            return result;
        } else {
            throw new IndexOutOfBoundsException("test exception");
        }
    }
    
    @Retryable(maxAttempts = 3,
            value = IndexOutOfBoundsException.class,
            backoff = @Backoff(delay = 100),
            recover = "recoverGetCountRetryAnnotationIndexOutOfBoundsException2")
    public int getCountRetryAnnotationRuntimeException(String param1) {
        int result = COUNTER.getAndIncrement();
        log.info("result : {}", result);

        if (result % 4 == 0) {
            return result;
        } else {
            throw new RuntimeException("test exception");
        }
    }
    
    @Recover
    public int recoverGetCountRetryAnnotationIllegalArgumentException(IllegalArgumentException e) {
        log.info("recoverGetCountRetryAnnotationIllegalArgumentException");
        return -1;
    }

    @Recover
    public int recoverGetCountRetryAnnotationIndexOutOfBoundsException1(IndexOutOfBoundsException e, String param1) {
        log.info("recoverGetCountRetryAnnotationIndexOutOfBoundsException1 : {}", param1);
        return -1;
    }

    @Recover
    public int recoverGetCountRetryAnnotationIndexOutOfBoundsException2(RuntimeException e, String param1) {
        log.info("recoverGetCountRetryAnnotationIndexOutOfBoundsException2 : {}", param1);
        return -2;
    }
    
    public int getCountRetryTemplateIllegalArgumentException() {
        return this.retryTemplate.execute(context -> {
                    int result = COUNTER.getAndIncrement();
                    log.info("result : {}", result);

                    if (result % 4 == 0) {
                        return result;
                    } else {
                        throw new IllegalArgumentException("test exception");
                    }
                },
                context -> {
                    log.info("recoverGetCountRetryTemplateIllegalArgumentException");
                    return -1;
                });
    }
}
```  

위 예제에 대응되는 테스트와 그 결과는 아래와 같다.  

```java
@Test
public void retryAnnotation_illegalArgumentException_recover() {
    Assertions.assertEquals(-1,
            this.recoverCounterService.getCountRetryAnnotationIllegalArgumentException());
}
/*
06:25:37.432  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 1
06:25:37.533  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 2
06:25:37.635  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 3
06:25:37.635  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : recoverGetCountRetryAnnotationIllegalArgumentException
 */

@Test
public void retryAnnotation_indexOutOfBoundsException_recover() {
    Assertions.assertEquals(-2,
            this.recoverCounterService.getCountRetryAnnotationIndexOutOfBoundsException("test param"));
}
/*
06:25:37.221  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 1
06:25:37.326  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 2
06:25:37.429  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 3
06:25:37.429  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : recoverGetCountRetryAnnotationIndexOutOfBoundsException2 : test param
 */

@Test
public void retryAnnotation_runtimeException_recover() {
    Assertions.assertEquals(-2,
            this.recoverCounterService.getCountRetryAnnotationRuntimeException("test param"));
}
/*
06:25:37.218  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 1
06:25:37.218  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : recoverGetCountRetryAnnotationIndexOutOfBoundsException2 : test param
 */

@Test
public void retryTemplate_illegalArgumentException_recover() {
    Assertions.assertEquals(-1,
            this.recoverCounterService.getCountRetryTemplateIllegalArgumentException());
}
/*
06:25:36.997  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 1
06:25:37.103  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 2
06:25:37.208  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : result : 3
06:25:37.208  INFO 40951 --- [           main] c.w.spring.retry.RecoverCounterService   : recoverGetCountRetryTemplateIllegalArgumentException
 */
```  

먼저 `Retry Annotation` 의 결과를 살펴 보면 아래와 같다. 
`getCountRetryAnnotationIllegalArgumentException` 는 `@Retryable` 에 `recover` 를 따로 지정해 주지 않았지만, 
매칭되는 `@Recover` 메소드가 1개만 있어서 자동으로 `recoverGetCountRetryAnnotationIllegalArgumentException` 가 수행됐다. 
그리고 `getCountRetryAnnotationIndexOutOfBoundsException` 는 와 매칭될 수 있는 `@Recover` 메소드는 2개가 있지만, 
`@Retryable` 의 `recover` 에 설정한 `recoverGetCountRetryAnnotationIndexOutOfBoundsException2` 가 
`RuntimeException` 은 `IndexOutOfBoundsException` 의 부모 클래스 이므로 수행 될 수 있다. 
다른 경우로 `getCountRetryAnnotationRuntimeException` 는 `RuntimeException` 을 던지기 때문에 
`@Retryable` 에는 해당되지 않지만 `recoverGetCountRetryAnnotationIndexOutOfBoundsException2` 가 `RuntimeException` 을 받기 때문에 
`@Recover` 는 정상적으로 수행 될 수 있다.  

`RetryTemplate` 을 사용한 경우에는, 
`execute()` 메소드에서 `RetryCallback` 의 다음 파라미터에 필요한 `RecoverCallback` 을 적절하게 정의해주면 `recover` 동작을 적용 할 수 있다.  


### Listener
`Listern` 는 재시도 수행 과정 사이에 `RetryContext`, `RetryCallback` 등의 파라미터를 받아, 
로깅이나 필요한 검증 및 추가 동작을 수행 할 수 있다. 
`Listener` 는 `RetryListener` 인터페이스를 구현하는 방식으로 사용 가능한데 그 원형은 아래와 같다.  

```java
public interface RetryListener {

	<T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback);

	<T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable);

	<T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable);

}
```  

`Retry Annotation` 방식의 경우 구현된 `RetryListener` 를 빈으로 등록해주면 자동으로 사용 가능하고, 
`RetryTemplate` 방식의 경우 직접 `RetryListener` 를 `RetryTemplate` 에 설정해야 한다.  

```java
@Slf4j
@Component
public class MyRetryListener implements RetryListener {
    @Override
    public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
        log.info("open : {}, {}", context, callback);
        return true;
    }

    @Override
    public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
        log.info("close : {}, {}, {}", context, callback, throwable);
    }

    @Override
    public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
        log.info("onError : {}, {}, {}", context, callback, throwable);
    }
}
```  

```java
@Configuration
@RequiredArgsConstructor
public class RetryTemplateConfig {
    private final MyRetryListener myRetryListener;

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();
        retryTemplate.registerListener(this.myRetryListener);

        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(100);
        retryTemplate.setBackOffPolicy(backOffPolicy);

        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3,
                Map.of(IllegalArgumentException.class, true,
                        MissingFormatArgumentException.class, false));
        retryTemplate.setRetryPolicy(retryPolicy);

        return retryTemplate;
    }
}
```  
