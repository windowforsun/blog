--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
toc: true
use_math: true
---  

## Spring Boot Web + ReactJS + Gradle
`Spring Boot`, `ReactJS`, `Gradle` 을 사용해서 간단하게 웹애플리케이션을 구성해 본다. 
`Spring Boot` 는 `Back-end` 서버의 역할과 웹서버 역할을 수행하고, 
`ReactJS` 를 기반으로 `Front-end` 스크립트가 작성된다. 
그리고 빌드는 `Gradle` 을 사용하는 구조이다.  

### Spring Boot Web
#### build.gradle
`build.gradle` 파일 내용은 아래와 같다.  

```groovy
plugins {
    id 'java'
    id 'io.spring.dependency-management' version "1.0.7.RELEASE"
    id 'org.springframework.boot' version '2.6.2'
}

group 'com.windowforsun.springreactjs'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

test {
    useJUnitPlatform()
}
```  

아직 `ReactJS` 를 포함한 빌드 스크립트는 작성되지 않은 상태이다. 
이는 추후에 다시 설명하도록 한다.  

#### Application, Controller
`Application` 클래스 내용은 아래와 같다.  

```java
@SpringBootApplication
public class BasicApplication {
    public static void main(String[] args) {
        SpringApplication.run(BasicApplication.class, args);
    }
}
```  

그리고 `/hello` 경로와 매핑되는 `API` 컨트롤러 클래스 파일은 아래와 같다.  

```java
@RestController
public class HelloRestController {

    @GetMapping("/api/hello")
    public List<String> hello() {
        return Arrays.asList("안녕하세요.", "hello", LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
    }
}
```  

#### ReactJS
`ReactJS` 를 사용하기 위해서는 먼저 `NodeJS` 설치가 필요한데, 
사용자 개발환경에 맞는 것으로 
[링크](https://nodejs.org/ko/download/)
에서 설치한다.  

설치가 완료 됐다면, `Spring Boot` 프로젝트 루트 경로에서 아래 명령어로 `frontend` 라는 `ReactJS` 프로젝트를 생성해 준다.  

```bash
$ npx create-react-app frontend
```  

그리고 테스트를 위해서 먼저 `npm start` 로 `ReactJS` 를 실행한다. 
그 후 브라우저를 사용해서 `http://localhost:3000` 로 접속하면 아래와 같은 페이지를 확인할 수 있다.  

![그림 1]({{site.baseurl}}/img/spring/practice-reactjs-gradle-1.png)  

`ReactJS` 는 `Back-end` 서버로 `Spring Boot` 와 연동된다. 
정상적인 통신을 위해 `proxy` 설정이 필요한데, `<projectRoot>/frontend/package.json` 파일에 아래 내용을 추가해 준다.  

```json
{
	...,
	"proxy" : "http://localhost:8080", // Spring Boot WebApplication URL
	...
}

```  

위 설정을 통해 로컬 개발환경에서 `3000` 번 포트로 구동되는 `ReactJS` 와 `8080` 포트로 구동되는 
`Spring Boot` 애플리케이션에서 발생할 수 있는 `CORS` 문제를 회피 할 수 있다.  

다음으로는 `Spring Boot API` 의 응답을 `ReactJS` 에서 받아 페이지에 표시하는 것을 진행한다.  





---
## Reference