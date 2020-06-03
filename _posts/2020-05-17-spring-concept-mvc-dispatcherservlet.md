--- 
layout: single
classes: wide
title: "[Spring 개념] Spring MVC DispatcherServlet"
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
    - Spring MVC
    - DispatcherServlet
toc: true
use_math: true
---  

# DispatcherServlet
- `Spring MVC` 는 많은 웹 프레임워크와 마찬가지로 `front controller pattern` 을 중앙 `Servlet` 인 `DispatcherServlet` 을 통해 제공하는데, 이는 요청처리에 대한 공유 알고리즘을 제공하고 실제 요청처리는 설정된 컴포넌트들에게 위임되어 실행된다. 또한 이러한 모델은 보다 유연하면서 다양한 워크플로우를 제공한다.
- `DispatcherServlet` 은 다른 `Servlet` 들과 마찬가지로 `Java Config` 혹은 `web.xml` 통해 서블릿을 설정하고 매핑해야 한다. 그리고 `DispatcherServlet` 은 `Spring Config` 를 사용해서 요청 매핑, 뷰, 예외처리 등에 대한 [설정](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-servlet-special-bean-types)이 가능하다. 
- 아래는 `Java Config` 를 사용해서 `Servlet Container` 에 의해 자동으로 감지 될 수 있도록 `DispatcherServlet` 에 대한 등록과 초기화하는 예제이다. 
	
	```java
	public class MyWebApplicationInitializer implements WebApplicationInitializer {
	
		@Override
		public void onStartup(ServletContext servletCxt) {
	
			// Load Spring web application configuration
			AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
			ac.register(AppConfig.class);
			ac.refresh();
	
			// Create and register the DispatcherServlet
			DispatcherServlet servlet = new DispatcherServlet(ac);
			ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
			registration.setLoadOnStartup(1);
			registration.addMapping("/app/*");
		}
	}
	```  
	
	- `Spring Boot` 는 `Servlet Container` 의 라이프사이클에 후킹하는 방법이 아닌, `Spring Config` 를 내장 `Servlett Container` 에 부트스트랩 하는 방식을 사용한다. `Filter` 와 `Servlet` 은 `Spring Config` 에 의해 탐지되어 `Servlet Container` 에 등록된다.
	
	
## Context Hierarchy
- `DispatcherServlet` 은 자체 구성에 `WebApplicationContext` 가 있어야 하는데, 
`WebApplicationContext` 는 `ServletContext` 와 연결되어 있고 `Servlet` 과 관련이 있기 때문이다. 
그리고 애플리케이션에서 `RequestContextUtil` 의 정적 메소드를 사용해서 `WebApplicationContext` 에 대한 접근이 필요한 경우 조회 할 수 있도록 `ServletContext` 와 바인딩 돼있다.
- 주로 `WebApplicationContext` 는 하나만 구성하더라도 충분하지만, 
필요한 경우 `root` `WebApplicationContext` 를 다수개의 자식 `WebApplicationContext` 를 가지는 `DispatcherServlet` 이 공유하는 계층적 구조로도 구성할 수도 있다.
- `root` `WebApplicationContext` 는 하위의 다중 `Servlet` 들과 공유해야 하는 `data repositories`, `business servies` 관련 빈을 포함하게 된다. 
이를 통해 하위 `WebApplicationContext` 에서는 효율적으로 상속구조를 통해 관련 빈들을 재정의 할 수 있고, 하위 `Servlet` 에서만 필요한 빈도 포함한다. 


![그림 1]({{site.baseurl}}/img/spring/concept-mvc-dispatcherservlet-1.png)

- `WebApplicationContext` 을 계층적으로 설정하는 예는 아래와 같다.

	```java
	public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
	
		@Override
		protected Class<?>[] getRootConfigClasses() {
			return new Class<?>[] { RootConfig.class };
		}
	
		@Override
		protected Class<?>[] getServletConfigClasses() {
			return new Class<?>[] { App1Config.class };
		}
	
		@Override
		protected String[] getServletMappings() {
			return new String[] { "/app1/*" };
		}
	}
	```  
	
	- 만약 계층적 구조가 필요하지 않다면, `getRootConfigClasses()` 에서 모든 설정을 반환하고 `getServletConfigClasses()` 에서는 `null` 을 반환한다.


## Special Bean Types
- `DispatcherServlet` 은 요청처리와 응답에 대한 적절한 뷰를 처리하기위해 `Special Bean` 을 사용하는데,
`Special Bean` 은 `Spring` 에서 관리하는 정해진 규격을 구현하는 객체 인스턴스를 의미한다. 
이런 객체는 기본적으로 제공하는 구현체가 있지만, 필요에 따라 커스텀하게 프로퍼티를 설정하거나 상속을 통해 확장도 가능하다.

Bean Type | Explanation
---|---
`HandlerMapping` | `Interceptor` 의 전처리, 후처리를 위해 요청을 매핑한다. 매핑에 대한 세부 구현은 `HandlerMapping` 구현체 마다 다르다. 주요 2가지 구현체인 `RequestMappingHandlerMapping`(`@RequestMapping` 지원을 위한), `SimpleUserHandlerMapping`(핸들러 URI 경로 패턴의 명시적인 등록을 유지) 가 있다.
`HandlerAdapter` | 핸들러를 어떻게 실제로 호출할지에 대한 부분과는 관련없이 `DispatcherServlet` 이 요청에 매핑된 핸들러를 호출하도록 돕는다. 예를들어 `Annotation` 으로 설정된 `Controller` 를 호출할 때는 `Annotation` 대한 확인 작업이 필요하다. `HandlerAdapter` 는 이런 확인 작업으로 부터 `DispatcherServlet` 대신 역할을 수행한다.
`HandlerExceptionResolver` | 발생된 예외를 해결하는 역할로 매핑된 하위 구현체를 통해 처리한다. [구현체](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-exceptionhandlers)
`ViewResolver` | 핸들러에서 실제 `View` 단으로 반환된 문자열 값을 처리해 응답으로 렌더링 한다. [View Resolution](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-viewresolver), [View Technologies](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-view)
`LocaleResolver`, `LocaleContextResolver` | 국제화에 제공되는 처리로 클라이언트에 해당되는 타임존을 사용할 수 있도록 한다. [Locale](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-localeresolver)
`ThemeResolver` | 개인화된 레이아웃을 제공하는 처리로, 웹 애플리케이션에서 사용할 수 있는 테마에 대한 처리가 가능하다. [Theme](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-themeresolver)
`MultiplepartResolver` | `multi-part` 파싱 라이브러리를 사용한 `multi-part` 요청에 대한 파싱의 추상화이다. [Multipart Resolver](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-multipart)
`FlashMapManager` | 라다이렉션을 통한 요청에서 다른 요청으로 속성을 `FlashMap` 의 저장과 검색을 통해 전달 할 수 있다. [Flash Attributes](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-flash-attributes)


## Web MVC Config
- 애플리케이션은 요청처리에 요구되는 `Special Bean` 들을 `WebApplicationContext` 에 빈을 사용해서 설정할 수 있다. 
`DispatcherServlet` 은 `WebApplicationContext` 를 통해 이런 `Special Bean` 에 대한 확인 작업을 수행한다. 
그리고 매칭되는 빈들이 없다면 [DispatcherServlet.propergies](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/resources/org/springframework/web/servlet/DispatcherServlet.properties)
 에 설정된 기본 구현체로 구성한다.
- 보편적인 [MVC Config](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-config)
가 가장 좋은 시작점으로, 요구되는 것들을 `Java Config` 혹은 `XML` 을 사용해서 빈으로 설정하고 높은 수준의 커스텀 설정인 콜백 API 등을 제공한다.
- 추가적으로 `Spring Boot` 은 `Spring MVC` 구성을 위해 `MVC Java Config` 에 의존하는 다양하고 편리한 옵션을 제공한다. 


## Servlet Config
- `Servlet 3.0` 이상의 환경부터 `Servlet Container` 에 대해 `Java Config` 혹은 `web.xml` 과 함께 사용해서 설정이 가능하다.

	```java
	import org.springframework.web.WebApplicationInitializer;
	
	public class MyWebApplicationInitializer implements WebApplicationInitializer {
	
		@Override
		public void onStartup(ServletContext container) {
			XmlWebApplicationContext appContext = new XmlWebApplicationContext();
			appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
	
			ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
			registration.setLoadOnStartup(1);
			registration.addMapping("/");
		}
	}
	```  
	
	- `WebApplicationInitializer` 는 `Spring MVC` 가 제공하는 인터페이스로, 
	이는 구현체가 탐지되어 `Servlet 3 Container` 를 초기화하는 과정에서 자동으로 사용되는것을 보장한다. 
	`WebApplicationInitializer` 의 추상 구현체인 `AbstractDispatcherServletInitializer` 는 `Servlet` 매핑과 `DispatcherServlet` 의 구성 위치에 대한 설정을 명시해서 메소드 오버라이딩을 통해 보다 쉽게 등록 가능하다.
- `Java` 기반 `Spring Config` 를 사용하는 애플리케이션에 권장되는 `AbstractDispatcherServletInitializer` 관련 설정은 아래와 같다.

	```java
	public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
	
		@Override
		protected Class<?>[] getRootConfigClasses() {
			return null;
		}
	
		@Override
		protected Class<?>[] getServletConfigClasses() {
			return new Class<?>[] { MyWebConfig.class };
		}
	
		@Override
		protected String[] getServletMappings() {
			return new String[] { "/" };
		}
	}
	```  
	
- `XML` 기반 `Spring Config` 의 예는 아래와 같다.

	```java
	public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {
	
		@Override
		protected WebApplicationContext createRootApplicationContext() {
			return null;
		}
	
		@Override
		protected WebApplicationContext createServletApplicationContext() {
			XmlWebApplicationContext cxt = new XmlWebApplicationContext();
			cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
			return cxt;
		}
	
		@Override
		protected String[] getServletMappings() {
			return new String[] { "/" };
		}
	}
	```  
	
- 아래와 같이 `AbstractDispatcherSerlvetIniatilizer` 는  `Filter` 인스턴스를 추가하고 `DispatcherServlet` 에 자동으로 매핑하는 편리한 방법을 제공한다. 

	```java
	public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {
	
		// ...
	
		@Override
		protected Filter[] getServletFilters() {
			return new Filter[] {
				new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
		}
	}
	```  
	
	- `Filter` 는 구현 클래스 타입을 사용해서 자동으로 `DispatcherServlet` 에 추가된다.
	- `AbstractDispatcherServletInitializer` 의 `isAsyncSupported` 메소드는 `DispatcherServlet` 과 매핑된 `Filter` 에서 비동기를 활성화 할 수 있도록 단일 장소를 제공한다.
	`isAsyncSupported` 의 기본값은 `true` 이다.
	
	
## Processing
- `DispatcherServlet` 은 아래와 같은 흐름으로 요청을 처리한다.
	1. `WebApplicationContext` 는 `Controller` 와 다른 요소들이 사용할 수 있는 속성들을 요청에서 검색하고 바인딩 된다.
	기본적인 속성 연결은 `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` 키의 값을 사용한다.
	1. `Locale Resolver` 는 프로세스의 요소들이 요청을 처리할 때 사용할 `Locale` 해결할 수 있도록 요청에 바인딩 된다.
	`Locale` 관련 처리가 필요하지 않다면, `Locale Resolver` 또한 필요하지 않다.
	1. `Theme Resolver` 는 `View` 요소들이 사용할 테마를 결정할 수 있도록 요청과 연결되는데, 테마를 사용하지 않을 경우 무시해도 된다.


---
## Reference
[Spring MVC - DispatcherServlet](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-servlet)  