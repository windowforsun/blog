--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Integration "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Integration 과 구성요소에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Integration
    - pipe-and-filters
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


#### Message

메타데이터와 함께 결합되어 있는 자바 오브젝트를 위한 포괄적인 래퍼(`generic wrapper`) 이다. 이는 `header`, `payload` 로 구성된다.

- `payload` 는 자바 객체(`POJO`)를 의미한다. 
- `header` 는 `org.springframework.integration.MessageHeaders` 오브젝트로 `Map<String, Object>` 타입의 컬렉션이다. 

![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-1.jpeg)  


#### Message Channel
`pipes-and-filters` 구조에서 `pipe` 역할을 한다. 이는 메시지를 채널로 보내면 소비자가 채널로부터 메시지를 수신하는 구조를 의미한다. 

- `Message Component` 간 메시지 중간 통로 역할로 컴포넌트간 디커플링을 유지할 수 있도록 하며 `Interception`, `Monitoring` 포인트가 된다. 
- `Point-to-Point`, `Publish/Subscribe` 의 2가지 방식을 제공한다. 
- `Point-to-Point` 채널은 하나의 `Consumer` 만 메시지를 받을 수 있다. 
- `Publish/Subscribe` 채널은 `Broadcast` 방식으로 전체 `Consumer` 가 모두 메시지를 받을 수 있다. 

![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-2.jpeg)  

#### Message Endpoint
`pipes-and-filters` 구조에서 `filter` 역할을 한다. 

- 주요 역할은 애플리케이션의 코드를 `Spring Integration` 과 연결해주는 것이다. 
- `Spring MVC` 에서 컨트롤러의 역할과 비슷하며, 사용자는 컨트롤러가 어떠한 구현으로 `HTTP` 요청을 처리하는지 알순 없지만 간편하게 사용하는 것과 같이 `Message Endpoing` 는 메시지를 처리한다. 
- 컨트롤러에 `URL` 에 대한 매핑작업을 하는 것과 같이 `Message Endpoint` 에도 메시지 채널에 매핑한다.
- 그러므로 실질적으로 비지니스를 구현하는 애플리케이션 코드에서는 `Message` 나 `Message Channel` 의 객체를 작접 제어하지 않는게 이상적이다. 

`Spring Integration` 에서 제공하는 `Message Endpoint` 는 아래와 같은 것들이 있다. 

- `Transfromer` : 메시지의 내용 또는 구조를 변환하고, 수정된 메시지를 반환한다. 
  - 문자열을 `Json`, `XML` 형식으로 변환하는 등 다양한 변환에 사용 될 수 있다. 
  - 메시지의 헤더값에 대한 수정, 추가, 삭제 동작도 수행 할 수 있다. 
- `Filter` : 어떤 메시지를 출력 채널(`output channel`)로 전달 할지를 결정한다. 
  - 필터링 역할을 수행하기 위해 특정 페이로드의 내용 혹은 타입, 프로퍼티값 헤더 존재 혹은 다른 조건등으로 체크해서 검사하는 메서드 정의가 필요하다. 
  - 조건을 통과한 경우 메시지를 출력 채널로 전달하고, 아닌 경우에는 예외를 던지거나 제외처리 할 수 있다. 
- `Router` : 어떤 채널이 메시지를 받아야 하는지 결정한다. 
  - 메시지 페이로드 내용 혹은 타입이나 헤더의 메타정보 등을 바탕으로 결정 할 수 있다. 
  - 고정된 출력이 아닌, 동적 채널로 메시지를 전달해야할 때 사용 할 수 있다. 

![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-3.jpeg)

- `Splitter` : 입력 채널로부터 메시지를 받아 메시지를 여러 개로 분할해 각각의 출력 채널로 보내는 역할을 한다. 
  - 일반적으로 메시지가 복합 페이로드로 구성된 경우 각 페이로드로 분할해 타겟이 되는 출력 채널로 보낼 때 사용한다. 
- `Aggregator` : 여러 메시지를 받아 하나의 메시지로 병합하는 역할을 한다. (`Splitter` 와 반대)
  - 메시지가 이동하는 전체 `pipeline` 에서 `Splitter` 뒤에 위치하며, 메시지 소비자 역할도 하는 `downstream` 이다. 
  - `Aggregator` 동작을 위해서는 메시지 상태 관리, 병합에 필요한 전체 메시지가 수신됐는지, 타임아웃 여부 등을 판단해야 하기 때문에 `Splitter` 보다 복잡성이 높다. 
- `Service Activator` : 서비스를 메시징 시스템에 연결하기 위한 엔드포인트이다. 
  - 입력 채널을 설정하고 서비스가 값을 리턴하도록 구현해야 하며 출력 채널도 설정해야 한다.
  - 메시지 페이로드 추출 및 변환 처리를 위해 서비스 객체를 호출한다. 
  - 서비스 객체의 리턴값이 `Message` 타입이 아니라면 `Message` 타입으로 자동 변환도 가능하다. 
  - 최종 리턴된 `Message` 는 출력 채널로 전송되는데, 출력 채널이 설정되지 않았다면 `return address` 에 명시된 체널로 보내는 것도 가능하다. 


![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-4.jpeg)

- `Channel Adapter` : 메시지 채널을 다른 시스템이나 전송 방식과 연결하는 엔드포인트이다. 
  - 아래는 `inbound channel adapter` 가 `source system` 을 메시지 채널에 연결한 예시이다. 

    ![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-5.jpeg)

  - 아래는 `outbound channel adapter` 가 메시지 체널을 `target system` 과 연결한 예시이다. 

    ![그림 1]({{site.baseurl}}/img/spring/practice-spring-integration-6.jpeg)

### Spring Integration Endpoint
`Spring Integration` 은 외부 시스템과 메시징 기반 통신을 위해 다양한 채널 어댑터 및 메시징 게이트웨이를 제공한다. 
`AMQP`, `Kafka`, `Webflux` 등 다양한 지원이 가능한데 각 시스템을 위한 고유한 요구사항 또한 존재한다.  

외부 시스템와 연동을 위해 사용되는 여러 엔드포인트는 종속성 관리를 위해 `Maven BOM` 형태로 제공된다. 

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.integration</groupId>
            <artifactId>spring-integration-bom</artifactId>
            <version>6.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```  

- `Inbound Adapter` : 데이터를 메시징 애플리케이션으로 가져오는 단방향 통신으로 사용된다.
- `Outbound Adapter` : 메시징 애플리케이션에서 데이터를 전송하기 위한 단방향 통신으로 사용된다. 
- `Inbound Gateway` : 외부 시스템이 메시징 애플리케이션을 호출하고 응답 수신이 가능한 양뱡향 통신으로 사용된다. 
- `Output Gateway` : 메시지 애플리케이션이 외부 시스템을 호출하고 결과를 예상하는 양방향 통신으로 사용된다. 

`Spring Integration` 과 연결 가능한 외부 시스템의 종류와 사용가능한 `Adapter`, `Gateway` 에 대한 나열은 아래 링크에서 확인 할 수 있다. 
[Endpoint Quick Reference](https://docs.spring.io/spring-integration/reference/html/endpoint-summary.html#endpoint-summary)




---  
## Reference
[Overview of Spring Integration Framework](https://docs.spring.io/spring-integration/docs/current/reference/html/overview.html)  
[Introduction to Spring Integration](https://www.baeldung.com/spring-integration)  
[Spring Integration](https://spring.io/projects/spring-integration)  
[Integration Endpoints](https://docs.spring.io/spring-integration/reference/html/endpoint-summary.html)  
