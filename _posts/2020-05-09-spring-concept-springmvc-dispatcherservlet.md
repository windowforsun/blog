--- 
layout: single
classes: wide
title: "[Spring MVC] DispatcherServlet"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Concept
  - Spring MVC
  - DispatcherServlet
toc: true
---  

## DispatcherServlet
- `Spring MVC` 는 많은 다른 웹프레임워크들과 같이, `front controller` 패턴을 기반하며 중앙 `Servlet` 를 두는 구조이다.
- 중앙 `Servlet` 을 `DispatcherServlet` 이라고 하며 요청 처리를 위한 공유 알고리즘을 제공하며 실제 동작은 구성된  컴포넌트들에게로 위임된다.
- 이러한 방식의 모델은 유연하면서 다양한 처리작업을 지원 할 수 있다.
- `DispatcherServelt`, `Servlet` 을 사용하기 위해서는 `Java Configuration` 혹은 `web.xml` 에 선언하고 매핑하는 작업이 필요하다.
- `DispatcherSevlet` 은 Spring 설정을 사용해서 `request mapping`, `view resolution`, `exception handling` 등 과같은 처리를 위함할 컴포넌트를 찾는다.
- 아래는 `Java Configuration` 을 사용해서 `Servlet Container` 에 의해 자동으로 탐색 될 수 있도록 `DispatcherServlet` 을 초기화하는 설정의 예이다.

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
	
	```xml
	<web-app>
    
        <listener>
            <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
    
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app-context.xml</param-value>
        </context-param>
    
        <servlet>
            <servlet-name>app</servlet-name>
            <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
            <init-param>
                <param-name>contextConfigLocation</param-name>
                <param-value></param-value>
            </init-param>
            <load-on-startup>1</load-on-startup>
        </servlet>
    
        <servlet-mapping>
            <servlet-name>app</servlet-name>
            <url-pattern>/app/*</url-pattern>
        </servlet-mapping>
    
    </web-app>
	```  
	
	- `ServletContext` API 를 직접 사용하는 방법 외에도 `AbstractAnnotationConfigDispatcherServletInitializer` 를 상속해 관련 메소드를 오버라이드하는 방법이 있다.

> `Spring Boot` 는 다른 방식을 통해 초기화를 수행한다. 
`Servlet container` 라이프 사이클에 후킹하는 방법 대신 Spring 설정을 사용해서 내장 된 `Servlet container` 를 `bootstrap` 한다. 
그리고 `Filter` 와 `Servlet` 의 선언은 Spring 설정에 의해 `Servlet container` 에 등록 된다.


### Context Hierarchy

- `WebApplicationContext` 는 `ServletContext`, `ServletContext` 와 연관된 `Servlet` 에 연결이 있다.

---
## Reference
[Spring Web MVC DispatcherServlet](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-servlet)  
