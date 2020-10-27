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
toc: true
use_math: true
---  

## JPA Persistence Context
`JPA` 를 사용하면 `JPA` 와 데이터베이스 사이에 영속성 컨텍스트(`Persistence Context`) 
라는 논리적인 개념을 두고 데이터를 관리한다. 
`Persistence Context` 는 데이터베이스의 데이터인 `Entity` 를 영구 저장하는 환경이다. 
이런 `Persistence Context` 는 `EntityManager` 를 통해 `Entity` 를 관리하는데, 
이러한 `EntityManager` 를 생성하는 것이 바로 `EntityManagerFactory` 이다. 

### EntityManagerFactory 
`EntityManagerFactory` 를 생성하는 비용은 비교적 


## Entity LifeCycle


## Persistence Context 장점

### 1차캐시

### 동일성

### 쓰기 지연

### Dirty Checking

## Persistence Context 특징
### flush

### remove

### detached

### merge


---
## Reference
[[JPA] 영속성 컨텍스트와 플러시 이해하기](https://ict-nroo.tistory.com/130)  
[JPA - Persistence Context (영속성 컨텍스트)](https://heowc.tistory.com/55)  
[더티 체킹 (Dirty Checking)이란?](https://jojoldu.tistory.com/415)  
[JPA 더티 체킹(Dirty Checking)이란?](https://interconnection.tistory.com/121)  
[JPA 변경 감지와 스프링 데이터](https://medium.com/@SlackBeck/jpa-%EB%B3%80%EA%B2%BD-%EA%B0%90%EC%A7%80%EC%99%80-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-2e01ad594b82)  
[(JPA - 2) 영속성(Persistence) 관리](https://kihoonkim.github.io/2017/01/27/JPA(Java%20ORM)/2.%20JPA-%EC%98%81%EC%86%8D%EC%84%B1%20%EA%B4%80%EB%A6%AC/)  
[[Spring JPA] 영속 환경 ( Persistence Context )](https://victorydntmd.tistory.com/207)  
[Getting started with Spring Data JPA](https://spring.io/blog/2011/02/10/getting-started-with-spring-data-jpa)  
[JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)  