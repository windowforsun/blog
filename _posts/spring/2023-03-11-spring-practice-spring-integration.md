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

## Spring Integration
`Spring Integration` 은 [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
에서 소개하는 통합(`Integration`) 패턴들을 `Spring Framework` 에서 사용할 수 있도록 구현한 프로젝트이다. 
각 패턴은 하나의 `Components`로 구현되고, 이러한 패턴들을 파이프라인으로 조립해서 메시지가 데이터를 운반하는 구조가 될 수 있다. 

`Spring Integration` 을 사용하면 스프링 기반 애플리케이션에서 경량 메시지를 사용해서, 
외부 시스템을 선언적인 방식인 어댑터를 사용해서 간편하게 통합 할 수 있다. 
이런 어댑터들은 추상화 레벨이 높기 때문에 사용자들은 좀더 비지니스에 집중해서 개발을 진행 할 수 있다.  

### Components
`Spring Integration` 은 `pipe and filters` 모델을 사용하는데, 
이런 모델 구현을 위해 아래 3가지 핵심 개념으로 구성된다. 


- `Message` : 메타데이터와 함께 결합되어 있는 자바 오브젝트를 위한 포괄적인 래퍼(`generic wrapper`) 이다. 이는 `header`, `payload` 로 구성되고 `payload` 는 자바 객체(`POJO`)를 의미하고, 메타 데이터는 메시지 `header` 의 컬렉션이다.    

  practice-spring-integration-1

- `Message Channel` : 



---  
## Reference
[Introduction to Spring Integration](https://www.baeldung.com/spring-integration)  
[Spring Integration](https://spring.io/projects/spring-integration)  
