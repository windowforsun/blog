--- 
layout: single
classes: wide
title: "[Java 실습] Gradle Docker 빌드와 이미지 생성"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Gradle
    - Spring Boot
    - Jib
toc: true
use_math: true
---  

## Gradle Docker 이미지 생성
- `Gradle` 빌드 도구를 사용해서 `Docker` 이미지를 빌드하는 방법은 여러가지가 있다.
- 직접 `Docker` 이미지 상에서 프로젝트를 빌드 하는 것도 하나의 방법일 수 있다.
- [palantir/gradle-docker](https://github.com/palantir/gradle-docker) 와 [GoogleContainerTools/jib](https://github.com/GoogleContainerTools/jib)  를 통해 `Gradle` 의 `Task` 를 사용해서 이미지를 빌드하는 방법에 대해 알아본다.

## 예제 프로젝트
- 예제 프로젝트는 `Spring Boot` 기반으로 구성했다. (일반 Java 애플리케이션도 무관하게 사용가능)
- 애플리케이션보다 빌드를 하고 빌드된 결과물을 실행할 수 있는 이미지를 만드는 것이 목적이기 때문에 아주 간단하게 구성했다.

### 프로젝트 구조
- 예제에 필요한 프로젝트의 구성한 나열하면 아래와 같다.

```
│  build.gradle
│  gradlew
│  gradlew.bat
│  settings.gradle
│
├─gradle
│  └─wrapper
│          gradle-wrapper.jar
│          gradle-wrapper.properties
│
└─src
    └─main
       └java
          └─com
              └─windowforsun
                  └─gradledockerbuild
                          GradledockerbuildApplication.java
```  

### build.gradle

```groovy
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

plugins {
    id 'org.springframework.boot' version '2.2.7.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'java'
}

group = 'com.windowforsun'
version = '0.0.1-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_1_8
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

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

- 아직 `Docker` 이미지 빌드 관련 설정을 추가하지 않은 상태이다.
- `Gradle` 을 통해 프로젝트를 설정하기 위한 설정들이 있다.

### GradledockerbuildApplication

```java
@SpringBootApplication
@RestController
public class GradledockerbuildApplication {

    public static void main(String[] args) {
        SpringApplication.run(GradledockerbuildApplication.class, args);
    }

    @GetMapping("/")
    public String home() {
        return "Hello Docker World!!";
    }
}
```  

- `Spring Boot` 를 시작하는 `Application` 클래스에서 바로 `@RestController` 를 사용해서 웹 요청을 받을 수 있도록 구성했다.


## palantir/gradle-docker
- 사용할 플러그인은 `com.palantir.docker` 이고, 이외에도 `com.palantir.docker-compose`, `com.palantir.docker-run` 등을 지원한다.
- 기본적으로 `Gradle` 을 통해 빌드의 결과물을 만들어내고 별도로 구성한 `Dockerfile` 을 빌드해 프로젝트의 `Docker` 이미지를 만들어 내는 방식이다.

### Dockerfile

```dockerfile
FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```  

- `openjdk:8-jdk-alpine` 이미지를 사용해서 빌드로 만들어진 `jar` 파일을 복사하고 `ENTRYPOINT` 를 지정해 주고 있다.

### build.gradle

```groovy
plugins {
	.. 생략 ..
    // for palantir/gradle-docker
    id 'com.palantir.docker' version '0.22.1'
}

.. 생략 ..

ext {
    BUILD_VERSION = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
}

docker {
    name "windowforsun/gradletest-simple"
    tag "${project.version}", "${BUILD_VERSION}"
    files tasks.bootJar.outputs.files
    buildArgs(['JAR_FILE': tasks.bootJar.outputs.files.singleFile.name])
}
```  

- `com.palantir.docker` 플러그인을 추가해 준다.
- `ext` 변수를 사용해서 빌드 버전인 `BUILD_VERSION` 을 시간을 기준으로 생성해 준다.
- `docker` 변수에 위와 같이 이미지 이름, 태그 등 필요한 설정을 해준다.





---
## Reference
[Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker/)  
[palantir/gradle-docker](https://github.com/palantir/gradle-docker)  
[GoogleContainerTools/jib](https://github.com/GoogleContainerTools/jib)  