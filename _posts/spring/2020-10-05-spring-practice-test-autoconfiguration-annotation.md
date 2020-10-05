--- 
layout: single
classes: wide
title: "[Spring 실습] Test Auto-configuration Annotations"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Junit
    - Auto-configuration
    - Test Annotation
toc: true
use_math: true
---  

## Test Auto-configuration Annotations
`Spring Boot` 에서 `@...Test` 와 같이 `Test` 로 끝나는 어노테이션은 테스트에 해당하는 설정을 자동 설정해주는 역할을 한다. 
어노테이션의 종류와 자동 대상이되는 리스트는 [여기](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html#test-auto-configuration)
에서 확인할 수 있다.  

본 포스트에서 모든 종류를 다루지 않으므로 설명할 어노테이션은 아래와 같다. 

Annotation|Desc|Bean
---|---|---
@SpringBootTest|통합 테스트, 전체|Bean 전체
@WebMvcTest|단위 테스트, MVC 테스트|MVC 관련 Bean
@DataJpaTest|단위 테스트, JPA 테스트|JPA 관련 Bean
@DataJdbcTest|단위 테스트, JDBC 테스트|JDBC 관련 Bean
@DataRedisTest|단위 테스트, Redis 테스트|Redis 관련 Bean
@RestClientTest|단위 테스트, Rest API 테스트|Rest API 관련 Bean
@JsonTest|단위 테스트, Json 테스트|Json 관련 Bean


### @SpringBootTest



---
## Reference
[Spring Boot test annotation](http://wonwoo.ml/index.php/post/1926)  
[Spring Boot Test](https://cheese10yun.github.io/spring-boot-test/)  
[Test Auto-configuration Annotations](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html#test-auto-configuration)   
