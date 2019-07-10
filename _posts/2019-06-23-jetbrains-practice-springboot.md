--- 
layout: single
classes: wide
title: "[Jetbrains] Intellij Spring Boot 프로젝트 만들기"
header:
  overlay_image: /img/jetbrains-bg.jpg
excerpt: 'Intellij 에서 Spring Boot 프로젝트를 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Jetbrains
tags:
  - Jetbrains
  - Intellij
  - Java
  - Spring
  - Spring Boot
  - Tomcat
---  

## 환경
- Intellij
- Java 8
- Spring Boot 2.1.6
- Tomcat 8.5

## Spring Boot 프로젝트 생성
- File -> New -> Project...

![그림 1]({{site.baseurl}}/img/jetbrains/practice-springboot-1.png)

- Project SDK 목록에서 사용할 Java 버전을 선택한다.

![그림 2]({{site.baseurl}}/img/jetbrains/practice-springboot-2.png)

- 생성할 프로젝트의 기본적인 설정을 해준다.
	- Type 에서 Gradle, Maven 중 선택가능하다.
	- Packaging 은 이후 프로젝트를 빌드할 때 jar 파일과 war 파일중 선택한다.
		- jar : Spring Boot 에 내장된 톰갯을 사용해서 Stand Alone 형식으로 빌드 및 구동을 한다.
		- war : 외부 톰캣을 사용해서 배포, 빌드, 구동한다.

![그림 3]({{site.baseurl}}/img/jetbrains/practice-springboot-3.png)

- 프로젝트에 필요한 의존성들을 선택해준다.
	- Lombok
	- Web
	- Thymeleaf (x)
	- JPA
	- H2

![그림 4]({{site.baseurl}}/img/jetbrains/practice-springboot-4.png)

- 프로젝트를 생성할 경로를 선택하고 Finish 를 누르면 프로젝트가 생성된다.

![그림 5]({{site.baseurl}}/img/jetbrains/practice-springboot-5.png)

- 프로젝트가 생성되면 아래와 같은 디렉토리와 파일로 구성된다.
	- 메인 클래스 : SpringbootStartApplication
	- 프로퍼티 파일 : application.properties
	- 의존성관리 : pom.xml
	- 기본 Spring 의 디렉토리 구조와 동일하다.

![그림 6]({{site.baseurl}}/img/jetbrains/practice-springboot-6.png)

### 테스트

- 기본적인 테스트를 위해 간단한 Java 파일을 추가한다.
- 구현코드는 아래와 같다.

```java
package com.example.springbootstart.domain;

import lombok.*;

import javax.persistence.Entity;
import javax.persistence.Id;

@Getter
@Setter
@AllArgsConstructor
@ToString
@EqualsAndHashCode
@Entity
public class Device {
    @Id
    private String serial;
    private String name;
    private int price;
}
```  

```java
package com.example.springbootstart.repository;

import com.example.springbootstart.domain.Device;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface DeviceRepository extends JpaRepository<Device, String> {
    List<Device> findByName(String name);
}
```  

- 테스트 코드는 아래와 같다.

```java
package com.example.springbootstart.repository;

import com.example.springbootstart.domain.Device;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class DeviceRepositoryTest {
    @Autowired
    private DeviceRepository deviceRepository;

    @Test
    public void test() {
        Device d = new Device("s1", "n1", 1);
        this.deviceRepository.save(d);

        d = new Device("s2", "n2", 2);
        this.deviceRepository.save(d);

        d = new Device("s3", "n2", 3);
        this.deviceRepository.save(d);

        Assert.assertEquals(new Device("s1", "n1", 1), this.deviceRepository.findById("s1").get());
        Assert.assertEquals(new Device("s2", "n2", 2), this.deviceRepository.findById("s2").get());
        Assert.assertEquals(new Device("s3", "n2", 3), this.deviceRepository.findById("s3").get());

        Assert.assertEquals(2, this.deviceRepository.findByName("n2").size());
    }
}
```  

- 기본적인 Spring Boot 프로젝트 생성은 완료 되었다.

![그림 7]({{site.baseurl}}/img/jetbrains/practice-springboot-7.png)

## 외부 톰캣 연결하기

- Run -> Edit Configurations..

![그림 8]({{site.baseurl}}/img/jetbrains/practice-springboot-8.png)

- `+` -> Tomcat Server -> Local
	- 로컬에 설치된 톰캣이 있어야 하고, Intellij 에 해당 톰캣을 추가한다.

![그림 9]({{site.baseurl}}/img/jetbrains/practice-springboot-9.png)

- Fix 를 눌러 Artifact 중 exploded 버전을 선택하여 추가한다.

![그림 10]({{site.baseurl}}/img/jetbrains/practice-springboot-10.png)

![그림 11]({{site.baseurl}}/img/jetbrains/practice-springboot-11.png)

- File -> Project Structure ... -> Facets 에서 Web Resource Directions 에 있는 경로에 맞게 webapp 디렉토리를 생성해준다.
	- 이 부분은 외부 톰캣을 연결하고 `.jsp` 를 view로 사용하기 위한 절차이다.
	
![그림 12]({{site.baseurl}}/img/jetbrains/practice-springboot-12.png)

- 하위 디렉토리와 테스트 용 `.jsp` 파일을 하나 만들어 준다.

![그림 13]({{site.baseurl}}/img/jetbrains/practice-springboot-13.png)

- hello.jsp 코드

```html
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Hello Spring Boot</title>
</head>
<body>
<h3>Hello Spring Boot</h3>
</body>
</html>
```  

- 아래와 같이 파일을 구성한다.

![그림 14]({{site.baseurl}}/img/jetbrains/practice-springboot-14.png)

- `/WEB-INF/views/` 의 리소스 경로 매핑 설정 `AppConfiguration`
	- 위 부분은 JavaConfig 를 사용하는 방법으로 JavaConfig 를 사용하지 않고 `application.properties` 에 아래 내용을 추가하여 매핑 가능하다.
	
		```properties
		spring.mvc.view.prefix= /WEB-INF/views/
        spring.mvc.view.suffix= .jsp
        spring.thymeleaf.enabled=false	# 타임리프 의존성이 추가되어 있는경우 비활성화
		```  

```java
package com.example.springbootstart;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration
public class AppConfiguration {
    @Bean
    public InternalResourceViewResolver internalResourceViewResolver() {
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("/WEB-INF/views/");
        viewResolver.setSuffix(".jsp");

        return viewResolver;
    }
}
```  

- 테스트 뷰 파일인 `hello.jsp` 의 컨트롤러 `HelloController`

```java
package com.example.springbootstart.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        System.out.println("hello");
        return "hello";
    }
}
```  

- 에코 기능만 있는 레스트 API `RestHelloController`

```java
package com.example.springbootstart.controller;

import org.springframework.web.bind.annotation.*;

@RestController
public class RestHelloController {
    @GetMapping("/echo/{path}")
    public String getEcho(@PathVariable String path) {
        return path;
    }

    @PostMapping("/echo")
    public String postEcho(@RequestBody String request) {
        return request;
    }
}
```  

- 서버를 구동 시키고 `http://localhost:<포트>` 로 접속하면 아래와 같다.

![그림 15]({{site.baseurl}}/img/jetbrains/practice-springboot-15.png)

- `http://localhost:<포트>/hello` 로 접속하면 `HelloController` 를 타고 `hello.jsp` 가 보이는 걸 확인 할 수 있다.

![그림 16]({{site.baseurl}}/img/jetbrains/practice-springboot-16.png)

- `http://localhost:<포트>/echo/<문자열>` 을 통해 `RestHelloController` 의 Get 방식 Echo 동작을 확인 할 수 있다.

![그림 17]({{site.baseurl}}/img/jetbrains/practice-springboot-17.png)

- 포스트 맨에서 `http://localhost:<포트>/echo` 을 통해 `RestHelloController` 의 Post 방식 Echo 동작을 확인 할 수 있다.

![그림 18]({{site.baseurl}}/img/jetbrains/practice-springboot-18.png)




---
## Reference
[Deploy a Spring Boot WAR into a Tomcat Server](https://www.baeldung.com/spring-boot-war-tomcat-deploy)  
[Spring Boot – Deploy WAR file to Tomcat](https://www.mkyong.com/spring-boot/spring-boot-deploy-war-file-to-tomcat/)  
[Spring Boot - Deploy WAR file to External Tomcat](https://www.javaguides.net/2018/09/spring-boot-deploy-war-file-to-external-tomcat.html)  
[[스프링부트] SpringBoot 개발환경 구성 #1 — 첫걸음](https://medium.com/@yongkyu.jang/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-springboot-%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%84%B1-1-%EC%B2%AB%EA%B1%B8%EC%9D%8C-2aa01e808f62)  
[IntelliJ에서 SpringBoot 프로젝트 생성하기](https://jongmin92.github.io/2018/02/04/Spring/springboot-start/)  
[Apache Tomcat + Spring boot + IntelliJ iDEA](https://duongame.tistory.com/481)  
[[Spring Boot #2] 스프링 부트 프로젝트 구조 (Spring Boot Project Structure)](https://engkimbs.tistory.com/750?category=767865)  
[Spring Boot 디렉토리 구조](http://blog.naver.com/PostView.nhn?blogId=islove8587&logNo=220953725926&parentCategoryNo=&categoryNo=44&viewDate=&isShowPopularPosts=true&from=search)  