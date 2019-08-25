--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Validation"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Validation 을 통해 효율적인 유효성 검사를 수행하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Validation
---  

# 목표
- Spring Validation 에 대해 알아본다.
- Spring Validation Annotation 을 이용해서 유효성 검사를 수행한다.
- Custom Validation 을 만들어 유효성 검사를 추가한다.

# 방법
## 유효성 검사란
- API 요청의 요소들이 특정한 데이터 타입과, 특정 도메인의 제약사항을 만족하는지 검사한다.
- 제약 사항을 만족하지 않을 경우, 아래와 같은 Error 를 응답한다.
	- 유효성 검사의 Error 를 나타내는 메시지, 어느 Filed 인지, 올바른 형태는 무엇인지
	- HTTP 상태 코드
	- Error 응답에 중요한 정보를 포함해서는 안된다.
- 유효성 검사 에러에 권장하는 HTTP 상태코드는 400(Bad Request) 이다.

## Validation Annotation
기본적으로 아래와 같은 Annotation 을 통해 유효성 검사를 수행할 수 있다.

- DecimalMax
- DecimalMin
- Digits
- Email
- Future
- FutureOrPresent
- Max
- Min
- Negative
- NegativeOrZero
- NotBlank
- NotEmpty
- NotNull
- Null
- Past
- PastOrPresent
- Pattern
- Positive
- PositiveOrZero
- Size

## 의존성
- `spring-boot-starter-web` 의존성이 포함되어 있다면, `org.hibernate.validator` 가 자식으로 포함되기 때문에 바로 사용가능하다.
- 아래와 같이 Validation 관련 의존성을 추가할 수 있다.

	```java
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-validation</artifactId>
	</dependency>		
	```  
	
	```java
	<dependency> 
		<groupId>org.hibernate</groupId> 
		<artifactId>hibernate-validator</artifactId> 
		<version>4.3.2.Final</version> 
	</dependency>
	```

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-1.png)

## pom.xml

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-redis</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-validation</artifactId>
	</dependency>

	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<optional>true</optional>
	</dependency>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<version>RELEASE</version>
	</dependency>
	<!-- Embedded Redis -->
	<dependency>
		<groupId>it.ozimov</groupId>
		<artifactId>embedded-redis</artifactId>
		<version>0.7.2</version>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```  

## Config
- EmbeddedRedisConfig

	```java
	@Configuration
    public class EmbeddedRedisConfig {
        private RedisServer redisServer;
        @Value("${spring.redis.port}")
        private int port;
    
        @PostConstruct
        public void startRedisServer() throws IOException {
            this.redisServer = new RedisServer(this.port);
            this.redisServer.start();
        }
    
        @PreDestroy
        public void stopRedisServer() {
            if(this.redisServer != null) {
                this.redisServer.stop();
            }
        }
    }
	```  
	


---
## Reference
[Spring Boot CRUD REST APIs Validation Example](https://www.javaguides.net/2018/09/spring-boot-crud-rest-apis-validation-example.html)   
[Implementing Validation for RESTful Services with Spring Boot](https://www.springboottutorial.com/spring-boot-validation-for-rest-services)   
