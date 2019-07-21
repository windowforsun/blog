--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Spring Profile 환경 분리 및 관리"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Spring Profile 을 이용해서 환경을 분리하고 관리해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Spring Profile
    - Profile
---  

# 목표
- Spring 및 Spring Boot 프로젝트의 환경을 분리하여 관리한다.
- Spring Profile 을 이용해 분리된 환경을 관리한다.

# 방법
- `application-{env}.profile` 또는 `application.yml` 을 통해 각 환경에 따른 설정을 분리한다.
- 프로젝트의 설정클래스에서는 `@Profile` 을 통해 각 환경에 따른 설정클래스를 분리한다.
- Junit 테스트에서는 `@ActiveProfiles` 을 사용해 원하는 환경설정으로 테스트를 진행한다.
- 애플리케이션 빌드 및 배포 시에도 각 환경에 맞게 패키징 한다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/practice-practice-springbootspringprofile-1.png)

## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-spring-profile</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```  

- [Maven Profile]({{site.baseurl}}{% link _posts/2019-07-03-spring-practice-mvnprofilemanagment.md %}) 에서는 pom.xml 에 환경에 따라 리소스 경로를 다르게 설정하였지만 Spring Profile 에서는 좀 더 편하게 사용가능하다.

## Spring Profile 분리
- Properties 를 사용할 경우 `application-<env>.profiles` 의 파일이름으로 각 환경 분리가 가능하다.
	- `application.propereties`
	
		```properties
		spring.profiles.active=local
		```  		
		
	- `application-dev.properties`
	
		```properties
		spring.profiles=dev
        property.a=6
        property.b=2
        property.result=4
		```  
	
	- `application-local.properties`
	
		```properties
		spring.profiles=local
        property.a=6
        property.b=2
        property.result=8
		```  
		
	- `application-prod.properties`
	
		```properties
		spring.profiles=prod
        property.a=6
        property.b=2
        property.result=3
		```  
		
	- `application-qa.properties`
	
		```properties
		spring.profiles=qa
        property.a=6
        property.b=2
        property.result=12
		```  
		
- YAML(YML) 을 사용할 경우에는 `application.properties` 의 파일에 모두 작성할 수 있다.

	```yaml
	spring:
      profiles:
        active: local
      
    ---
    spring:
         profiles: local
    property:
      a: 6
      b: 2
      result: 8
    
    ---
    spring:
      profiles: dev
    property:
      a: 6
      b: 2
      result: 4
      
    ---
    spring:
      profiles: qa
    property:
      a: 6
      b: 2
      result: 12
    
    ---
    spring:
      profiles: prod
    property:
      a: 6
      b: 2
      result: 3
	```   
	
	- `---` 로 Profile 설정을 분리한다.
	
- 기본 Profile 설정은 모두 local 이다.
- Properties 는 `application.properties`, YAML 은 맨 상단의 설정은 모든 Profile 에서 포함된다.

## 서비스 구현 코드
- 간단한 서비스를 통해 분리된 환경에 따라 서비스 처리 또한 분리시켜 본다.

```java
public interface Calculate {
    int calculate(int a, int b);
}
```  

```java
public class DivideCalculateImpl implements Calculate {
    @Override
    public int calculate(int a, int b) {
        return a / b;
    }
}
```  

```java
public class MinusCalculateImpl implements Calculate {
    @Override
    public int calculate(int a, int b) {
        return a - b;
    }
}
```  

```java
public class MultiplyCalculateImpl implements Calculate{
    @Override
    public int calculate(int a, int b) {
        return a * b;
    }
}
```  

```java
public class PlusCalculateImpl implements Calculate {
    @Override
    public int calculate(int a, int b) {
        return a + b;
    }
}
```  

## 설정 클래스 파일
- `@Profile` Annotation 을 통해 각 환경에 맞는 서비스의 빈설정을 작성해 준다.

```java
@Configuration
@Profile("dev")
public class DevConfiguration {
    @Bean
    public Calculate calculate() {
        return new MinusCalculateImpl();
    }
}
```  

```java
@Configuration
@Profile("local")
public class LocalConfiguration {
    @Bean
    public Calculate calculate() {
        return new PlusCalculateImpl();
    }
}
```  

```java
@Configuration
@Profile("prod")
public class ProdConfiguration {
    @Bean
    public Calculate calculate() {
        return new DivideCalculateImpl();
    }
}
```  

```java
@Configuration
@Profile("qa")
public class QaConfiguration {
    @Bean
    public Calculate calculate() {
        return new MultiplyCalculateImpl();
    }
}
```  

## 테스트 코드

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@ActiveProfiles("local")
public class CalculateTest {
    @Autowired
    private Calculate calculate;
    @Value("${property.a}")
    private int a;
    @Value("${property.b}")
    private int b;
    @Value("${property.result}")
    private int result;

    @Test
    public void testCalculate() {
        System.out.println(this.calculate.calculate(this.a, this.b));
        Assert.assertEquals(this.result, this.calculate.calculate(this.a, this.b));
    }
}
```  

- 각 환경의 property 에서 a, b 로 환경설정에 따라 연산된 result 값이 동일하게 계산되는 것으로, 테스트시에 필요한 환경이 잘 적용되었음을 확인할 수 있다.
- `@ActiveProfiles` Annotation 의 값에 `local` 이 있기 때문에 위의 코드는 Profile 이 `local` 인 환경으로 실행되고 테스트가 진행된다.
	- `@ActiveProfiles` Annotation 을 선언하지 않아도 기본 Profile 인 `local` 로 실행된다.
- Junit 테스트에서는 `@ActiveProfiles` 의 값에 `dev`, `qa`, `prod` 등을 넣어 각 환경에 따른 테스트를 수행할 수 있다.

## 빌드
- Intellij 에서는 `Run -> Edit Configurations... -> Configuration -> VM Options` 의 항목에 아래 코드를 추가해준다.
	- `-Dspring.profiles.active={profile}`
- Jenkins 에서 빌드 및 배포시 ...

	
---
## Reference
[Properties and Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-properties-and-configuration.html)   
[Spring Boot Jenkins CD/CI Change profile before deployment](https://stackoverflow.com/questions/54517734/spring-boot-jenkins-cd-ci-change-profile-before-deployment)   
[How do I activate a Spring Boot profile when running from IntelliJ?](https://stackoverflow.com/questions/39738901/how-do-i-activate-a-spring-boot-profile-when-running-from-intellij)   
[Spring Profiles or Maven Profiles?](https://dzone.com/articles/spring-profiles-or-maven)   
[Spring boot 설정 파일 yaml 사용법 (설정 파일을 읽어서 bean으로 필요할 때 사용하는 방법)](https://jeong-pro.tistory.com/159)   
[Spring Boot Profile 설정](https://dhsim86.github.io/web/2017/03/28/spring_boot_profile-post.html)   