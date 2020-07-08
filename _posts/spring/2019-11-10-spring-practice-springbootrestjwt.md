--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Security Rest JWT"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 REST API 를 Security 와 JWT 를 연동해서 구현해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - SpringBoot
    - Spring Security
    - JWT
---  

# 목표
- 클라이언트에 대한 관리를 세션이 아닌, JWT 를 사용한다.
- JWT 를 사용해서 Stateless 구조의 서버를 설계한다.
- Security 와 JWT 의 연계 과정에 대해 알아본다.
- Access Token + Refresh Token 을 사용해서 보안성을 증대시킨다.
- JWT 를 JWS + JWE 로 사용해서 보안성을 증대시킨다.

# 방법
- `JJWT(Java JWT)` 라이브러리를 사용한다.
- Spring Security 에서 관리하던 권한과 인증에 대한 정보를 세션이 아닌 JWT 에 저장한다.
- JWT 은 서버에서 정해진 `secret` 을 통해 서명되어진다.


# 예제
## 프로젝트 구조

## pom.xml


## Reference
[Java JWT: JSON Web Token for Java and Android](https://github.com/jwtk/jjwt)  