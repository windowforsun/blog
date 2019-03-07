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
    - Inject
    - Resource
    - POJO
---  

# 목표
- @Autowired 를 사용하지 않고, 자바 표준 Annotation @Resource, @index 로 POJO 를 자동연결 해본다.

# 방법
- @Resource 는 JSR-250(Common Annotation for the Java Platform) 에 규정된 Annotation 이다.
- 이름을 사용하여 POJO 레퍼런스를 찾아 연결한다.
- JSR-330(표준 주입 Annotation)에 규저어된 @Inject 는 타입으로 POJO 레퍼런스를 찾아 연결한다.

# 예제
- @Autowired 는 Spring Framework 의 org.springframework.bean.factory.annotation 패키지에 속해 있어서 스프링에서만 쓸 수 있다.
- 자바에서도 @Autowired 와 동일한 기능을 수행하는 Annotation 인 javax.annotation.Resource, javax.inject.Inject 를 표준화 하였다.

## @Resource 로 POJO 자동 연결하기
- 타입을 통해 POJO 를 자동연결 해주는 것은 @Resource, @Autowired 모두 동일하다.
- 아래와 같이 prefixGenerator 프로퍼티에 @Resource 를 붙이면 PrefixGenerator 형 POJO가 자동 연결된다.

```java
public class SequenceGenerator {
	@Resource
	private PrefixGenerator prefixGenerator;
}
```  

- 위와 같은 코드에서 같은 타임의 POJO 가 어려개 일때 @Autowired 는 가라키는 대상이 모호해 @Qualifier 를 써서 이름으로 다시 POJO 를 찾아야 한다.
- @Resource 는 기능적으로 @Autowired 와 @Qualifier 를 합한 것과 동일하다.

## @Inject 로 POJO 자동 연결하기
- @Resource 와 @Autowired 처럼 @Inject 도 일단 타입으로 POJO 를 찾는다.