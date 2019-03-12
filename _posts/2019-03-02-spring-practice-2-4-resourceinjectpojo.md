--- 
layout: single
classes: wide
title: "[Spring 실습] @Resource 와 @Inject 로 POJO 자동연결"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '@Resource, @Inject 를 이용해서 POJO 를 Autowired 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - A-Inject
    - A-Resource
    - POJO
    - IoC
---  

# 목표
- @Autowired 를 사용하지 않고, 자바 표준 Annotation @Resource, @Inject 로 POJO 를 자동연결 해본다.

# 방법
- @Resource 는 JSR-250(Common Annotation for the Java Platform) 에 규정된 Annotation 이다.
- 이름을 사용하여 POJO 레퍼런스를 찾아 연결한다.
- JSR-330(표준 주입 Annotation)에 규정된 @Inject 는 타입으로 POJO 레퍼런스를 찾아 연결한다.

# 예제
- @Autowired 는 Spring Framework 의 org.springframework.bean.factory.annotation 패키지에 속해 있어서 스프링에서만 쓸 수 있다.
- 자바에서도 @Autowired 와 동일한 기능을 수행하는 Annotation 인 javax.annotation.Resource, javax.inject.Inject 를 표준화 하였다.

## @Resource 로 POJO 자동 연결하기
- 타입을 통해 POJO 를 자동연결 해주는 것은 @Resource, @Autowired 모두 동일하다.
- 아래와 같이 monitor 프로퍼티에 @Resource 를 붙이면 monitor 형 POJO가 자동 연결된다.

```java
public class Computer {
	@Resource
	private Monitor monitor;
}
```  

- 위와 같은 코드에서 같은 타임의 POJO 가 어려개 일때 @Autowired 는 가라키는 대상이 모호해 @Qualifier 를 써서 이름으로 다시 POJO 를 찾아야 한다.
- @Resource 는 기능적으로 @Autowired 와 @Qualifier 를 합한 것과 동일하다.

## @Inject 로 POJO 자동 연결하기
- @Resource 와 @Autowired 처럼 @Inject 도 일단 타입으로 POJO 를 찾는다.

```java
public class Computer {
	@Inject
	private Monitor monitor;
}
```  

- @Inject 는 @Resource, @Autowired 처럼 타입이 같은 POJO 가 여러개일 경우 다른 방법을 사용해야 한다.
- @Inject 를 이용해 이름으로 자동 연결을 하려면 먼저 POJO 주입 클래스와 주입 지점을 구별하기 위해 Custom Annotation (사용자가 제작한 Annotation) 을 작성해야 한다.

```java
@Qualifier
@Target(ElementType.Type, ElementType.Field, ElementType.PARAMETER)
@Document
@Retention(RetentionPolicy.RUNTIME)
public @interface MonitorAnnotation {
	
}
```  

- 위 Custom Annotation 에 붙인 @Qualifier 는 스프링에서 쓰는 @Qualifier 와는 다른, @Inject 와 동일한 패키지(javax.inject) 에 속한 Annotation 이다.
- Custom Annotation 을 작성한 다음, 빈 인스턴스를 생성하는 POJO 주입 클래스 즉 DellMonitor 에 붙인다.

```java
@MonitorAnnotation
public class DellMonitor implements Monitor {
	
}
```  

- Custom Annotation 을 POJO 속성 또는 주입 지점에서 사용하면 된다.

```java
public class Computer {
	@Inject @MonitorAnnotation
	private Monitor monitor;
}
```  

- @Autowired, @Resource, @Inject 셋 중 어느것을 사용하더라도 결과는 같다.
- @Autowired 는 스프링, @Resource, @Inject 는 자바 표준(JSP) 에 근거한 방법이라는 차이점만 있다.
- 이름을 기준으로 한다면 구문이 단순한 @Resource 가 낫고, 타입을 기준으로 한다면 셋중 어느것을 사용하더라도 무방하다.


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  


