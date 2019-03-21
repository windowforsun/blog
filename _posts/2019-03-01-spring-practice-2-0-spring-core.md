--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Core"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Core 란 무엇이고, 어떠한 특징이 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-core
---  

# Spring Core
- IoC(Inversion of Control) 제어의 역전은 Spring Framework 의 핵심이라고 할 수 있다.
- IoC 컨테이너는 POJO(Plain Old Java Object) 를 구성하고 관리한다.
- Spring Framework 의 가장 중요한 목적은 POJO 로 Java Application 을 개발하는 것이므로 
Spring 주요 기능은 대부분 IoC 컨테이너 안에서 POJO 를 설정 및 관리 하는 일과 연관되어 있다.
- Web Application, Enterprise 연계 또는 다양한 프로젝트에 Spring Framework 를 사용할 때 가장 중요한 부분은 POJO 와 IoC 컨테이너를 다루는 기술이다.

## 주요 기능
### Java Config Class
- @Configuration, @Bean 을 붙이면 POJO 를 생성할 수 있다.
- 이런 POJO 는 Spring 에서 @Component 를 이용해 관리한다.
- @Repository, @Service, @Controller 는 @Component 보다 더 특화된 기능을 제공하는 Annotation 이다.

### POJO 간 참조
- @Autowired 를 붙이면 타입이나 이름으로 POJO 를 자동 연결 가능하다.
- Spring 전용 @Autowired 대신 Java 표준 Annotation 인 @Resource, @Inject 를 붙여도 자동 연결된 POJO 를 참조 할 수 있다.

### POJO 의 생명주기(Scope)
- Spring POJO 의 스코프는 @Scope 로 설정한다.

### 외부 리소스
- Spring 에서 외부 리소스를 읽을 때는 @PropertySource, @Value 로 구성/생성한 POJO 에서 사용할 읽어 사용할 수 있다.

### @Bean 

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  

