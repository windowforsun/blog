--- 
layout: single
classes: wide
title: "[Spring 실습] Filter, Interceptor, AOP 이용한 요청/응답 공통 처리 "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '요쳥/응답에 대한 공통 부분을 처리할 수 있도록 해주는 Filter, Interceptor, AOP 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - Filter
  - Interceptor
  - AOP
toc: true
---  

## 요청/응답의 공통 처리
- Web Application 을 개발하다보면 요청/응답에 대해 공통으로 처리할 부분이 발생한다.
- 세션체크, 권한, 인증, 로깅 등 이러한 요소는 다양하게 존재할 수 있다.
- 이런 작업들을 모든 Controller 에서 처리하게 될 경우 중복되는 코드가 많아지고 가독성에 좋지 않다.
- 공통적으로 처리해야 되는 부분을 따로 분리해서 처리하는 구성을 할떄 사용할 수 있는 방법은 아래와 같다.
	- Filter
	- Interceptor
	- AOP
- 위 3가지 모두 가능하지만, 각각의 처리흐름이 다르기 때문에 적절한 부분에 알맞는 동작을 구현해 구성 할 수 있다.

## 요청/응답의 처리 흐름

![그림 1]({{site.baseurl}}/img/spring/practice_filter_interceptor_aop_1.png)

- 위 그림에서 알 수 있는 것과 같이 `Filter -> Interceptor -> AOP` 순으로 실행 된다.
- `Filter` 는 Spring Framework 와는 무관하게 `Servlet` 단위에서 실행된다.
- `DispatcherServlet` 을 시작으로 `Interceptor` 와 `AOP` 는 Spring Framework 안에서 동작한다.
- 전체적인 흐름은 `Filter -> Interceptor -> AOP -> Controller -> AOP -> Interceptor -> Filter` 가 된다.

### Filter
- 요청/응답을 거르고 정제하는 역할이다.
- 요청 `Path` 를 기준으로 대상을 설정할 수 있다.
- `Filter` 에서 처리하는 요청/응답은 `ServletRequest/ServletResponse` 이다.
- `DispatcherServlet` 앞단에서 요청/응답 내용 수정 및 필요한 검증을 수행할 수 있다.
- Spring Framework 과 별도의 자원에서 동작하기 때문에 `Bean` 에 접근할 수 없다.
- 인코딩 변환, 로그인확인, 권한체크, XSS 방어 등 처리를 수행할 수 있다.
- 하나 이상의 `Filter` 를 등록해 처리할 수 있고, `Filter` 간 우선순위 설정도 가능하다.
- `Filter` 를 구현할 때는 아래 `Filter` 인터페이스를 구현한다.

	```java
	public interface Filter {
        default void init(FilterConfig filterConfig) throws ServletException {
        }
    
        void doFilter(ServletRequest var1, ServletResponse var2, FilterChain var3) throws IOException, ServletException;
    
        default void destroy() {
        }
    }
	```  
	
	- `init()` : `Filter` 인스턴스 초기화
	- `doFilter()` : 전/후 처리
	- `destroy()` : `Filter` 인스턴스 종료
- `Filter` 는 `@WebFilter` 애노테이션을 사용한 방법과 `FilterRegistryBean` 을 사용한 방법, 애노테이션을 사용한이 있다.

#### @WebFilter 애노테이션을 사용한 방법
- `Filter` 간 우선순위 설정이 불가능하다.

```java
@WebFilter("/**")
public class CustomFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        // 요청 전 처리
        filterChain.doFilter(modifyRequestWrapper, modifyResponseWrapper);
        // 응답 후 처리
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 초기화
    }

    @Override
    public void destroy() {
        // 종료
    }
}
```  

- `@WebFilter` : 적용될 경로를 설정 및 `Filter` 를 등록한다.		
- `@ServletComponentScan` 을 통해 `@WebFilter` 을 등록해 준다.

	```java
	@SpringBootApplication
    @ServletComponentScan
    public class ExamApplication {        
        public static void main(String[] args) {
            SpringApplication.run(ExamApplication.class, args);
        }        
    }
	```  
		
#### `FilterRegistrationBean` 을 사용한 방법

```java
public class CustomFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        // 요청 전 처리
        filterChain.doFilter(modifyRequestWrapper, modifyResponseWrapper);
        // 응답 후 처리
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 초기화
    }

    @Override
    public void destroy() {
        // 종료
    }
}
```  

- Java Config 를 통해 `Filter` 를 등록하고 설정한다.

	```java
	@Configuration
      public class FilterConfiguration implements WebMvcConfigurer {
         @Bean
         public FilterRegistrationBean filterRegistrationBean() {
             FilterRegistrationBean bean = new FilterRegistrationBean(new CustomFilter());
             bean.setOrder(2);
             bean.addUrlPatterns("/*");
             return bean;
         }
      }
	```  
		
#### 애노테이션을 사용한 방법
- 경로설정이 불가능하다.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)
public class CustomFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        // 요청 전 처리
        filterChain.doFilter(modifyRequestWrapper, modifyResponseWrapper);
        // 응답 후 처리
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 초기화
    }

    @Override
    public void destroy() {
        // 종료
    }
}
```  

- `@Component` 로 빈으로 등록한다.
- `@Order` 로 `Filter` 우선순위를 설정한다.
	
### Interceptor
- 요청 처리에 대해서 전/후로 가로챈다.
- 요청 `Path` 를 기준으로 대상을 설정할 수 있다.
- `Interceptor` 에서 처리하는 요청/응답은 `HttpServletRequest/HttpServletResponse` 이다.
- `DispatcherServlet` 이후인 Spring Framework 안에서 동작하기 때문에 `Bean` 에 접근해 동작을 처리할 수 있다.
- 하나 이상의 `Interceptor` 를 등록해 처리할 수 있고, `Interceptor` 간 우선순위를 설정할 수 있다.
- `Bean` 을 사용한 공통처리 등인 세션체크, 권한등 처리를 수행할 수 있다.
- `Interceptor` 를 구현할때는 `HandlerInterceptor` 인터페이스를 구현할 수도있고, `HandlerInterceptorAdapter` 를 상속받아 구현할 수 있다.

	```java
	public interface HandlerInterceptor {
        default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            return true;
        }
    
        default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
        }
    
        default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        }
    }
	```  
	
	- `preHandle()` : Controller 메소드가 실행되기 전에 실행된다.
	- `postHandler()` : Controller 메소드가 실행된 후 실행된다.
	- `afterCompletion()` : `View` 페이지가 렌더링 된 후 실행된다.
	
	```java
	public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {
        public HandlerInterceptorAdapter() {
        }
    
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            return true;
        }
    
        public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
        }
    
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        }
    
        public void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        }
    }
	```  
	
	- `afterConcurrentHandlerStarted()` : 비동기 요청시 `postHandler()` 와 `afterCompletion` 은 실행되지 않고, 해당 메소드가 실행된다.
- `Interceptor` 를 구현하면 아래와 같다.

	```java
	@Component
    public class CustomInterceptor implements HandlerInterceptor {
        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        	// 요청 전 처리
            return true;
        }
    
        @Override
        public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        	// 요청 후 처리
        }
    
        @Override
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        	// View 렌더링 후 처리
        }
    }
	```  
	
	- `@Component` : 빈으로 등록한다.
	- `preHandler()` 에서 `true` 를 리턴하면 해당 요청을 처리하고, `false` 를 리턴하면 요청을 처리하지 않는다.
- `Interceptor` 는 Java Config 에서 등록 작업이 필요하다.

	```java
	@Configuration
    public class AppConfig implements WebMvcConfigurer {
        @Autowired
        private CustomInterceptor customInterceptor;
    
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(this.customInterceptor).addPathPatterns("/**").order(1);
        }
    }
	```  
	
	- `order()` : `Interceptor` 의 우선순위를 설정한다.
	
### AOP
- 특정 메소드 전/후에 처리를 수행하는 OOP 보완 목적으로 나온 관점지향 프로그래밍 방식이다.
- `AOP` 는 특정 위치에 있는 Controller Method 를 대상으로 전/후에 대한 공통처리가 가능하다.
- `Filter`, `Interceptor` 보다 폭넓은 조건(파라미터, 애노테이션, 패키지)를 사용해서 대상을 설정할 수 있다.
- `Bean` 을 사용한 트랜잭션, 에러처리, 로깅 등 아주 다양한 범위에서 사용될 수 있다.
- 아래와 같은 공통 처리 범위가 존재한다. (이런 처리에 대한 부분을 AOP 에서는 `Advice` 라고 한다.)
	- `@Around` : 메소드 실행 전/후에 수행
	- `@Before`: 메소드 실행 전에 수행
	- `@After` : 메소드 실행 후에 수행
	- `@AfterReturning` : 메소드 정상 실행 후 수행
	- `@AfterThrowing` : 메소드 실행시 예외 빌생 후 수행
- 더 자세한 설명은 [AOP]({{site.baseurl}}{% link _posts/2019-03-07-spring-practice-2-13-aop.md %})
에서 확인할 수 있다.
- `AOP` 를 구현할 때는 새로운 의존성과 Application 에 애노테이션 추가가 필요하다.

	```xml	
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>	
	```  
	
	```java
	@SpringBootApplication
    @EnableAspectJAutoProxy
    public class ExamApplication {    
        public static void main(String[] args) {
            SpringApplication.run(ExamApplication.class, args);
        }    
    }
	```  
	
	- `@EnableAspectJAutoProxy` 을 추가해야 `AOP` 프로그래밍이 가능하다.

- `AOP` 의 모든 `Advice` 에 대해 구현하면 아래와 같다.

	```java
	@Aspect
    @Component
    public class ControllerAdvice {
        @Pointcut("execution(* com.windowforsun.exam.controller..*.*(..))")
        public void pointcut() {}
        
        @Before("pointcut()")
        public void before(JoinPoint joinPoint) {
        	// 메소드 실행 전 처리
        }
    
        @Around("pointcut()")
        public Object around(ProceedingJoinPoint joinPoint) throws Throwable{
        	// 메소드 실행 전 처리
            Object result = joinPoint.proceed();
    		// 메소드 실행 후 처리
            return result;
        }
    
        @After("pointcut()")
        public void after(JoinPoint joinPoint) {
        	// 메소드 실행 후 처리
        }
    
        @AfterReturning(pointcut = "allPointcut()", returning = "result")
        public void afterReturning(JoinPoint joinPoint, Object result) {
        	// 메소드 실행 완료 후 처리
        }
        
        @AfterThrowing(pointcut = "pointcut()", throwing = "e")
        public void afterThrowing(JoinPoint joinPoint, Throwable e) {
        	// 메소드 실행 중 에러발생 후 처리
        }
    }
	```  
	
	- `@Aspect` : 현재 클래스가 `AOP` 의 처리에 대한 부분이 작성된 클래스임을 설정한다.
	- `@Component` : 빈으로 등록한다.
	- `@Pointcut`, `pointcut()` : `Pointcut` 는 실행 대상을 설정하는 부분으로, 해당 애노테이션으로 메소드를 정의하면 메소드를 바탕으로 대상 범위를 예제와 같이 공유해서 사용할 수 있다.


## 예제
- 예제를 통해 실제 공통처리에 대한 실행 흐름과 `Filter` 를 통한 요청/응답을 수정하는 부분에 대해 알아본다.

### 디렉토리 구조

```bash
src
├─main
│  ├─java
│  │  └─com
│  │      └─windowforsun
│  │          └─exam
│  │              │  ExamApplication.java
│  │              │
│  │              ├─advice
│  │              │      AopAdvice.java
│  │              │      ExceptionAdvice.java
│  │              │
│  │              ├─config
│  │              │      AppConfig.java
│  │              │
│  │              ├─controller
│  │              │  │  AllController.java
│  │              │  │
│  │              │  └─a
│  │              │          AController.java
│  │              │
│  │              ├─exception
│  │              │      CustomException.java
│  │              │
│  │              ├─filter
│  │              │      AFilter.java
│  │              │      AllFilter.java
│  │              │
│  │              ├─interceptor
│  │              │      AInterceptor.java
│  │              │      AllInterceptor.java
│  │              │
│  │              └─util
│  │                     Util.java
│  │
│  └─resources
│      │  application.properties
│      │
│      ├─static
│      └─templates
└─test
    └─java
        └─com
            └─windowforsun
                └─exam
                    └─controller
                            AControllerTest.java
                            AllControllerTest.java
```  
	
### pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```  

### Application

```java
@SpringBootApplication
@ServletComponentScan
public class ExamApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExamApplication.class, args);
    }
}
```  

- `@EnableAspectJAutoProxy` 를 통해 `AOP` 를 활성화 했다.

### Java Config

```java
@Configuration
public class AppConfig implements WebMvcConfigurer {
    @Autowired
    private AllInterceptor allInterceptor;
    @Autowired
    private AInterceptor aInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(this.allInterceptor).addPathPatterns("/**").order(Ordered.HIGHEST_PRECEDENCE + 1);
        registry.addInterceptor(this.aInterceptor).addPathPatterns("/a/**").order(Ordered.HIGHEST_PRECEDENCE + 2);
    }

    @Bean
    public FilterRegistrationBean allFilter() {
        FilterRegistrationBean bean = new FilterRegistrationBean(new AllFilter());
        bean.setOrder(1);
        bean.addUrlPatterns("/*");

        return bean;
    }

    @Bean
    public FilterRegistrationBean aFilter() {
        FilterRegistrationBean bean = new FilterRegistrationBean(new AFilter());
        bean.setOrder(2);
        bean.addUrlPatterns("/a/*");
        return bean;
    }
}
```  

- `Interceptor` 과 `Filter` 를 등록하고 설정하는 내용이 있다.
- `Filter` 는 설정된 우선순위에 따라 `AllFilter`, `AFilter` 순으로 실행된다.
	- `AllFilter` 는 모든 URL 에서 실행된다.
	- `AFilter` 는 `/a` 의 하위 URL 에서 실행된다.
- `Interceptor` 도 설정된 우선순위에 따라 `AllInterceptor`, `AInterceptor` 순으로 실행된다.
	- `AllInterceptor` 는 모든 URL 에서 실행된다.
	- `AInterceptor` 는 `/a` 의 하위 URL 에서 실행된다.

### Util

```java
public class Util {
    public static LinkedList<String> LIST = new LinkedList<>();

    public static String getMethodName(Throwable throwable) {
        return throwable.getStackTrace()[0].getMethodName();
    }

    public static String getClassName(Throwable throwable) {
        return throwable.getStackTrace()[0].getClassName();
    }

    public static String getSignature(Throwable throwable) {
        return getClassName(throwable) + "." + getMethodName(throwable);
    }

    public static String getSignature(Throwable throwable, String info) {
        return getSignature(throwable) + ":" + info;
    }

    public static void addList(Throwable throwable) {
        addList(getSignature(throwable));
    }

    public static void addList(Throwable throwable, String info) {
        addList(getSignature(throwable, info));
    }

    private static void addList(String str) {
        LIST.addLast(str);
    }
}
```  

- 메소드가 실행 될떄마다 `LIST` 정적필드에 메소드의 정보가 차례대로 추가된다.
- `getMethodName()`, `getClassName()` 은 인자값의 `Throwable` 을 사용해서 호출한 곳의 클래스와 메소드이름을 리턴한다.
- `getSignature()` 는 클래스와 메소드 이름을 통해 고유한 정보를 만들어 리턴한다.
	- `a.b.c.D` 클래스에서 `method()` 메소드가 `getSignature()` 를 호출하면 리턴값은 `a.b.c.D.method` 가 된다.
- `addList()` 는 `LIST` 필드에 `getSignature()` 의 정보를 추가한다.

### CustomException 

```java
public class CustomException extends Exception {

}
```  

- `CustomException` 는 `Controller` 에서 발생하는 예외를 의미한다.

### Controller

```java
@RestController
@RequestMapping("/a")
public class AController {
    @PostMapping
    public void post() {
        Util.addList(new Throwable());
    }
    
    @PostMapping("/exception")
    public void exception() throws Exception{
        Util.addList(new Throwable());
        throw new CustomException();
    }
}
```  

```java
@RestController
public class AllController {
    @PostMapping
    public void post() {
        Util.addList(new Throwable());
    }
    
    @PostMapping("/exception")
    public void exception() throws Exception{
        Util.addList(new Throwable());
        throw new CustomException();
    }
}
```  

### Filter

```java
public class AllFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Util.addList(new Throwable(), "before");
        filterChain.doFilter(servletRequest, servletResponse);
        Util.addList(new Throwable(), "after");
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    	// Spring Framework 영역이 아니기 때문에 Test 시 검증될수 없어 제외한다.
    }

    @Override
    public void destroy() {
    	// Spring Framework 영역이 아니기 때문에 Test 시 검증될수 없어 제외한다.
    }
}
```  

```java
public class AFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Util.addList(new Throwable(), "before");
        filterChain.doFilter(servletRequest, servletResponse);
        Util.addList(new Throwable(), "after");
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    	// Spring Framework 영역이 아니기 때문에 Test 시 검증될수 없어 제외한다.
    }

    @Override
    public void destroy() {
    	// Spring Framework 영역이 아니기 때문에 Test 시 검증될수 없어 제외한다.
    }
}
```  

### Interceptor

```java
@Component
public class AllInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Util.addList(new Throwable());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        Util.addList(new Throwable());
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        Util.addList(new Throwable());
    }
}
```  

```java
@Component
public class AInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Util.addList(new Throwable());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        Util.addList(new Throwable());
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        Util.addList(new Throwable());
    }
}
```  

### AOP

```java
@Aspect
@Component
public class AopAdvice {
    @Pointcut("execution(* com.windowforsun.exam.controller..*.*(..))")
    public void allPointcut() {}

    @Pointcut("execution(* com.windowforsun.exam.controller.a..*.*(..))")
    public void aPointcut() {}

    @Before("allPointcut()")
    public void beforeAll(JoinPoint joinPoint) {
        Util.addList(new Throwable());
    }

    @Around("allPointcut()")
    public Object aroundAll(ProceedingJoinPoint joinPoint) throws Throwable{
        Util.addList(new Throwable(), "before");
        Object result = joinPoint.proceed();
        Util.addList(new Throwable(), "after");

        return result;
    }

    @After("allPointcut()")
    public void afterAll(JoinPoint joinPoint) {
        Util.addList(new Throwable());
    }

    @AfterReturning(pointcut = "allPointcut()", returning = "result")
    public void afterReturningAll(JoinPoint joinPoint, Object result) {
        Util.addList(new Throwable());
    }

    @AfterThrowing(pointcut = "allPointcut()", throwing = "e")
    public void afterThrowingAll(JoinPoint joinPoint, Throwable e) {
        Util.addList(new Throwable());
    }

    @Before("aPointcut()")
    public void beforeA(JoinPoint joinPoint) {
        Util.addList(new Throwable());
    }

    @Around("aPointcut()")
    public Object aroundA(ProceedingJoinPoint joinPoint) throws Throwable{
        Util.addList(new Throwable(), "before");
        Object result = joinPoint.proceed();
        Util.addList(new Throwable(), "after");

        return result;
    }

    @After("aPointcut()")
    public void afterA(JoinPoint joinPoint) {
        Util.addList(new Throwable());
    }

    @AfterReturning(pointcut = "aPointcut()", returning = "result")
    public void afterReturningA(JoinPoint joinPoint, Object result) {
        Util.addList(new Throwable());
    }

    @AfterThrowing(pointcut = "aPointcut()", throwing = "e")
    public void afterThrowingA(JoinPoint joinPoint, Throwable e) {
        Util.addList(new Throwable());
    }
}
```  

- `allPointcut()` 은 모든 경로에 적용되는 `Pointcut` 이다.
- `aPointcut()` 은 `/a` 하위 경로에 적용되는 `Pointcut` 이다.
- `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` 종류의 `Advice` 를 `allPointcut()`, `aPointcut()` 에 각각 등록 한다.

### ExceptionAdvice

```java
@ControllerAdvice
public class ExceptionAdvice {
    @ExceptionHandler(CustomException.class)
    public ResponseEntity<?> customException(CustomException customException) {
        Util.addList(new Throwable());
        return new ResponseEntity<>(null, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```  

- `ExceptionAdvice` 는 `@ControllerAdvice` 가 선언된, `Controller` 에서 발생하는 예외를 처리하는 클래스이다.

### 테스트

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class AllControllerTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @After
    public void tearDown() {
        Util.LIST.clear();
    }

    @Test
    public void post_CallSequence() {
        // when
        this.restTemplate.postForObject("/", null, Void.class);

        // then
        System.out.println(Util.LIST);
        List<String> actual = Util.LIST;
        assertThat(actual, contains(
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:before",
                "com.windowforsun.exam.advice.AopAdvice.beforeAll",
                // Controller
                "com.windowforsun.exam.controller.AllController.post",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:after",
                "com.windowforsun.exam.advice.AopAdvice.afterAll",
                "com.windowforsun.exam.advice.AopAdvice.afterReturningAll",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }

    @Test
    public void exception_CallSequence() {
        // when
        this.restTemplate.postForObject("/exception", null, Void.class);

        // then
        System.out.println(Util.LIST);
        List<String> actual = Util.LIST;
        assertThat(actual, contains(
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:before",
                "com.windowforsun.exam.advice.AopAdvice.beforeAll",
                // Controller
                "com.windowforsun.exam.controller.AllController.exception",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.afterAll",
                "com.windowforsun.exam.advice.AopAdvice.afterThrowingAll",
                // ControllerAdvice
                "com.windowforsun.exam.advice.ExceptionAdvice.customException",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class AControllerTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @After
    public void tearDown() {
        Util.LIST.clear();
    }

    @Test
    public void post_CallSequence() {
        // when
        this.restTemplate.postForObject("/a", null, Void.class);

        // then
        System.out.println(Util.LIST);
        List<String> actual = Util.LIST;
        assertThat(actual, contains(
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                "com.windowforsun.exam.filter.AFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                "com.windowforsun.exam.interceptor.AInterceptor.preHandle",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundA:before",
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:before",
                "com.windowforsun.exam.advice.AopAdvice.beforeA",
                "com.windowforsun.exam.advice.AopAdvice.beforeAll",
                // Controller
                "com.windowforsun.exam.controller.a.AController.post",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:after",
                "com.windowforsun.exam.advice.AopAdvice.aroundA:after",
                "com.windowforsun.exam.advice.AopAdvice.afterA",
                "com.windowforsun.exam.advice.AopAdvice.afterAll",
                "com.windowforsun.exam.advice.AopAdvice.afterReturningA",
                "com.windowforsun.exam.advice.AopAdvice.afterReturningAll",
                // Interceptor
                "com.windowforsun.exam.interceptor.AInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AllInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AInterceptor.afterCompletion",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AFilter.doFilter:after",
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }

    @Test
    public void exception_CallSequence() {
        // when
        this.restTemplate.postForObject("/a/exception", null, Void.class);

        // then
        System.out.println(Util.LIST);
        List<String> actual = Util.LIST;
        assertThat(actual, contains(
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                "com.windowforsun.exam.filter.AFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                "com.windowforsun.exam.interceptor.AInterceptor.preHandle",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.aroundA:before",
                "com.windowforsun.exam.advice.AopAdvice.aroundAll:before",
                "com.windowforsun.exam.advice.AopAdvice.beforeA",
                "com.windowforsun.exam.advice.AopAdvice.beforeAll",
                // Controller
                "com.windowforsun.exam.controller.a.AController.exception",
                // AOP
                "com.windowforsun.exam.advice.AopAdvice.afterA",
                "com.windowforsun.exam.advice.AopAdvice.afterAll",
                "com.windowforsun.exam.advice.AopAdvice.afterThrowingA",
                "com.windowforsun.exam.advice.AopAdvice.afterThrowingAll",
                // ControllerAdvice
                "com.windowforsun.exam.advice.ExceptionAdvice.customException",
                // Interceptor
                "com.windowforsun.exam.interceptor.AInterceptor.afterCompletion",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AFilter.doFilter:after",
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }
}
```  

- 테스트 결과는 요청 경로(`Filter`, `Interceptor`), 예외 발생 여부에 따라 달라진다.
- 예외가 발생하지 않은 경우 결과는 `Filter -> Interceptor -> AOP -> Controller -> AOP -> Interceptor -> Filter` 순으로 실행된다.
- 예외가발생한 경우 결과는 `Filter -> Interceptor -> AOP -> Controller -> AOP -> ControllerAdvice -> Interceptor -> Filter` 순으로 실행된다.

---
## Reference
[Spring Boot Filter](https://www.concretepage.com/spring-boot/spring-boot-filter)  
[Spring MVC: filter filter, interceptor interceptor, AOP difference](http://www.programmersought.com/article/9752520747/)  
[(Spring)Filter와 Interceptor의 차이](https://supawer0728.github.io/2018/04/04/spring-filter-interceptor/)  
[[Spring] Filter, Interceptor, AOP 차이 및 정리](https://goddaehee.tistory.com/154)  
[Spring - Filter, Interceptor, AOP](https://doublesprogramming.tistory.com/133)  
