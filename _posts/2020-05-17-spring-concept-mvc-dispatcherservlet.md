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
	1. `Multipart File Resolver` 를 명시하게 되면, `Multipart` 요청일 경우 요청을 `MultipartHttpServletRequest` 로 랩핑해서 이후 처리부터 사용하게 된다. `Multipart` 요청에 대한 처리는 [ Multipart Resolver](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-multipart)
	에서 보다 자세히 확인 할 수 있다.
	1. 요청을 처리할 적절한 핸들러가 발견되면 모델 렌더링을 준비하기 위한 핸들러(preprocessor, postprocessor, controller)와 관련된 체인이 실행된다. 
	`Annotation` 이 설정된 `Controller` 일 경우, `View` 렌더링 대신 `HandlerAdapter` 에서 응답을 만들어 전달해 줄 수 있다.
	1. 모델이 리턴될 경우 `View` 를 렌더링하고, 그렇지 않을 경우 요청이 이미 수행되었을 수 있기때문에 `View` 에 대한 처리를 하지 않는다.
		- 모델이 리턴되지 않았을 경우는 보안상 이슈나, 전처리, 후처리에서 요청을 가로챈 상황이 될 수 있다.
- `WebApplicationContext` 에 선언된 `HandlerExceptionResolver` 빈은 요청을 처리하는 과정에서 발생하는 예외를 해결하기 위해 사용되는데, 예외 햬결은 커스텀하게 관련 로직을 사용자가 정의 할 수 있다.
	- 관련된 더 자세한 내용은 [여기](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-exceptionhandlers)
	 에서 확인 가능하다.
- `Spring` 의 `DispatcherServlet` 은 `Servlet API` 에서 명시된 `last-modification-date` 의 반환을 지원하는데,
 이는 `DispatcherServlet` 이 `HandlerMapping` 을 통해 적절한 핸들러를 찾은 다음 
 해당 핸들러가 `LastModified` 인터페스르를 구현하는지 검사하고 
 구현하고 있다면 `getLastModified(request)` 메소드를 통해 클라이언트에게 반환한다.
- `web.xml` 설정 중 `Servlet` 선언 부분에서 `Servlet` 초기화 파라미터(`init-param`) 을 통해 개별로 구성되는 `DispatcherServlet` 을 추가할 수 있다.

	Parameter|Explaination
	---|---
	`contextClass`|설정하는 `Servlet` 에 의해 로컬로 설정되는 `ConfiguredWebApplicationContext` 의 구현체 클래스로 `XmlWebApplication` 이 기본으로 사용된다.
	`contextConfigLocation`|`Context` 의 인스턴스의 위치를 나타내는 문자열로, 구분자(`,`) 를 통해 하나 이상으로도 설정 가능하다. 두번이상 정의된 빈의 위치의 경우에는 최신위치를 우선해서 사용한다.
	`namespace`|`WebApplicationContext` 의 네임스페이스관련 설정으로 기본으로는 `[servlet-name]-servlet` 이 사용된다.
	`throwExceptionIfNotHandlerFound`|요청을 처리할 핸들러를 찾지 못했을 때 `NoHandlerFoundException` 을 던질디에 대한 여부의 설정으로, `HandlerExceptionResolver`(컨트롤러 사용시 `@ExceptionHandler`) 에서 발생한 예외를 적절한 방법으로 처리할 수 있다. 기본 값은 `false` 로 설정되고 `DispatcherServlet` 은 예외를 발생시키지 않고 응답 상태값을 `404 Not Found` 로만 설정한다. [기본 servlet handler](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-default-servlet-handler) 사용시 처리할 수 없는 요청은 항상 기본 `Servlet` 에 전달되고 `404` 는 발생하지 않는다.
	
## Interception
- 모든 `HandlerMapping` 은 특정 요청에(권한 검사 등) 대해 특정 기능을 수행해야하는 경우에 `handler interceptor` 기능을 제공한다. `Interceptor` 를 구현하기 위해서는 `org.springframework.web.servlet` 패키지의 `HandlerInterceptor` 를 구현해야 하는데 이는 관련처리를 위해 아래와 같은 3가지 메소드를 제공한다.
	- `preHandler(..)` : 요청을 처리할 핸들러 실행 전 호출 된다.
	- `postHandler(..)` : 요청을 처리하는 핸들러 실행 후 호출 된다.
	- `afterCompletion(..)` : 요청 처리가 완료되고 나서 호출 된다. (응답 전송 후)
- `preHandler()` 메소드는 `boolean` 을 리턴하게 되는데, 이후 `execution chain` 을 계속해서 실행 할지에 대한 여부를 결정할 수 있다. `true` 를 리턴하면 계속해서 실행을 수행하고, `false` 를 리턴하면 `DispatcherServlet` 에서는 요청을 모두 처리했다고 가정하고 이후 `execution chain` 은 실행하지 않는다.
- [Interceptors](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-config-interceptors)
에서 구성하는 방법에 대한 설명이 있다. 개별 `HandlerMapping` 구현체에 `setter` 를 통해 직접 등록하는 것도 가능하다.
- 요청을 처리하는 핸들러를 실행하는 `HandlerAdapter` 에서 `postHandler` 호출 전에 응답이 작성되고 커밋되기 때문에 `@ResponseBody` 또는 `ResponseEntity` 관련 메소드에서 공통 처리에 대한 부분이 유용하지 않을 수 있다.
`postHandler()` 을 사용해서 헤더 및 응답을 추가한다던가, 변경은 적용되지 않는다. 해당 동작을 위해서는 `ResponseBodyAdvice` 를 구현하고 [Controller Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-controller-advice)
빈으로 설정하는 방법과 직접 `RequestMappingHandlerAdapter` 를 구성하는 방법이 있다.

## Exception
- 요청 매핑(`HandlerMapping`) 중 에러 및 요청 처리(`HandlerAdapter`, `@Controller`) 중 에러가 발생하면, `DispatcherServlet` 은 체인으로 구성된 `Handler ExceptionRevolver` 빈에게 위임해서 예외 처리 및 오류 응답에 대한 처리를 수행한다.
- 사용가능한 `HandlerExceptionResolver` 의 구현체는 아래와 같다.

	`HandlerExceptionResolver`|Description
	---|---
	`SimpleMappingExceptionResolver`|예외 클래스이름과 에러 페이지의 이름을 매핑한다. 브라우저에서 에러페이지에 대한 처리에 유용하다.
	`DefaultHandlerExceptionResolver`|`Spring MVC` 에서 발생한 예외를 처리하고 `HTTP` 상태코드에 매핑한다. 관련된 다른 구현체로는 `ResponseEntityExceptionHandler` 와 [REST API exception](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-rest-exceptions) 이 있다.
	`ResponseStatusExceptionResolver`|`@ResponseStatus` 를 사용해서 예외를 해결하고, `Annotation` 에 설정된 값으로 `HTTP` 상태코드를 매핑한다.
	`ExceptionHandlerExceptionResolver`|`@Controller` 및 `@ControllerAdvice` 클래스에 선언된 `@ExceptionHandler` 메소드를 호출해서 예외를 처리르한다. [@ExceptionHandler methods](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-exceptionhandler)

### Chain of Resolvers
- `Spring` 설정을 통해 `HandlerExceptionResolver` 관련 설정을 하나 이상 설정할 수 있는데, 각 빈은 `order` 프로퍼티를 통해 우선순위를 설정할 수 있다. `order` 가 높을 수록 해당 `exception resolver` 는 늦게 호출된다.
- `HandlerExceptionResolver` 는 아래와 같은 값을 반환할 수 있다.
	- 에러 페이지를 위한 `ModelAndView`
	- 이미 에러 처리가 된 상태인 경우 비어있는 `ModelAndView`
	- 예외가 처리되지 않은 상태이고 체인으로 연결된 상태에서 다음 `resolver` 에게 넘기고 싶을 경우 `null`
- [Spring MVC Config](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-config) 
는  `Spring MVC` 예외와 `@ResponseStatus`, `@ExceptionHandler` 메소드 에외 처리를 하기위해서는 내장된 기본 `resolver` 를 자동으로 선언해서 사용한다. 관련 설정은 사용자가 커스텀하게 변경 할 수 있다.

### Container Error Page
- 예외가 `HandlerExceptionResolver` 에 의해 처리되지 못하거나, 응답 상태가 에러(4xx, 5xx) 일 경우 `Servlet container` 는 `HTML` 에 기본 에러 페이지를 렌더링 한다. 
기본 에러 페이지에 대한 설정은 `web.xml` 에서 에러 페이지를 매핑을 선언하면 가능하다.

	```xml
	<error-page>
		<location>/error</location>
	</error-page>
	```  
	
- 예외 처리가 계속 전파되어 에러 상태일경우 `Servlet container` 에 설정된 `Error URL`(`/error`) 로 에러를 전송한다. 그리고 에러 경로는 `DispatcherServlet` 에 의해 처리 되기 때문에 `@Controller` 를 통해 에러 경로를 매핑해서 처리 가능하고 `ModelAndView` 반환하거나 `JSON` 응답을 반환 할 수 있다.

	```java
	@RestController
	public class ErrorController {
	
		@RequestMapping(path = "/error")
		public Map<String, Object> handle(HttpServletRequest request) {
			Map<String, Object> map = new HashMap<String, Object>();
			map.put("status", request.getAttribute("javax.servlet.error.status_code"));
			map.put("reason", request.getAttribute("javax.servlet.error.message"));
			return map;
		}
	}
	```  
	
- `Servlet API` 에서는 `Java Config` 를 통해 기본 에러 페이지에 대한 설정을 제공하지 않는다. 하지만 `WebApplicationIntiailizer` 와 최소한의 `web.xml` 을 사용하는 방법으로는 가능하다.
































---
## Reference
[Spring MVC - DispatcherServlet](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-servlet)  