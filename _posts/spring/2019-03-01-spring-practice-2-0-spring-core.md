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

### @Bean 초기화/폐기
- initMethod(), destroyMethod() 속성 또는 @PostConstruct, @PreDestroy 를 이용해 POJO 를 초기화, 폐기관련 로직을 커스터마이징 할 수 있다.
- @PreDestroy 로 초기화를 지연시키고, @DependOn 으로 초기화 의존 관계를 정의 할 수 있다.

### Bean(POJO) 검증
- Spring Post Processor 를 활 용해서 POJO 값을 검증 및 수정 할 수 있고, Spring 환경 및 Profile 을 이용해 여러 가지 POJO 를 로드 할 수 있다.
- @Required, @Profile 도 활용 가능하다.

### AOP
- @Aspect 를 비롯해 @Before, @After, @AfterReturning, @AfterThrowing, @Around 같은 다양한 Annotation 을 활용해 AOP 를 관련 로직 수정없이 적용 할 수 있다.
- AOP Joinpoint 정보를 가져와 다른 프로그램의 실행 포인트에 적용할 수 있다.
- @Order 로 Aspect 간 우선순위를 지정할 수 있고, Aspect Pointcut 정의부를 재활용할 수 있다.
- AspectJ Pointcut 표현식을 작성하여 AOP 를 활용 가능하다.
- AOP Introduction 개념을 활용해 여러 구현 클래스로부터 동시에 로직을 상속 받을 수 있다.
- AOP 를 통해 상태를 POJO 레 들여오거나, 로드 타임 Weaving 을 할 수 있다.
- Spring 에서 AspectJ 를 구성하거나, POJO 를 도메인 객체에 주입 할 수 있다.

### Thread/Concurrent
- Spring TaskExecutor 를 사용해서 동시성을 다룰 수 있다.

### Event
- ApplicationEvent 및 @EventListener 를 사용해서 이벤트를 생성, 발행, 리스닝 할 수 있다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  

