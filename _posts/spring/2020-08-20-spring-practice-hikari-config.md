--- 
layout: single
classes: wide
title: "[Spring 실습] HikariCP "
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
    - Spring Boot
    - HikariCP
    - MySQL
    - JPA
toc: true
use_math: true
---  

## HikariCP Config
`HikariCP` 에는 여러 옵션 값을 설정해서 구성한 애플리케이션과 환경에 적합한 설정을 할 수 있다. 
기본적으로 시간관련 값의 경우 `milliseconds` 를 사용한다. 

### 필수
#### dataSourceClassName
`JDBC` 드라이버에서 지원하는 드라이버에 대한 클래스이름을 설정한다. 
관련 리스트는 [여기](https://github.com/brettwooldridge/HikariCP#popular-datasource-class-names)
에서 확인 할 수 있다. 
기본값은 `none` 이고, 
만약 `DriverManager-based`  사용하는 속성인 `jdbcUrl` 을 사용할 경우 해당 필드는 설정할 필요 없다. 

#### jdbcUrl




























































---
## Reference
[HikariCP Configuration (knobs, baby!)](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)  

