--- 
layout: single
classes: wide
title: "[Jetbrains] Intellij Spring MVC Maven 프로젝트 만들기"
header:
  overlay_image: /img/jetbrains-bg.jpg
excerpt: 'Intellij 에서 Spring MVC Maven 프로젝트를 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Jetbrains
tags:
  - Jetbrains
  - Intellij
  - Spring MVC
  - Maven
---  

## 환경
	- Intellij
	- Java 8
	- Spring 4
	- Maven
	- Tomcat 8.5

## 프로젝트 생성

- File -> New -> Project .. 

![spring mvc maven 프로젝트 생성 버튼]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-1.png)

- Spring -> Spring MVC

![spring mvc maven spring 선택]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-2.png)

- 프로젝트의 이름과 경로 설정

![spring mvc maven 프로젝트 이름 및 경로]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-3.png)

- 프로젝트 생성 화면

![spring mvc maven 프로젝트 초기 생성 화면]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-3-1.png) 

- Add Framework Support...

![spring mvc maven 프로젝트 프레임워크 추가]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-4.png)

- Maven 추가

![spring mvc maven 프로젝트 maven 추가]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-5.png)

- Project Structure

![spring mvc maven 프로젝트 project structure]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-6.png)

- Spring Framework 라이브러리 추가

![spring mvc maven 프로젝트 spring 라이브러리]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-7.png)

![spring mvc maven 프로젝트 spring 라이브러리 추가 완료]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-8.png)

- Run -> Edit Configurations...

![spring mvc maven 프로젝트 run edit configurations]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-9.png) 

- Tomcat Server 추가

![spring mvc maven 프로젝트 tomcat 서버 추가]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-10.png)

- Tomcat Server 추가 화면 및 Deployment 설정을 위해 fix 클릭

![spring mvc maven 프로젝트 deployment 설정]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-11.png)

![spring mvc maven 프로젝트 deployment 설정 완료]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-12.png)

- 생성된 프로젝트 구조

![spring mvc maven 프로젝트 생성 구조]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-13.png)

- 초기 프로젝트 구동 화면

![spring mvc maven 프로젝트 초기 구동 화면]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-14.png)

## 프로젝트 테스트

![spring mvc maven 프로젝트 프로젝트 테스트 구조]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-15.png)

- web.xml 수정

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

- dispatcher-servlet.xml 수정

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
    <mvc:annotation-driven/>
    <!-- Component 패키지 지정 -->
    <context:component-scan base-package="com"/>

    <!-- View Resolver 를 지정해서 View Object prefix 경로, suffix 확장자를 설정한다. -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```  

- /web/WEB-INF/views/first_view.jsp 추가

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
    <h2>Hello First View</h2>
</body>
</html>
```  

- FirstController 추가

```java
package com.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class FirstController {
    @RequestMapping(value="/first")
    public String firstHandlerMethod() {
        return "first_view";
    }
}

```  

- 서버 구동 결과 화면

![spring mvc maven 프로젝트 테스트 구동 화면]({{site.baseurl}}/img/jetbrains/jetbrains-springmaven-newproject-16.png)

## 추가 사항

```
Error:java: javacTask: source release 8 requires target release 1.8
```  

- 위 에러 발생시 pom.xml 에 아래 내용 추가

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>dontrise</artifactId>
    <version>1.0-SNAPSHOT</version>

	<!-- 추가할 부분 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```  

---
## Reference
[IntelliJ에서 Spring MVC 시작하기 (Maven)](https://cjh5414.github.io/intellij-spring-start/)  
[[SpringMVC] IntelliJ에서 SpringMVC, Tomcat 설정](https://gmlwjd9405.github.io/2018/10/25/intellij-springmvc-tomcat-setting.html)  
[Error javaTask: source release 8 requires target release 1.8](https://stackoverflow.com/questions/29888592/errorjava-javactask-source-release-8-requires-target-release-1-8)  
