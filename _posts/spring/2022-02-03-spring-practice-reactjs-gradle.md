--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot + ReactJS + Gradle 환경 구성"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 애플리케이션을 Back-End, ReactJS 를 FE로 사용하는 구조에서 Gradle 을 사용해서 빌드, 패키징을 수행해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - ReactJS
    - Gradle
    - Spring Web
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
`Spring Boot Web Application` 을 구성하는 디렉토리 구조는 아래와 같다.  

```
.
├── build.gradle
└── src
    └── main
        └── java
            └── com
                └── windowforsun
                    └── springreactjs
                        └── basic
                            ├── BasicApplication.java
                            └── HelloRestController.java

```

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

`Controller` 에서는 `안녕하세요.`, `hello`, 그리고 현재 날짜와 시간을 배열로 응답한다.  

### ReactJS
`ReactJS` 를 사용하기 위해서는 먼저 `NodeJS` 설치가 필요한데, 
사용자 개발환경에 맞는 것으로 
[링크](https://nodejs.org/ko/download/)
에서 설치한다.  

설치가 완료 됐다면, `Spring Boot` 프로젝트 루트 경로에서 아래 명령어로 `frontend` 라는 `ReactJS` 프로젝트를 생성해 준다.  

```bash
$ npx create-react-app frontend
```  

> `create-react-app` 는 `facebook` 에서 만든 `boilerplate` 로 간단한 명령어로 `ReactJS` 개발환경을 구성할 수 있다. 

명령어 실행후 디렉토리 구조는 아래와 같다.  

```
.
├── build.gradle
├── frontend
│   ├── README.md
│   ├── package-lock.json
│   ├── package.json
│   ├── public
│   │   ├── favicon.ico
│   │   ├── index.html
│   │   ├── logo192.png
│   │   ├── logo512.png
│   │   ├── manifest.json
│   │   └── robots.txt
│   └── src
│       ├── App.css
│       ├── App.js
│       ├── App.test.js
│       ├── index.css
│       ├── index.js
│       ├── logo.svg
│       ├── reportWebVitals.js
│       └── setupTests.js
└── src
    └── main
        └── java
            └── com
                └── windowforsun
                    └── springreactjs
                        └── basic
                            ├── BasicApplication.java
                            └── HelloRestController.java

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
간단한 연동 예제를 위해 기본으로 만들어진 `App.js` 를 수정하는 방법을 사용한다. 
아래와 같이 `App.js` 파일에 내용을 추가해 준다.  

```javascript
import logo from './logo.svg';
import './App.css';
import {useEffect, useState} from "react";

function App() {
  const [message, setMessage] = useState([]);

  useEffect(() => {
    fetch("/api/hello")
        .then((response) => {
          return response.json();
        })
        .then(function(data) {
          setMessage(data);
        })
  }, []);

  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.js</code> and save to reload.
        </p>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          Learn React
        </a>
        <ul>
          {message.map((text, index) => <li key={`${index}-${text}`}>{text}</li>)}
        </ul>
      </header>
    </div>
  );
}

export default App;
```  

다시 `localhost:3000` 으로 접속하면 아래와 같이 `Back-end` 서버인 `Spring Boot Controller` 로 요청후 그 응답을 `ReactJs` 를 사용해서 그린 결과를 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/spring/practice-reactjs-gradle-2.png)  


### Gradle 빌드
`CI/CD` 구성시에 필요한 `Gradle` 을 사용한 빌드와 패키징에 대해서 알아본다. 
기존 `build.gradle` 파일에 아래 내용을 추가해 준다.  

```groovy
def frontendDir = "$projectDir/frontend"

sourceSets {
    main {
        resources {
            srcDirs = ["$projectDir/src/main/resources"]
        }
    }
}

processResources {
    dependsOn "copyReactBuildFiles"
}

task installReact(type: Exec) {
    workingDir "$frontendDir"
    inputs.dir "$frontendDir"
    group = BasePlugin.BUILD_GROUP
    if (System.getProperty('os.name').toLowerCase(Locale.ROOT).contains("windows")) {
        commandLine "npm.cmd", "audit", "fix"
        commandLine 'npm.cmd', 'install'
    } else {
        commandLine "npm", "audit", "fix"
        commandLine 'npm', 'install'
    }
}

task buildReact(type: Exec) {
    dependsOn "installReact"
    workingDir "$frontendDir"
    inputs.dir "$frontendDir"
    group = BasePlugin.BUILD_GROUP
    if (System.getProperty('os.name').toLowerCase(Locale.ROOT).contains('windows')) {
        commandLine "npm.cmd", "run-script", "build"
    } else {
        commandLine "npm", "run-script", "build"
    }
}

task copyReactBuildFiles(type: Copy) {
    dependsOn "buildReact"
    from "$frontendDir/build"
    into "$projectDir/src/main/resources/static"
}
```  

빌드 및 패키징 방법은 간단하다. 
`frontend` 경로에서 `npm` 을 사용해서 `ReactJS` 구성을 우선 해주고, 
빌드해서 나온 결과를 `Spring Boot Web Application` 의 `Resources` 경로로 복사해주는 방법으로 진행된다.  

`IDE` 에서 `Spring Boot Application` 을 실행해 주거나, 
아래 `./gradlew build` 명렁을 사용해서 빌드한 `.jar` 파일을 실행해준다.  

실제 빌드후 타겟 `ReactJS` 의 타겟 결로인 `resources/static` 을 확인하면 아래와 같다.  

```
resources/static
├── asset-manifest.json
├── favicon.ico
├── index.html
├── logo192.png
├── logo512.png
├── manifest.json
├── robots.txt
└── static
    ├── css
    │   ├── main.073c9b0a.css
    │   └── main.073c9b0a.css.map
    ├── js
    │   ├── 787.cda612ba.chunk.js
    │   ├── 787.cda612ba.chunk.js.map
    │   ├── main.429a561a.js
    │   ├── main.429a561a.js.LICENSE.txt
    │   ├── main.429a561a.js.map
    │   ├── main.52eac330.js
    │   ├── main.52eac330.js.LICENSE.txt
    │   └── main.52eac330.js.map
    └── media
        └── logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg
```  

그리고 `localhost:8080` 으로 접속하면 아래와 같이, `localhost:3000` 으로 접속했을 때와 동일하게 `ReactJS` 의 결과화면을 확인할 수 있다.  

![그림 1]({{site.baseurl}}/img/spring/practice-reactjs-gradle-3.png)



---
## Reference