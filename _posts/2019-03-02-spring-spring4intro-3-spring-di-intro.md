--- 
layout: single
classes: wide
title: "[Spring4 입문] Spring DI - Intro"
header:
  overlay_image: /img/spring-bg.jpg
subtitle: 'Spring DI 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - DI
    - Intro
    - Spring4 입문
---  

'Spring DI 란 무엇이고, 어떠한 특징을 가지고 있는지'

# DI 란
- DI는 인터페이스를 이용해 컴포넌트화를 실현하는 것이다.
- DI는 Dependency Injection의 약자로 의존 관계 주입으로 객체 사이에 관계를 만드는 것이다.
- 어떤 객체의 프로퍼티(인스턴스 변수)에 그 객체가 이용할 객체를 설정한다는 의미이다.
- 다른말로 어떤 객체가 의존(이용)할 객체를 주입 혹은 인젝션(Injection) 한다는 것이다.
- DI를 구현하는 컨테이너는 이 밖에도 클래스의 인스턴스화 등의 생명 주기 관리 기능이 있다.


# 예시
- 제품(Product)을 관리하는 웹 애플리케이션  
- 간단한 예시를 위해서 실제 데이터베이스는 사용하지 않음

## 인터페이스와 DI를 사용하지 않은 경우
![인터페이스와 DI를 이용하지 않은 예]({{site.baseurl}}/img/spring/spring-spring4intro-3-spring-di-intro-notinterface-ex-1.png)
- main 메서드가 있는 ProductSampleRun 클래스에서 웹 애플리케이션에서의 뷰와 컨트롤러 역할을 대신한다고 가정한다.
- 비지니스 로직을 담당하는 ProductService
- 데이터베이스 액세스를 담당하는 ProductDao
- ProjectSampleRun에서 ProductService를 new하고 ProductService가 ProductDao를 new 한다음 각각의 인스턴스를 생성해 이용하는 형태이다.

## DI를 사용하는 경우
![DI를 사용하는 예]({{site.baseurl}}/img/spring/spring-spring4intro-3-spring-di-intro-usedi-ex-1.png)
- DI 컨터에너는 ProductSampleRun이 사용하는 ProductService의 인스턴스, ProductService가 사용하는 ProductDao의 인스턴스를 생성한다.
- DI 컨테이너는 ProductDao의 인스턴스를 사용하는 ProductService 등과 같은 참조가 필요한 부분에 인젝션(의존 관계 주입)을 한다.
- DI 컨테이너가 인스턴스를 생성할 클래스, 인스턴스를 전달받을 클래스 모두 POJO(Plain Old Java Object)(컨테이너나 프레임워크에 독립적인 자바 오브젝트)로 작성해도 문제 없다.
- DI 컨테이너로 클래스에서 new 연산자가 사라졌고, 결과적으로 개발자가 팩토리 메서드 패턴 같은 디자인 패턴을 구사하지 않아도 DI 컨테이너를 통해 인터페이스 기반의 컴포넌트화를 구현할 수 있다.
- DI 를 이용할때는 원칙은 클래스는 인터페이스에 의존하고 실현 클래스에는 의존하지 않아야 한다 이다.

## 인터페이스와 DI를 사용하는 경우
![인터페이스와 DI를 사용하는 예]({{site.baseurl}}/img/spring/spring-spring4intro-3-spring-di-intro-useinterfacedi-ex-1.png)
- DI 컨테이너의 인스턴스화를 통해 생성된 인스턴스는 Singleton 객체로 생성된다. (default 설정의 경우)
- 생성된 인스턴스는 필요한 곳에 인젝션해 사용된다.

![인터페이스와 DI를 사용하는 예 시퀀스 다이어그램]()



---
## Reference
[스프링4 입문](https://book.naver.com/bookdb/book_detail.nhn?bid=12685135)  
