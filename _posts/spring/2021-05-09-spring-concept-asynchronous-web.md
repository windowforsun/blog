--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Asynchronous Web"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 비동기 프로그래밍을 할때 사용 할 수 있는 API 와 기능에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Async
    - ThreadPoolTaskExecutor
    - AsyncResult
    - ListenableFuture
toc: true
use_math: true
---  

## Spring Asynchronous Web
[Java Asynchronous Programming]({{site.baseurl}}{% link _posts/java/2021-05-02-java-concept-asynchronous-programming.md %})
에서는 `Java` 에서 비동기 프로그래밍을 할때 사용할 수 있는 `API` 에 대해서 알아보았고, 
[]()
에서는 `Spring` 에서 지원해주는 비동기 기능에 대해서 알아 보았다. 
`Spring` 을 사용하면 다양한 종류의 애플리케이션을 개발할 수 있지만, 
대부분의 경우 `Web` 애플리케이션을 구현을 위해 사용된다. 
이번 포스트에서는 `Spring Web` 애플리케이션을 개발할때 지원하는 비동기 방식에 대해서 알아본다.  

<!--
DeferredResult|Spring 3.2
AsyncRestTemplate|Spring 4.0
ResponseBodyEmitter|Spring 4.2
-->

`Spring Boot` 은 기본으로 컴파일된 `Web Server` 를 제공한다. 
작성한 애플리케이션 코드는 자동으로 `Web Server` 에 의해 동작 되기 때문에 별도로 빌드 결과물을 `Web Server` 에 올리는 작업은 하지 않아도 웹 애플리케이션을 구동 할 수 있다. 
그리고 `Spring Boot` 가 기본으로 제공하는 `Web Server` 는 `Tomcat` 으로 `spring-boot-starter-web` 의존성에 `spring-boot-starter-tomcat` 의존성이 포함돼 있는 것을 확인 할 수 있다.  

`Tomcat` 에서 웹 통신은 `Servlet API` 를 사용해서 이뤄지는데, `3.0` 버전을 기점으로 비동기 지원을 시작했다. 
- `Servlet 3.0`
  - 비동기 서블릿 지원
  - `HTTP connection` 은 이미 `NIO` 를 통해 구현됨
  - `Servlet` 요청 읽기, 응답 쓰기는 `Blocking`
  - 비동기 작업 시작 즉시 `Servlet Thread` 반납
  - 비동기 작업 완료시 `Servlet Thread` 재 할당 후 응답
  - 비동기 서블릿 컨텍스트 이용(`AsyncContext`)
- `Servlet 3.1`
  - `Non-Blocking` 서블릿 요청, 응답 처리 가능
  - `Callback` 활용

- `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '2.4.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group 'com.windowforsun.spring'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.11

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testCompileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testAnnotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'

}
```  


### Synchronous



### Callable

practice_asychronous_web_1


### DeferredResult
practice_asychronous_web_2

### Emitter

### WebClient(skip)


### ListenableFuture
controller 에서 그냥 외부 api 비동기 결과 리턴
listenablefuture 에서 callback 으로 결과 처리는 결과처리에 대한 결과를 다시 리턴 받을 수 없음


### DiferredResult
controller 에서 외부 api 비동기 결과 가공 후 리턴 
callback hell 까지

### DeferredResult Callback hell 개선 ? Completion
callback hell 해결 라이브러리 구현

### CompletableFuture ?
java 에서 공식 지원하는 비동기 작업 파이프라이닝 라이브러리 사용

---
## Reference
[Creating Asynchronous Methods](https://spring.io/guides/gs/async-method/)  
[Effective Advice on Spring Async: Part 1](https://dzone.com/articles/effective-advice-on-spring-async-part-1)  