--- 
layout: single
classes: wide
title: "[Spring 실습] 클라이언트 Locale(지역) 식별하고 적용하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '다국어를 지원하는 웹 에플리케이션에서 각 유저마다 로케일에 따른 콘텐츠를 제공하자'
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

```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
