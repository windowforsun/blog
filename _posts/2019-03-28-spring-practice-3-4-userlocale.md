--- 
layout: single
classes: wide
title: "[Spring 실습] 클라이언트 Locale(지역) 식별하고 적용하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '다국어를 지원하는 웹 에플리케이션에서 각 유저 로케일에 따른 콘텐츠를 제공하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-webmvc
    - Locale
---  

# 목표
- 다국어를 지원하는 Web Application 에서 각 유저마다 Locale 을 식별하고 해당하는 콘텐츠를 화면에 표시하자.

# 방법
- Spring MVC Application 에서 유저 로케일은 LocaleResolver 인터페이스를 구현한 Locale Resolver 가 식별한다.
- 로케일을 해석하는 기준에 따라 여러 LocaleResolver 구현체가 Spring MVC 에 구현되어 있다.
- 직접 LocaleResolver 인터페이스를 구현해서 Custom Locale Resolver 를 만들어 사용할 수도 있다.
- Locale Resolver 는 Web Application Context 에 LocaleResolver 형 빈으로 등록한다.
- DispatcherServlet 이 자동 감지하려면 Locale Resolver 빈을 localeResolver 라고 명시한다.
- Locale Resolver 는 DispatcherServlet 당 하나만 등록 할 수 있다.
	
# 예제
- Spring MVC 가 제공하는 다양한 LocaleResolver 를 살펴보고 인터셉터로 유저 로케일을 어떻게 변경하는지 알아 본다.

## HTTP 요청 헤더에 따라 로케일 해석하기
- AcceptHeaderLocaleResolver 는 Spring 기본 Locale Resolver 로서 요청 헤더 값에 accept-language 로 로케일을 해석한다.
	- 클라이언트 웹 브라우저는 자신을 실행한 운영체제의 로케일 설정으로 이 헤더를 설정한다.
	- 클라이언트 운영체제의 로케일 설정을 바꿀 수는 없으므로 Locale Resolver 로 클라이언트 로케일을 변경하는 것 역시 불가능하다.

## 세션 속성에 따라 로케일 해석하기
- SessionLocaleResolver 는 클라이언트 세션에 사전 정의된 속성에 따라 로케일을 해석한다.
- 세션 속성이 없으면 요청 헤더 값인 accept-language 헤더로 기본 로케일을 결정한다.

```java
@Bean
public LocaleResolver localeResolver() {
	SessionLocaleResolver localeResolver = new SessionLocaleResolver();
	localeResolver.setDefaultLocale(new Locale("en"));
	
	return localeResolver;
}
```  

- 로케일 관련 세션 속성이 없을 경우 setDefaultLocale() 메서드로 대체 프로퍼티 defaultLocale 을 설정할 수 있다.
- 세션 LocaleResolver 는 로케일이 저장된 세션 속성을 변경함으로써 클라이언트 로케일을 변경한다.

## 쿠키에 따라 로케일 해석하기
- CookieLocaleResolver 는 유저 브라우저의 쿠키값에 따라 로케일을 해석한다.
- 쿠키가 없으면 accept-language 해더로 기본 로케일을 설정한다.

```java
@Bean
public LocaleResolver localeResolver() {
	return new CookieLocaleResolver();
}
```  

- 쿠키 설정은 cookieName, cookieMaxAge 프로퍼티로 커스텀 할 수 있다.
- cookieMaxAge 는 쿠키를 유지할 시간(초)이고, -1은 브라우저 종료와 동시에 쿠키가 삭제되니다.

```java
@Bean
public CookieLocaleResolver localeResolver() {
	CookieLocaleResolver cookieLocaleResolver = new CookieLocaleResolver();
	cookieLocaleResolver.setCookieName("language");
	cookieLocaleResolver.setCookieMaxAge(3600);
	cookieLocaleResolver.setDefaultLocale(new Locale("en"));
	return cookieLocaleResolver;
}
```  

- 해당 쿠키가 존재하지 않으면 setDefaultLocale() 메서드로 대체 프로퍼티 defaultLocale 을 설정할 수 있다.
- 쿠키 LocaleResolver 는 로케일이 저장된 쿠키값을 변경함으로써 클라이언트 로케일을 변경한다.

## 클라이언트 로케일 변경하기
- LocaleResolver.setLocale() 호출로 클라이언트 로케일을 명시적으로 변경한다.
- LocaleChangeInterceptor 를 핸들러 매핑에 적용하여 변경한다.
	- Interceptor 는 현재 HTTP 요청에 특정한 매개변수가 존재하는지 감지할 수 있다.
	- 특정한 매개변수의 이름은 Interceptor 의 paramName 프로퍼티값으로 지정하고, 그 값으로 클라이언트 로케일을 변경한다.

```java
@Configuration
public class I18NConfiguration implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor localeChangeInterceptor = new LocaleChangeInterceptor();
        localeChangeInterceptor.setParamName("language");
        return localeChangeInterceptor;
    }

    @Bean
    public CookieLocaleResolver localeResolver() {
        CookieLocaleResolver cookieLocaleResolver = new CookieLocaleResolver();
        cookieLocaleResolver.setCookieName("language");
        cookieLocaleResolver.setCookieMaxAge(3600);
        cookieLocaleResolver.setDefaultLocale(new Locale("en"));
        return cookieLocaleResolver;
    }
}
```  

- 위의 설정 파일을 통해 요청 URL 의 language 매개변수를 통해 클라이언트의 로케일을 변경할 수 있다.
- 클라이언트 로케일을 영어-미국(en_US), 독일어(de) 로케일로 변경하려면 아래 URL 로 접속한다.
	- `http://localhost:8080/court/welcome?language=en_US`
	- `http://localhost:8080/court/welcome?language=de`
	


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
