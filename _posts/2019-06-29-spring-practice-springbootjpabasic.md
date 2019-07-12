--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot JPA 사용기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 JPA 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - JPA
    - Spring Boot
---  

# 목표
- Spring JPA 사용법에 대해 익힌다.
- JPA 기본 동작 외에 커스텀 메서드를 사용해본다.

# 방법
- Spring JPA 란

# 예제
- 완성된 프로젝트의 구조는 아래와 같다.

![그림 1]({{site.baseurl}}/img/practice-springbootjpabasic-1.png)

- pom.xml 에서 사용된 의존성은 아래와 같다.

```xml
<dependencies>
	<!-- Spring JPA 의존성 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

	<!-- 데이터 저장소로 사용할 Java Embedded DB H2 의존성 -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Java Dev Tool Lombok 의존성 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- Spring Boot Test 의존성 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```  

---
## Reference
[Spring Boot JPA 사용해보기](https://velog.io/@junwoo4690/Spring-Boot-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0-erjpw41nl7)  