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
	
## AuthenticationManager 커스텀 하기
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
	
## Authorization 과 Access Control(접근제어)
	
	
	
---

- 모든 접근 주체(유저)에 대해 `Authentication` 을 생성한다.
	
	```java
	public interface Authentication extends Principal, Serializable {
        Collection<? extends GrantedAuthority> getAuthorities();
    
        Object getCredentials();
    
        Object getDetails();
    
        Object getPrincipal();
    
        boolean isAuthenticated();
    
        void setAuthenticated(boolean var1) throws IllegalArgumentException;
    }
	```  
	
- Authentication 은 `SecurityContext` 에 보관되고 사용된다.
	
	```java
	public interface SecurityContext extends Serializable {
        Authentication getAuthentication();
    
        void setAuthentication(Authentication var1);
    }
	```  
	
- Authentication 은 Spring Security 의 `AuthenticationManager` 를 통해 처리된다.

	```java
	public interface AuthenticationManager {
       Authentication authenticate(Authentication var1) throws AuthenticationException;
   }
	```  

- `AuthenticationManager` 의 구현체로 `ProviderManager` 가 있는데, 인증 구현은 `AuthenticationProvider` 에 위임한다.
	- `AuthenticationProvider` 를 커스텀마이징해서 인증을 구현할 수 있다.
	
	```java
	public interface AuthenticationProvider {
        Authentication authenticate(Authentication var1) throws AuthenticationException;
    
        boolean supports(Class<?> var1);
    }
	```  

## Spring Security Filter 의 흐름

![]()

---
## Reference
[Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture)   
[spring security 파헤치기 (구조, 인증과정, 설정, 핸들러 및 암호화 예제, @Secured, @AuthenticationPrincipal, taglib)](https://sjh836.tistory.com/165)   
[Spring-security-구조](https://minwan1.github.io/2017/03/25/2017-03-25-spring-security-theory/)   
[Spring Security 아키텍쳐](https://happyer16.tistory.com/entry/Spring-Security-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90)   
