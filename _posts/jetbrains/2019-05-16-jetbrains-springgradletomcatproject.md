--- 
layout: single
classes: wide
title: "[Jetbrains] Intellij Spring Gradle 프로젝트 Tomcat 연동"
header:
  overlay_image: /img/jetbrains-bg.jpg
excerpt: 'Intellij 에서 Spring Gradle 에서 Tomcat 연동을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Jetbrains
tags:
  - Jetbrains
  - Intellij
  - Spring MVC
  - Spring
  - Java
  - Gradle
---  

## 환경
- Intellij
- Java 8
- Spring 4
- Gradle
- Tomcat 8.5

## 필요한 선행 작업
- [Intellij Gradle 프로젝트 만들기]({{site.baseurl}}{% link _posts/jetbrains/2019-05-14-jetbrains-gradleproject.md %})
- [Intellij Spring Gradle 프로젝트 만들기]({{site.baseurl}}{% link _posts/jetbrains/2019-05-15-jetbrains-springgradleproject.md %})

## Spring Gradle 프로젝트 Tomcat 연동
- Run -> Edit Configurations..

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-17.png)

- `+` -> Tomcat Server -> Local

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-18.png)

- Tomcat Server 생성 화면
	- Application server 가 없다면 옆에 Configure 를 눌러 로컬에 설치한 Tomcat 을 추가해 준다.
	- 그리고 Deployment 설정을 위해 아래 fix 클릭
	
![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-19.png)

- Deployment 설정 화면

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-20.png)


## Spring Gradle + Tomcat 프로젝트 테스트

- File -> Project Structure -> Artifacts
	- Available Element 에서 Spring 관련 라이브러리들을 블럭지어 오른쪽 클릭하고 `Put into /WEB-INF/lib` 를 눌러준다.
	- 하나씩 더블클릭 해서 넣을 수도 있다.

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-16.png)

- web.xml 설정
	- servlet-mapping - url-pattern 부분을 `/` 로 변경해 준다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```  

- dispatcher-servlet.xml 설정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xmlns:context="http://www.springframework.org/schema/context"
     xmlns:mvc="http://www.springframework.org/schema/mvc"
     xsi:schemaLocation="http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/context
     http://www.springframework.org/schema/context/spring-context.xsd
     http://www.springframework.org/schema/mvc
     http://www.springframework.org/schema/mvc/spring-mvc.xsd">

  <!-- Annotation 활성화 -->
  <mvc:annotation-driven></mvc:annotation-driven>
  <!-- Component 패키지 지정 -->
  <context:component-scan base-package="com.spring"></context:component-scan>

  <!-- view 이름 및 경로 매핑 -->
  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
      <property name="prefix" value="/WEB-INF/views/"></property>
      <property name="suffix" value=".jsp"></property>
  </bean>
</beans>
```  

- view 디렉토리 생성 및 jsp 파일 생성

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-21.png)

```html
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Hello</title>
</head>
<body>
    Hello Spring Gradle Tomcat
</body>
</html>
```  

- Controller 생성
	- HelloController : 웹 페이지(jsp) 관련 Controller
	- RestHelloController : Rest API Controller

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-22.png)

```java
@Controller
public class HelloController {
    @RequestMapping(value = "/")
    public String hello() {
        return "hello";
    }
}
```  

```java
@RestController
@RequestMapping("/resthello")
public class RestHelloController {

    @GetMapping("/{echoString}")
    public String echoGet(@PathVariable String echoString) {
        return echoString;
    }

    @PostMapping
    public String echoPost(@RequestBody String echoString) {
        return echoString;
    }
}
```  

- Tomcat Server 구동

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-23.png)

- Controller 결과

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-24.png)

- RestController 결과

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-25.png)

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-26.png)


---
## Reference
