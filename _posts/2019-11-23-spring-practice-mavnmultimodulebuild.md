--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Maven Multi Module 구성 과 빌드"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 프로젝트를 Maven Multi Module 을 사용해서 구성하고 빌드를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Maven
    - Docker
---  

# 목표
- Spring 프로젝트를 모듈 단위로 분리해서 구성한다.
- 모듈 단위 분리를 통해 공통적으로 사용하는 부분들은 하나의 공통 모듈로 구성한다.
- 모듈 단위로 분리된 프로젝트를 빌드해서 하나의 애플리케이션을 구성한다.
- Intellij 를 사용해서 프로젝트를 구성한다.

# 방법
- `Gradle` 과 `Maven` 에서는 프로젝트를 모듈단위로 구성할 수 있도록 기능을 제공한다.
- 모듈 단위로 분리해서 프로젝트를 구성하는 것은 중복코드를 줄일수 있다는 부분에서 큰 장점이 될 수 있다.
- 잘못 사용할 경우에는 구성하고 있는 프로젝트에 치명적인 독이 될 수도 있으므로 주의하고, 잘 구성해서 사용해야 한다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-1.png)

- 단순한 예시로 구성한 프로젝트의 구조는 아래와 같다.
	- `root` : 아래 3개의 모둘을 포함하는 프로젝트로 아래 모듈을 연결해주는 역할과 공통적으로 가지는 의존성으로 구성되어 있다.
	- `core` : 공통적으로 사용하는 코드를 포함된 모듈(domain, repository)
	- `firstapp` : `core` 모듈을 사용한 Web Application 으로 Get 관련 API 를 제공한다.
	- `secondapp` : `core` 모듈을 사용한 Web Application 으로 Create, Update 관련 API 를 제공한다.
	
## root
- `root` 프로젝트의 하위에 모듈을 구성하기 전에 프로젝트를 생성한다.
- `File -> New -> Profject...` 를 눌러 Maven 프로젝트를 만든다.

	![그림 2]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-2.png)
	
	![그림 3]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-3.png)
	
	![그림 4]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-4.png)
	
- `pom.xml` 에 하위 모듈들에서 공통으로 사용하는 의존성을 추가해준다.

	```xml	
	<!-- 생략 -->
	
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <groupId>com.windowforsun</groupId>
    <artifactId>root</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    
	<!-- 생략 -->
	
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    
	<!-- 생략 -->
	
	```  
	
	- 프로젝트의 `packaging` 은 `pom` 이다.
	
## core
- 프로젝트에서 오른쪽 클릭을 한 후 `core` 모듈을 추가해준다.

	
## firstapp
- 프로젝트에서 오른쪽 클릭을 한 후 `firstapp` 모듈을 추가해준다.

	![그림 5]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-5.png)
	
	![그림 6]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-6.png)
	
	여기 아래부터 `core` 모듈로 다시 찍어서 `core` 모듈 추가하는 방법으로 올리고 `firstapp, secondapp` 은 생략하기
	![그림 7]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-7.png)
	
	![그림 8]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-8.png)
	
- 아래처럼 `root` 프로젝트 하위에 `firstapp` 모듈이 추가 된것을 확인 할 수 있다.

	![그림 9]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-9.png)


	

	
---
## Reference
[]()