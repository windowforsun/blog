--- 
layout: single
classes: wide
title: "[Spring 실습] POJO 와 IoC 컨테이너 리소스"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'POJO 에게 IoC 컨테이터의 리소스를 알려주자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
---  

# 목표
- 컴포넌트가 IoC 컨테이너와 직접적인 의존 관계를 가지도록 설계하는 방법은 바람직하지 않지만, 때로는 빈에서 컨테이너 리소스를 인지해야 하는 경우가 존재한다.

# 방법
- 빈이 IoC 컨테이너 리소스를 인지하게 하려면 Aware 인터페이스를 구현해야 한다.
- 스프링은 이 인터페이스를 구현한 빈은 감지해 대상 리소스를 setter 메서드로 주입한다.

Aware 인터페이스 | 대상 리소스 타입
---|---
BeanNameAware | IoC 컨테이너에 구성한 인스턴스의 빈 이름
BeanFactoryAware | 현재 빈 팩토리 컨테이너, 서비스를 호출한다.
ApplicationContextAware | 현재 애플리케이션, 컨텍스트 컨테이너 서비스를 호출한다.
MessageSourceAware | 메시지 소스, 텍스트 메시지를 해석한다.
ApplicationEventPublisherAware | 애플리케이션 이벤트 발생기, 애플리케이션 이벤트를 발생한다.
ResourceLoaderAware |리소스 로더, 외부 리소스를 로드한다.
EnvironmentAware | ApplicationContext 인터페이스에 묶인 org.springframework.core.env.Environment 인스턴스


# 예제

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
