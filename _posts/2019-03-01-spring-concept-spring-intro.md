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
    - Concept
    - IoC
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
![spring framework 구성]({{site.baseurl}}/img/spring/spring-concept-overview.png)
### Core Container
-  Core Container 는 `spring-core`, `spring-beans`, `spring-context`, `spring-context-support`, `spring-expression` 모듈들로 구성 되어 있다.
- `spring-core` 와 `spring-beans` 모듈은 Spring Framework 의 기본적이면서 핵심 기능인 IoC(제어역전) 와 DI(의존성 주입)를 제공해 준다.
	- `BeanFactory` 는 팩토리 패턴의 구현채로, 코드상에서 Singleton 에 대한 구현이 필요 없어지고 실제 프로그램 로직에서 의존성의 구성과 명세를 분리하도록 제공한다.
- Context(`spring-conext`) 모듈은 Core And Beans(`spring-core` 와 `spring-beans` 모듈)을 통해 SOLID 기반으로 구현 되었고, 추가적인 기능들과 좀 더 쉬운 개발이 가능하도록 제공해 준다.
	- JNDI 레지스트리와 비슷한 프레임워크 스타일로 오브젝트에 접근 할 수 있도록 제공한다.
	- Servlet 컨테이너를 통한 국제화, 이벤트 전파, 자원 로딩을 제공한다.
	- EJB, JMX 같은 Java EE 기능을 제공한다.
	- Application Context 인터페이스는 Context 모듈이 기반이다.
- `spring-context-support` 모듈은 Spring 애플리케이션 컨텍스트에 있는 통합된 공통 Third-Party 라이브러리들에 대한 지원을 제공한다.
	- Caching : EhCache, Guava, JCache
	- Maling : JavaMail
	- Scheduling : CommonJ, Quartz
	- Template Engines : FreeMarker, JasperReports, Velocity
- `spring-expression` 모듈은 런타임에 객체를 조작할수 있는 강력한 Expression Language(JSP 2.1 사양의 Unified EL) 를 제공한다.
	- 속성값의 getter/setter, 속성 할당, 메서드 호출, 배열 요소 접근, 콜렉션 과 인덱싱, 논리 및 산술 연산자, Named 변수, IoC 컨테이너 별 빈 인스턴스 검색 등을 지원한다.

### AOP and Instrumentation


### Messaging


### Data Access/Integration


### Web


### Test

	
# IoC 
## IoC(Inversion of controller) 이란
- 기존에는 객체를 생성하고 객체간의 의존관계를 연결시키는 등의 제어권을 개발자가 직접 가지고 있었다.
- Servlet, EJB 가 등장하면서 개발자들의 독점적으로 가지고 있던 제어권이 Servlet 과 EJB 를 관리하는 컨테이너에게 넘어갔다.
- 객체에 대한 제어권이 컨테이너에게 넘어가면서 객체의 생명주기를 관리하는 권한 또한 컨테이너들이 전담하게 되었다.
- 이처럼 객체의 생성부터 생명주기의 관리까지 모든 객체에 대한 제어권이 바뀐것을 의미하는 것이 **제어권의 역전, IoC** 라는 개념이다.

## Martin Fowler 의 Inversion Of Controller Container and the Dependency Injection Pattern
- 마틴 마울러는 IoC 컨테이너와 Dependency Injection Pattern 에 대한 글로써 IoC 에 대한 개념을 잘 보여주었다.
- [IoC 컨테이너와 의존성 삽입 패턴](http://wiki.javajigi.net/pages/viewpage.action?pageId=68)


## IoC 컨테이너의 분류
![ioc 컨테이너 분류]({{site.baseurl}}/img/spring-concept-spring-ioccategory.png)

### DL(Dependency Lookup) 혹은 DP(Dependency Pull)
- Dependency Lookup 은 저장소에 저장되어 있는 빈에 접근하기 위하여 개발자들이 컨테이너에서 제공하는 API를 이용해서 사용하고자 하는 빈을 Lookup 하는 것이다.
- 아래 코드는 JNDI 에 저장되어 있는 UserService EJB 를 Lookup 하는 코드이다.

```java
public class UserServiceDelegate {
	protected static final Log logger = LogFactory.getLog(UserServiceDelegate.class);
	private static final Class homeClazz = UserServiceDelegate.class;
	private UserService userService = null;
	
	public UserServiceDelegate() {
		ServiceLocator serviceLocator = ServiceLocaltor.getInstance();
		
	}
}
```







---
## Reference
[Spring Framework Reference Documentation](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/)  
[스프링4 입문](https://book.naver.com/bookdb/book_detail.nhn?bid=12685135)  
[웹 개발자를 위한 Spring 4.0 프로그래밍](https://book.naver.com/bookdb/book_detail.nhn?bid=7918153)  
[Spring 프레임워크 소개와 IoC 및 Spring IoC의 개념](http://www.javajigi.net/pages/viewpage.action?pageId=3664)  
[Introduction to the Spring Framework](https://www.theserverside.com/news/1364527/Introduction-to-the-Spring-Framework)  
[[스프링] 스프링이란 무엇인가?](https://12bme.tistory.com/157)  
[스프링(Spring) 프레임워크 기본 개념 강좌 (1) - 스프링 이해하기](https://ooz.co.kr/170)  

