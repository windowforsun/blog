--- 
layout: single
classes: wide
title: "[Spring 개념] Spring 개요"
header:
  overlay_image: /img/spring-bg.jpg
subtitle: 'Spring 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Spring4 입문
    - Intro
---  

'Spring 란 무엇이고, 어떠한 특징을 가지고 있는지'

Lightweight Container 의 선두두자인 Spring Framework 를 알아보기 전에 웹 아텍쳐의 변화(발전) 과정에 대해 간단하게 알아보자.  

# 웹 아키텍쳐 발달 과정
## Non EJB 아키텍쳐
![non ejb 아키텍쳐]({{site.baseurl}}/img/spring-concept-nonejb-architecture.jpg)
- 장점
	- 별도의 교육없이 초보 개발자들도 쉽게 개발 가능
	- 초기 개발 속도가 빠르므로 비교적 단시간에 가시적 성과 가능
	- Servlet Container 만으로 운영 가능
	- Application Server, Servlet Engine 에 대해 더 좋은 이식성
	- 개발주기가 짧기 때문에 개발 생산성의 향상을 가져옴
- 단점
	- 초기 개발 속도가 빠르지만 추가 요구사항이나 변경사항에 대응 속도가 느림
	- 명확한 계층분리가 어렵고 중복코드가 많이 발생할 확률이 높음
	- 표준화된 환경과 관리 부족으로 유지 보수가 어려움
	- 분산환경 지원 불가능, 필요한 기능이 있을 때마다 구현이 필요함
	- 비지니스 POJO 객체의 생명주기를 애플리케이션에서 제어 해야 한다.
	- EJB 가 지원하는 선언적인 트랜잭션과 같은 기능 사용 불가

## EJB 아키텍쳐
![ejb 아키텍쳐]({{site.baseurl}}/img/spring-concept-ejb-architecture.jpg)
- 장점
	- 정형화된 Service Layer 제공, 논리, 물리적이니 분리 가능
	- 선언적인 트랜잭션 관리와 같은 EJB 서비스 제공
	- Business Object 를 여러 서버에 분산 가능
	- 다양한 클라이언트에 대한 지원 가능
- 단점
	- 실행속도와 개발속도등 여러 부분에서 상당한 Overhead 가 발생함
	- 개발 사이클이 복잡한 관계로 개발 생산성 저하
	- 테스트하기 어려움, 항상 EJB Container 에서만 테스트가 가능함
	- EJB 의 실행 속도 해결을 위한 변형된 패턴들이 나타나면서 객체지향적 개발에 제약사항이 됨
	- 대형 벤더들의 고유기능으로 인하여 컨테이너 사이의 이식성이 떨어짐
	
## Lightweight Container 아키텍쳐
![lightweight container]({{site.baseurl}}/img/spring-concept-lightweight-architecture.jpg)
- 장점
	- EJB 보다 배우기 쉬우며, Configuration 또한 쉽다.
	- Application Server 와 Servlet Engine 의 이식성이 높음
	- Servlet Engine 에서 실행이 가능하므로 이식성이 높음
	- IOC(Inversion of Controller)을 통한 Business Object 의 관리가 용이함
	- AOP 의 자원으로 인해 선언적인 트랜잭션 관리와 같이 EJB 에서 지원하던 기능들의 지원이 가능함
	- 특정 인터페이스에 종속되지 않은 POJO 를 기반으로 하기 때문에 테스트가 용이함
	- OOP 형태로 개발하는데 제한이 없음
- 단점
	- Remote 분산환경을 지원하지 않음, Web service 를 통하여 해결 가능
	- 현재 Lightweight Container 에 대한 표준이 없음
	- EJB 아키텍쳐와 개발자들에게 친숙하지 않음

	
# Spring Framework 개요
Java Enterprise 개발을 편하게 해주는 오픈소스 경량급 애플리케이션 프레임워크
- Java 플랫폼을 위한 오픈소스 애플리케이션 프레임워크
- Java 개발을 위한 프레임워크로 종속 객체를 생성해주고, 조립해주는 도구
- Java로 된 프레임 워크로 Java SE로 된 Java 객체(POJO)를 Java EE에 의존적이지 않게 연겨려해주는 역할

## POJO 란
- Plain Old Java Object
- 특정 규약(contract) 에 종족되지 않는다.
- POJO 는 자바 언어와 꼭 필요한 API 외에는 종속되지 않아야 한다.
- 진정한 POJO 란 객체지향적인 원리에 충실하면서, 환경과 기술에 종속되지 않고 필요에 따라 재활용될 수 있는 방식으로 설계된 오브젝트
- Java SE 로 된 오브젝트

## Spring Framework 구성
![spring framework 구성]({{site.baseurl}}/img/spring-concept-spring-composition.png)
- Spring Core
	- Spring Framework 의 근간이 되는 IoC(또는 DI) 기능을 지원한다.
	- BeanFactory 를 기반으로 Bean 클래스들을 제어할 수 있는 기능을 지원한다.
- Spring Context
	- Spring Core 바로 위에 있으면서 Spring Core 에서 지원하는 기능외에 추가적인 기능들과 좀 더 쉬운 개발이 가능하도록 지원하고 있다.
	- JNDI, EJB 등을 위한 Adapter 들을 포함하고 있다.
- Spring DAO
	- JDBC 기반하의 DAO 개발을 좀 더 쉽고, 일관된 방법으로 개발하는 것이 가능 하도록 지원한다.
	- Spring DAO 를 이용할 경우 지금까지 개발하던 DAO 보다 적은 코드와 쉬운 방법으로 DAO 를 개발하는 것이 가능하다.
- Spring ORM
	- ORM(Object Relation Mapping) 프레임워크인 Hibernate, IBatis(?), JDO 와의 결합을 지원하기 위한 기능이다.
	- Spring ORM 을 이용할 경우 위의 프레임워크 들과 쉽게 통합하는 것이 가능하다.
- Spring AOP
	- Spring Framework 에 AOP(Aspect Oriented Programming)을 지원하는 기능이다.
	- AOP Alliance 기반하에서 개발되었다.
- Spring Web
	- Web Application 개발에 필요한 Web Application Context 와 Multipart Request 등의 기능을 지원한다.
	- Struts, Webwork 와 같은 프레임워크의 통합도 지원한다.
- Spring Web MVC
	- Spring Framework 에서 독립적으로 Web UI Layer 에 Model-View-Controller 를 지원하기 위한 기능이다.
	- Struts, Webwork 가 담당했던 기능들을 Spring Web MVC 를 이용하여 대체하는 것이 가능하다.
	- Velocity, Excel, PDF 와 같은 다양한 UI 기술들을 사용하기 위한 API를 제공하고 있다.
	
# IoC 
	









---
## Reference
[Spring Framework Reference Documentation](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/)  
[스프링4 입문](https://book.naver.com/bookdb/book_detail.nhn?bid=12685135)  
[웹 개발자를 위한 Spring 4.0 프로그래밍](https://book.naver.com/bookdb/book_detail.nhn?bid=7918153)  
[Spring 프레임워크 소개와 IoC 및 Spring IoC의 개념](http://www.javajigi.net/pages/viewpage.action?pageId=3664)  
[Introduction to the Spring Framework](https://www.theserverside.com/news/1364527/Introduction-to-the-Spring-Framework)  
[[스프링] 스프링이란 무엇인가?](https://12bme.tistory.com/157)  
[스프링(Spring) 프레임워크 기본 개념 강좌 (1) - 스프링 이해하기](https://ooz.co.kr/170)  

