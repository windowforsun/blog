--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Maven Profile 환경 분리 및 관리"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 Maven Profile 을 이용해서 환경을 분리하고 관리해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Maven
    - Profile
---  

# 목표
- Spring 및 Spring Boot 프로젝트의 환경을 분리하여 관리한다.
- Maven Profile 을 통해 분리된 환경을 관리한다.

# 방법
- `application.properties` 또는 `application.yml` 을 사용해서 각 설정을 환경에 따라 분리한다.
- Spring 프로젝트에서는 `@Profile` 을 통해 각 환경에 따른 설정을 해준다.
- 애플리케이션 빌드 및 배포 시에도 각 환경에 맞게 패키징 한다.

# 예제

- 완성된 프로젝트 구조는 아래와 같다.

```
src
	main
		java
			com.example.demo
				config
					DevConfiguration.java
					LocalConfiguration.java
					ProdConfiguration.java
					QaConfiguration.java
				service
					Calculate.java
					DivideCalculateImpl.java
					MinusCalculateImpl.java
					MultiplyCalculateImpl.java
					PlusCalculateImpl.java
				SpringBootMavenProfileBasicApplication.java
			resources
				dev
					application.properties
				local
					application.properties
				prod
					application.properties
				qa
					application.properties
	test
		java
			com.example.demo
				service
					CalculateTest.java
```  

- pom.xml 은 아래와 같다.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
         
<!-- 생략 -->

	<!-- 각 환경 local, dev, qa, prod 에 대해 기술한다. -->
    <profiles>
        <profile>
            <id>local</id>
            <!-- 기본으로 설정되는 Profile -->
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <!-- Properties 파일 경로에 사용될 값 -->
            <properties>
                <environment>local</environment>
            </properties>
        </profile>
        <profile>
            <id>dev</id>
            <properties>
                <environment>dev</environment>
            </properties>
        </profile>
        <profile>
            <id>qa</id>
            <properties>
                <environment>qa</environment>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <environment>prod</environment>
            </properties>
        </profile>
    </profiles>

    <dependencies>
    	<!-- 생략 -->
    </dependencies>

    <build>
        <plugins>
        	<!-- 생략 -->
        </plugins>
        <!-- 각 환경에 따른 Properties 파일의 경로를 설정한다. -->
        <resources>
            <resource>
                <directory>src/main/resources/${environment}</directory>
            </resource>
        </resources>
    </build>
</project>
```  

- `${environment}/application.properties` 파일은 아래와 같다.

```
spring.profiles.active=dev
property.a=6
property.b=2
property.result=4
```  

```
spring.profiles.active=local
property.a=6
property.b=2
property.result=8
```  

```
spring.profiles.active=prod
property.a=6
property.b=2
property.result=3
```  

```
spring.profiles.active=qa
property.a=6
property.b=2
property.result=12
```  

- 간단한 서비스 구현 코드는 아래와 같다.

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

- `@Profile` Annotation 을 사용해서 각 환경에 따른 설정 분리는 아래와 같다.

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

- 사용하고 있는 Intellij IDEA 에서는 아래와 같이 쉽게 Profiles 값을 변경할 수 있다.

![그림 1]({{site.baseurl}}/img/spring/practice-mavenprofilemanage-1.png)

- Junit 테스트 코드는 아래와 같다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
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
        Assert.assertEquals(this.result, this.calculate.calculate(this.a, this.b));
    }
}
```  

- `Properties` 파일에 `Calculate` 서비스의 인자값 a, b 그리고 리턴값 `result` 가 있다.
- 각 환경에 따라 `Calcaulte` 서비스에 할당되는 구현 클래스가 달라지고 이를 실제로 계산했을때 `Properties` 의 값과 맞다면 현재 자신의 환경이 잘 적용 된 것을 확인 할 수 있다.
- Jenkins 를 통해 빌드 및 배포 또는 패키징 시에는 `mvn package -P <Profile>` 를 통해 원하는 `Profiles`  적용할 수 있다.

```
## 기본 profile로 local 적용
mvn clean package
  
## dev deploy
mvn clean package -P dev
  
## qa deploy
mvn clean package -P qa

## prod deploy
mvn clean package -P prod
```  
	
---
## Reference
[Spring Profiles](https://www.baeldung.com/spring-profiles)  
[maven profile 을 이용하여 운영 환경에 맞게 패키징 하기](https://www.lesstif.com/pages/viewpage.action?pageId=14090588)  
[Spring Boot Profile 설정](https://dhsim86.github.io/web/2017/03/28/spring_boot_profile-post.html)  
[Maven Profile 를 통해 설정 관리하기](https://dreambringer.tistory.com/15)  
[[SPRING] Maven 프로파일로 스프링 활성 프로파일을 설정하는 방법](https://cnpnote.tistory.com/entry/SPRING-Maven-%ED%94%84%EB%A1%9C%ED%8C%8C%EC%9D%BC%EB%A1%9C-%EC%8A%A4%ED%94%84%EB%A7%81-%ED%99%9C%EC%84%B1-%ED%94%84%EB%A1%9C%ED%8C%8C%EC%9D%BC%EC%9D%84-%EC%84%A4%EC%A0%95%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95)  
[Spring Boot에서 배포환경 나누기](https://yookeun.github.io/java/2017/04/08/springboot-deploy/)  
[Spring Boot maven profile 적용](https://kktaehun.tistory.com/1)  
