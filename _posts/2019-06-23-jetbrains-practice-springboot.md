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
---  

## 환경
- Intellij
- Java 8
- Spring Boot

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
	- Thymeleaf
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

![그림 9]({{site.baseurl}}/img/jetbrains/practice-springboot-9.png)

- Fix 를 눌러 Artifact 중 exploded 버전을 선택하여 추가한다.

![그림 10]({{site.baseurl}}/img/jetbrains/practice-springboot-10.png)

![그림 11]({{site.baseurl}}/img/jetbrains/practice-springboot-11.png)





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