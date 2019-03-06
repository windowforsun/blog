--- 
layout: single
classes: wide
title: "SOA 개요"
header:
  overlay_image: /img/web-bg.jpg
subtitle: 'SOA 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Web
tags:
    - Web
    - SOA
    - Intro
---  

'SOA 란 무엇이고, 어떠한 특징을 가지고 있는지'

# SOA(Service Oriented Architecture) 란
![SOA 구조1]({{site.baseurl}}/img/web-soa-ex-1.jpg)
- 기존 애플리케이션의 기능들을 비지니스적인 의미를 가지는 기능 단위로 묶어서 표준화된 호출 인터페이스를 통해 서비스를 구현 한 것
- 이 서비스들을 서로 조합(Orchestration)하여 업무 기능을 구현한 애플리케이션을 만드는 소프트웨어 아키텍쳐이다.
- 기존의 시스템이 각각 독립된 업무 시스템으로 개발되었다면, SOA는 기업의 전체 업무가 하나의 거대한 SOA 시스템으로 구성된다.
- SOA는 새로운 개념이 아니라 소프트웨어 개발에 있어 계속해서 야기 되어 왔던, **소프트웨어의 재사용성**과 **레고웨어**(소프트웨어를 레고블럭처럼 재사용하고 조립하는 것)의 연장선이다.
- SAO가 주목 받는 이유
	- 서비스 구축의 표준 인터페이스에 대한 방안인 CORBA의 기술의 난이도가 높았다는 점
	- XML/HTTP나 SOAP 기반의 웹서비스 기술등의 등장으로 서비스 구현의 기술 난이도 문제 해결
	- 각 엄무별 독립된 시스템의 형태로 개발이 되어, 이에 대한 통합이 필요하게 됨
	- 급격한 비지니스 환경의 변화에 따라 비지니스의 요구를 민첩하게 IT시스템에 반영되어야 할 필요성이 대두됨
	

# 서비스 란
**플랫폼**에 종속되지 않는 표준 인터페이스(CORBA, 웹서비스)를 통해서 기업의 업무를 표헌한 Loosely coupled하고 상호 조합 가능한 소프트웨어 이다.
- SOA에서 서비스의 플랫폼 종속성은 SOAP 기반의 웹서비스 또는 XML을 통해서 구현된다.
- 서비스를 표현하는데 있어서 가장 중요한 특징은 **기업의 업무를 표현한다**는 것이다.
- 서비스의 예시
	- 임직원 정보 서비스
	- 계좌이체 서비스
	- 상품 주문 서비스
- 서비스로 적절하지 않은 것
	- JNDI Lookup
	- SMTP 이메일 클라이언트

## 서비스의 구성
비지니스적 의미를 가지는 기능(Method)들을 모아 놓은 소프트웨어 컴포넌트이다.  
![서비스의 구성]({{stie.baseurl}}/img/web-soa-service-architecture-1.jpg)
- 서비스 인터페이스
	- 서비스내의 하나의 업무 기능을 뜻한다.
	- 주문 서비스라는 서비스가 있을 때, 상품 주문과 주문 내용 조회라는 인터페이스를 가진다.
- 서비스 규약 (Contract)
	- 서비스를 사용하기 위한 여러가지 규약(데이터 포맷, 변수형, 서비스를 호출하기 위한 인자, 인터페이스 이름 등)들이 정의 되는 곳이 서비스 Contract 이다.
	- 현대 SOA에서는 대부분이 웹서비스를 표준 인터페이스로 사용하기 때문에 서비스 Contract는 WSDL로 정의된다.
- 서비스의 구현체 (Implementation)
	- 이러한 서비스의 실제 구현체가 Implementation 이다.

## 서비스의 특징
- 수직적 분할(Vertical Slicing)
	- ![서비스의 특징 수직 분할]({{site.baseurl}}/img/web-soa-service-feature-verticalslicing-1.png)
	- 수직적 분할이란 애플리케이션을 개발할 때 전체 애플리케이션을 여러 개의 서비스로 나누고 각각의 서비스를 독립적으로 개발하는 것을 이야기 한다.
	- 이전 소프트웨어 개발은 애플리케이션을 각 Data Layer, Business Logic, View Layer와 같이 수평적(Horizontal)으로 분리하였다.
	- 그러나 SOA에서는 각각의 서비스가 DataLayer, Business Logic, View Layer에 대한 모듈을 모두 가지고 있고 그래서 각 서비스간의 의존성이 최소화 된다.
- Has standard interface
	- 서비스가 제공하는 인터페이스는 표준 기술로 구현되어야 한다.
	- 서비스를 사용하고자 하는 사람이 **서비스 규약** 만을 가지고도 해상 서비스를 호출할 수 있어야 하며, 이는 해당 SOA시스템 내에서 플랫폼이나 기술에 족속되지 않아야 한다.
- Loosely coupled
	- Vertical Slicing 에서도 설명했듯이 각 서비스 컴포넌트들은 다른 서비스에 대해서 의존성이 최소화어야 한다.
	- 서비스의 구현 내용 등을 변경 하였을 때 다른 서비스가 이에 의해서 영향을 거의 받지 않아야 한다.
- Composed
	- 서비스 컴포넌트들은 서로 연결되어 조합된 형태의 하나의 애플리케이션을 구성해야 한다.
	- 서비스간에 연결 및 조립이 가능해야한다.
- Coarse grained
	- 서비스의 구성단위나 인터페이스의 단위는 업무 단위를 기본으로 한다.
	- IT 개발 조직이 아니라 현업 비지니스 조직이라도 해당 서비스가 무엇이고 무슨 기능을 하는지 이해할 수 있어야 한다.
- Discoverable
	- 서비스에 대한 정보가 검색가능해야 한다.
	- SOA 시스템의 규모가 증가함에 따라 서비스의 중복이 발생할 수 있고, 이를 방지하기 위해서 이미 구현된 서비스가 있는지를 검색할 수 있어야 한다.
	- 검색 내용에는 서비스의 내용과 서비스에 대한 사용방법, 권한, 보안에 대한 정보들이 포함되어야 한다.

## 서비스의 종류
![서비스의 종류]({{site.baseurl}}/img/web-soa-service-type-1.png)
- 비지니스 서비스
	- 일반적으로 SOA에서 정의하는 서비스
	- 비지니스 적인 의미를 가지는 서비스로 SOA의 최소 단위가 된다.
	- 비지니스 로직을 구현한 Task centric service  -> EJB Session Bean
	- 비지니스 데이터를 대표하는 Data centric service -> EJB Entity Bean
- Intermediary
	- Routing
		- ![서비스 종류 Intermediary routing]({{site.baseurl}}/img/web-soa-service-intermediary-routing-1.jpg)
		- 기존 구매 프로세스가 존재하였을 때, 일반 고객과 VIP고객에 대한 프로세스 차별화가 추가되었다.
		- 고객의 요건에 따라 구매 프로세스를 서로 다르게 호출해야 한다. 이런 유형을 Routing 서비스라고 한다.
	- Transformation
		- ![서비스 종류 Intermediary Transformation]({{stie.baseurl}}/img/web-soa-service-intermediary-transformation-1.jpg)
		- 서비스의 인자값의 데이터 타입이 변경되었을 때 기존에 서비스를 호출하던 모든 서비스들은 맞춰 변경해야 한다.
		- 이런 경우 구 데이터 타입을 새로운 데이터 타입으로 변화시켜주는 서비스가 Tranformation 서비스 이다.
	- 이 외에도 새로운 기능을 추가하는 Functional adding 서비스, Facade 기능을 구현한 서비스등을 예로 들수 있다.
		- ![서비스 종류 Intermediary Functional adding]({{stie.baseurl}}/img/web-soa-service-intermediary-functionaladding-1.jpg)
- Process centric
	- ![서비스 종류 Process centric]({{site.baseurl}}/img/web-soa-service-processcentric-1.jpg)
	- 비지니스 서비스를 조합하여 하나의 업무 프로세스를 구현한다.
	- 주로 상태가 있는(Stateful) 것을 구현하는데 이용된다.
- Application 서비스
	- Technical한 서비스로  트랜젝션 서비스, 로깅 서비스 등을 예로 들수 있다.
	- 서비스는 비지니스 의미를 가지는 기능을 모아놓은 컴포넌트라고 설명했었다. 하지만 실제 개발에서는 이러한 테크니컬한 서비스가 나올 수 있으므로 SOA에서 예외적인 서비스이다.
	- 잘 설계된 SOA에는 Application 서비스가 존재하지 않는다.
- Public enterprise
	- 다른 회사나 다른 SOA 시스템으로 제공되는 서비스이다.
	- 다른 서비스에 비해서 외부로 제공되는 서비스인 만큼 성능, 트랜젝션, 보안에 대한 깊은 고려가 필요하다.



# SOA 아키텍쳐 모델
- 서비스의 구성 방법은 기업의 SOA 성숙도와 발전 정도에 따라 단계적으로 적용되어야 한다.
- Application frontend란 서비스들이 사용되어 최종 사용자에게 보이는 곳이다.(ex. 클라이언트, 웹브라우저)
## Fundamental SOA (통합)


# SOA 아케텍쳐 구현시 고려 사항


# SOA 수행 전략




---
## Reference
[서비스 지향 아키텍쳐 (SOA)](https://www.slideshare.net/Byungwook/soa-61487404)  
[What is SOA? How to SOA?](https://bcho.tistory.com/48)  
[서비스 지향 아키텍처](https://ko.wikipedia.org/wiki/%EC%84%9C%EB%B9%84%EC%8A%A4_%EC%A7%80%ED%96%A5_%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98)  