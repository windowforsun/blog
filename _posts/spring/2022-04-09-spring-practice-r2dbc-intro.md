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
    - Spring Boot
toc: true
use_math: true
---  

## R2DBC
[R2DBC](https://r2dbc.io/) 
는 `Reactive Stream`(`Non-Blocking`) 으로 구현된 `DataSource` 라고 할 수 있고, 
이는 `Blocking` 방식인 `JDBC` 의 대체제라고도 할 수 있다. 
`Spring Webflux` 를 사용한다면 `JDBC` 보다는 `R2DBC` 를 통해 완벽한 `Reactive Stream` 를 구현 할 수 있다.  

`Spring Webflux` 로 인해 `Java` 및 `Spring` 진영에서도 `Reactive Stream` 의 활용과
실제 서비스에서 사용되는 빈도가 높아지고 있다. 
`Reactive Stream` 기반인 `Spring Webflux` 를 100% 활용하기 위해서는 
애플리케이션에서 요청을 받고 응답하는 모든 흐름이 `Reactive` 하게 구현돼야 한다. 
흐름 구간에서 하나의 `blocking` 만 발생하더라도 `Spring Webflux` 의 활용도가 크게 떨어진다.  

웹 애플리케이션의 잘 알려진 구조는 `Application + DB` 인
애플리케이션이 요청 처리를 위해서 `DB` 를 조회해 그 결과를 응답해주는 구조일 것이다.  

`Spring MVC` 를 사용할 때는 `Spring MVC + JDBC` 를 조합을 사용하게 된다. 
몇전 전까지만 하더라도 `Non-Blocking` 데이터소스가 없어 
`Spring Webflux` 에도 `Blocking` 방식인 `JDBC` 조합을 많이 사용했었다. 
`JDBC` 관련 동작은 모두 `Schedulers` 를 사용해서 `Reactive Stream` 상에 `Blocking` 을 발생하지 않도록 회피 하는 방법이 많이 사용됐다. 
`R2DBC` 를 통해 `Non-Blcoing` 방식인 데이터소스 사용방법에 대해 알아본다.   


## Spring Data R2DBC
[Spring Data R2DBC](https://github.com/spring-projects/spring-data-r2dbc)  
는 `Spring Data` 의 `R2DBC` 버전으로 해당 의존성을 사용하면 `Spring Boot` 프로젝트에서 
보편적인 방법으로 `R2DBC` 를 사용할 수 있다.  

`Spring Data R2DBC` 의 특징을 간단하게 나열하면 아래와 같다. 

- `Spring JPA` 와는 다른 구현체이다. 
- `ORM` 이 아니므로 `Spring Data JPA` 에서 사용가능한 모든 기능을 사용할 수는 없다. 
- `QueryDSL` 이 현재 공식적으로 지원은 되지 않는다. 
  - [외부 라이브러리](https://github.com/infobip/infobip-spring-data-querydsl) 를 사용해서 연동하는 방법은 있는 것 같다.
- `Spring Webflux` 와 조합이 매우 좋다. 

### 의존성
테스트에 사용한 `Spring Boot` 버전은 `Spring Boot 2.5.0` 을 사용했다. 
`Spring Data R2DBC` 사용을 위해서는 아래와 같은 의존성 추가가 필요하다.  

```groovy
implementation "org.springframework.boot:spring-boot-starter-data-r2dbc"

// H2 를 사용하는 경우
runtimeOnly "com.h2database:h2"
runtimeOnly "io.r2dbc:r2dbc-h2"

// MySQL 을 사용하는 경우
runtimeOnly "mysql:mysql-connector-java"
runtimeOnly "dev.miku:r2dbc-mysql"
```  






---  
## Reference
[]()  