--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Spring Security"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Security 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Security
---  

## Spring Security 란
- Spring 기반 애플리케이션의 보안(인증과 권한)을 담당하는 프레임워크이다.
- 보안과 관련해서 체계적으로 많은 옵션들로 세션체크 및 redirect 를 지원한다.
- Filter 를 기반으로 동작하기 때문에 Spring 의 비지니스 부분과 분리되어 관리 및 동작한다.
- XML 뿐만 아니라 JavaConfig 를 통해 간단하게 설정할 수 있다.

## 보안관련 용어
### 접근 주체(Principle)
- 보호된 대상에 접근하는 사용자
- Spring Security 에서는 `Authentication` 
### 인증(Authenticate)
- 사용자가 누구인지 확인하는 과정
- 일바나적이니 아이디/암호를 이용해 인증 처리
- Spring Security 에서는 `AuthenticationManager`
### 인가(Authorize)
- 현재 사용자가 특정 대상(URL, 기능 등)을 사용(접근) 할 권한이 있는지 검사
- Spring Security 에서는 `SecurityInterceptor`

## Spring Security 의 구조

![]()

- Spring Security 는 Session-Cookie 방식의 인증을 사용한다.

### 처리흐름
1. Http Request 를 통해 로그인 시도
1. `AuthenticationFilter` 를 거쳐 `UserDetails` 데이터가 있는 저장소에 접근
1. `UserDetails` 정보를 꺼내내고 Session 생성
1. Session 을 인메모리 세션 저장소인 `SecurityContextHolder` 에 저장
1. 클라이언트에게 Session ID 와 함께 응답
1. 로그인 이후 요청 부터는 요청쿠키에 있는 `JSESSION` 의 데이터를 통해 검증 후 Authentication 을 부여

## Authentication

![]()

- 인증에서 핵심적인 부분은 `AuthenticationManager` 인터페이스 이다.

	```java
	public interface AuthenticationManager {
       Authentication authenticate(Authentication var1) throws AuthenticationException;
   }
	```  
	
- `AuthenticationManager` 의 `authenticate` 메서드는 아래와 3가지 동작을 할 수 있다.
	1. 유효한 접근 주체일 경우 `Authentication` 을 리턴한다.
	1. 유효하지 않은 접근 주체일 경우 `AuthenticationException` 을 던진다.
	1. 결정할 수 없을 경우 null 을 리턴한다.
- `AuthenticationManager` 의 일반적인 구현체로는 `ProviderManager` 를 사용한다.
- `ProviderManager` 는 다시 `AuthenticationProvider` 에게 인증관련 동작을 위임한다.

	```java	
	public interface AuthenticationProvider {
        Authentication authenticate(Authentication var1) throws AuthenticationException;
    
        boolean supports(Class<?> var1);
    }
	```  
	
	- `support()` 메서드를 통해 `authentication()` 메서드의 인자로 패스 될 수 있는지 확인한다.
- `ProviderManager` 는 한 애플리케이션에서 다른 여러개의 인증 매커니즘을 `AuthenticationProvider` 에게 위임함으로서 제공한다.

- `/api/**` 경로와 그 하위 경로에 리소스들이 있을 때 이를 그룹화 하여 `AuthenticationManager` 를 적용 할 수 있다.
	
	![그림 3]({{site.baseurl}}/img/spring/concept-springsecurity-3.png)

	- 부모(Global 한)가 되는 `ProviderManager` 가 있고 그 하위 경로에 각각의 `ProviderManager` 와 각 인증관련 동작을 수행하는 `AuthenticationProvider` 로 구성 할 수 있다.
	
### AuthenticationManager 커스텀 하기
- `AuthenticationManagerBuilder` 를 사용하면 손쉽게 커스텀이 가능하다.
- 커스텀을 할때 대표적으로 필요한 부분들은 유저정보를 저장할 저장소(DB, In-Memory), `UserDetailService` 의 구현체 등이 있다.
- 아래 설정 코드는 Global(Parent) 한 설정의 예시이다.

	```java
	@Configuration
    public class ApplicationSecurity extends WebSecurityConfigurerAdapter {
    
       ... // web stuff here
    
      @Autowired
      public void initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
        builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
          .password("secret").roles("USER");
      }    
    }
	```  
	
	- `@Autowired` 로 `AuthenticationManagerBuilder` 를 자동 주입 받아서 설정을 수행한다.
	
- 아래 설정 코드는 Local(Child) 한 설정의 예시이다.

	```java
	@Configuration
    public class ApplicationSecurity extends WebSecurityConfigurerAdapter {
    
      @Autowired
      DataSource dataSource;
    
       ... // web stuff here
    
      @Override
      public void configure(AuthenticationManagerBuilder builder) {
        builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
          .password("secret").roles("USER");
      }
    
    }
	```  
	
	- `AuthenticationManagerBuilder` 는 Local `AuthenticationManager` 의 설정을 하고 있다.
- Spring boot 에서는 기본으로 Global `AuthenticationManager` 를 제공한다.
	- 추가적인 설정은 Local `AuthenticationManagerBuilder` 를 통해 설정해 주면된다.
	
### Authorization 과 Access Control
- `AuthenticationManager` 를 통해 인증에 성공하게 되면 다음 단계는 `Authorization` 과 `AccessControler` 이다.
- `AccessDecisionManager` 는 `Authorization`, `Accesscontroler` 의 핵심 인터페이스이다.

	```java
	public interface AccessDecisionManager {
        void decide(Authentication var1, Object var2, Collection<ConfigAttribute> var3) throws AccessDeniedException, InsufficientAuthenticationException;
    
        boolean supports(ConfigAttribute var1);
    
        boolean supports(Class<?> var1);
    }
	```  
	
- `AccessDecisionManager` 는 세 가지 동작을 가지는데, 인가와 접근제어 관련동작을 `AccessDecisionVoter` 에게 위임한다. (이는 `AuthenticationManager` 가 `ProviderManager` 에게 인증 관련 동작을 위임한것과 비슷하다.)
	
	```java
	public interface AccessDecisionVoter<S> {
        int ACCESS_GRANTED = 1;
        int ACCESS_ABSTAIN = 0;
        int ACCESS_DENIED = -1;
    
        boolean supports(ConfigAttribute var1);
    
        boolean supports(Class<?> var1);
    
        int vote(Authentication var1, S var2, Collection<ConfigAttribute> var3);
    }
	```  
	
- `AccessDecisionVoter` 는 `Authentication` 과 `ConfigAttributes` 로 장식된 안전한 `Object` 를 고려해서 결정한다.
- `Object` 는 사용자가 접근하려하는 리소스 또는 API 를 나태난다.
- `ConfigAttributes` 는 접근을 결정하는 요구 권한에 대한 메타데이터 정보를 나타낸다.

	```java
	public interface ConfigAttribute extends Serializable {
        String getAttribute();
    }
	```  
	
	- `getAttribute()` 메서드는 문자열을 리턴하는데, 접근을 위해 필요한 권한의 문자열을 의미한다.(일반적으로 ROLE_ 프리픽스를 가진다.)
- 일반적으로 기본 `AccessDecisionVoter` 인 `AffirmativeBase` 를 사용하면 된다.
	- `Voter` 중 한명이라도 허용한다면 접근이 허용된다.
- `Voter` 를 추가하거나, 기존의 `Voter` 의 동작을 수정하는 방식으로 커스텀도 할 수 있다.
- Spring Expression Language(SpEL) 를 통해 `ConfigAttributes` 를 사용한다.
	- `isFullyAuthenticated() && hasRole('Foo')`
- SpEL 표현식은 `AccessDecisionVoter` 에 의해 지원되고, 표현의 범위를 확장하기 위해서는 `SecurityExpressionRoot` 또는 `SecurityExressionHandler` 의 구현이 필요한다.
	
## Web Security
- 웹 티어에 있는 Spring Security 는 Servlet 의 `Filter` 를 기반으로 한다.
- 아래 그림은 하나의 HTTP 요청에 대한 처리 계층을 보여 주고 있다.

	![그림 4]({{site.baseurl}}/img/spring/concept-springsecurity-4.png)
	
- 요청이 앱서버로 오게 되면 해당 요청을 처리할 `Filter` 와 `Servelt` 을 결정하게 된다.
- 하나의 `Servlet` 은 하나의 요청을 처리할 수 있지만, `Filter` 는 체인을 이루고 있기 때문에 정렬된다.
- `Filter` 에서 요청을 처리할 때 남은 `Filter` 처리에 대해 거부할 수 있고, `Filter`, `Servlet` 에서 사용하는 요청과 응답을 변경할 수도 있다.
- `Filter` 를 정렬하는 것은 매우 중요한데, Spring Boot 는 이를 아래 2가지 방식으로 관리한다.
	- `Filter` 의 Bean 이 `@Order` Annotation 을 가지거나, `Ordered` 를 구현하는 하는 방식
	- `Filter` 의 Bean 이 `FilterRegistrationBean` 의 일부가 되는 방식
- Spring Security 의 `Filter` 는 `FilterChainProxy` 라는 이름으로 지정된다.
- `FilterChainProxy` 는 `ApplicationContext` 의 Bean 이면서, 모든 요청에 적용된다.
- `FilterChainProxy` 의 정렬 순서는 `SecurityProperties.DEFAULT_FILTER_ORDER` 의 값으로 지정되고, `FilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER` 의 값으로 고정된다.
- 컨테이너의 관점에서 Spring Security 의 `Filter` 는 단일 필터이지만, `FilterChainProxy` 의 내부에는 각 역할을 가진 `Filter` 들이 다시 체인을 이루고 있다.
	
	![그림 5]({{site.baseurl}}/img/spring/concept-springsecurity-5.png)
	
- Spring Security 는 물리적으론 하나의 `Filter` 이지만, 필터의 처리를 내부의 필터들에게로 위임한다.
- 컨테이너가 모든 Spring Security 필터를 알진 않지만, Spring Security 는 `FilterChainProxy` 의 레벨로 여러개의 필터를 관리 할 수 있다.
- Spring Security `Filter` 는 필터들의 체인 리스트를 포함하고 있기 때문에 `dispatcher` 에서 요청이 왔을 때 매칭되는 첫번째 `Filter` 에 요청을 전달한다.
- 아래 그림은 경로를 기반으로 `dispatcher` 에서 `Filter`로 매칭되는 상황을 보여주고 있다.

	![그림 6]({{site.baseurl}}/img/spring/concept-springsecurity-6.png)
	
	- `/foo/**` 의 필터와 매칭 되기 전에 `/**` 필터와 먼저 매칭된다.
	
### Filter Chains 만들고 커스텀 하기
- Spring Boot 애플리케이션에서 기본으로 정의된 필터 체인은 미리 정의된 `SecurityProperties.BASIC_AUTH_ORDER` 에 의해 정렬된다.
- `security.basic.enabled=false` 설정을 통해 기본설정을 해제 하거나, 이보다 더 낮은 정렬 기준으로 다른 룰(새로운 Filter)을 적용할 수 있다.
- 새로운 `Filter` 를 적용하기 위해서는 `WebSecurityConfigurerAdapter` 또는 `WebSecurityConfigurer` 타입의 빈을 생성하고 `@Order` Annotation 을 적용하면 된다.

	```java
	@Configuration
    @Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
    public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
      @Override
      protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/foo/**")
         ...;
      }
    }
	```  
	
	- 새로추가된 위 `Filter` 는 Spring Boot 의 기본 필터보다 먼저 요청을 처리하게 된다.
- UI와 API 를 모두 제공하는 애플리케이션에서 각 자원에 대한 접근 룰의 설정은 아래의 예시와 같이 다를 수 있다.
	- UI 는 로그인 페이지를 사용하고, Cookie 를 기반으로 한 인증을 사용한다.
	- API 는 토큰 기반 인증으로, 인증이 실패할 경우 401 응답을 내려주는 방식이다.
- 한 애플리케이션에서 다른 방식의 인증을 사용할 때, 각 인증 방식이 설정된 `WebSecurityConfigurerAdapter` 의 고유한 정렬을 통해 요청을 해당 `Filter` 로 매칭 시킨다.
- 다른 방식의 인증에서 매칭의 룰이 같을 경우, 정렬에서 앞단에 위치한 `Filter` 와 매칭된다.

### 요청의 전달과 인가를 위한 Request Matching
- 보안과 관련된 `Filter` 는 `Matcher` 를 통해 해당 HTTP 요청을 허용할지 결정한다.
- 어느 한 `Filter` 에서 접근이 허용되면 하위 `Filter` 들에게는 해당 요청이 전달되지 않는다.
- `HttpSecurity` 를 통해 한 `Filter` 내에서 다양하게 요청 인가와 관련된 설정을 수행 할 수 있다.

	```java
	@Configuration
    @Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
    public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
      @Override
      protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/foo/**")
          .authorizeRequests()
            .antMatchers("/foo/bar").hasRole("BAR")
            .antMatchers("/foo/spam").hasRole("SPAM")
            .anyRequest().isAuthenticated();
      }
    }
	```  
	
	- `Matcher` 는 아래와 같은 2가지 설정을 해주어야 한다.
		- `Filter` 전체에서 요청을 매칭시키는 설정
		- 접근에 대한 룰(권한)에 대한 설정
 
### Combining Application Security Rules with Actuator Rules

```java
@Configuration
@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

- `ManagementServerProperties.BASIC_AUTH_ORDER` 의 우선순위 값은 `SecurityProperties` 의 값보다 5가 작기(우선순이가 높다) 때문에, 전자의 기본 필터가 먼저 수행되고 후자 필터가 수행된다.
- 위 설정처럼 정렬값을 설정하게되면, `SecurityProperties` 의 기본 필터보다 먼저 설정 코드의 필터들이 수행된다.

## Method Security
- Spring Security 는 요청에 대한 접근제어를 통한 보안 뿐만 아니라, Java Method 에 대한 접근제어를 통한 보안을 제공한다.
- Java Method 에 대한 보안을 적용하기 위해서는 먼저 아래 설정와 같이 설정을 해주어야 한다.
	
	```java
	@SpringBootApplication
    @EnableGlobalMethodSecurity(securedEnabled = true)
    public class SampleSecureApplication {
    }
	```  
	
- Bean 으로 등록된 클래스의 메서드에 `@Secure` Annotation 을 통해 보안 설정을 하게 되면, 해당 매서드 실행 전에 Interceptor 에서 권한을 검사하고 권한이 없을 경우 `AccessDeinedException` 을 발생시킨다.
- 메서드 보안을 강제하는 Annotation 으로는 `@PreAuthorize` 와 `@PostAuthorize` 가 있고, 메서드의 파라미터와 리턴값에 대한 표현식을 작성하여 적용할 수 있다.

## Working with Threads

	
	
	
	

---
## Reference
[Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture)   
[spring security 파헤치기 (구조, 인증과정, 설정, 핸들러 및 암호화 예제, @Secured, @AuthenticationPrincipal, taglib)](https://sjh836.tistory.com/165)   
[Spring-security-구조](https://minwan1.github.io/2017/03/25/2017-03-25-spring-security-theory/)   
[Spring Security 아키텍쳐](https://happyer16.tistory.com/entry/Spring-Security-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90)   
