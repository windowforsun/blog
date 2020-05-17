--- 
layout: single
classes: wide
title: "[Java 실습] Gradle Multi-Project"
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
    - Multi-Project
toc: true
use_math: true
---  

## Gradle Multi-Project
- 서비스를 구성하다보면, 다양한 목적에 맞는 애플리케이션을 통해 하나의 서비스를 구축하게 된다.
- 다양하지만 하나의 서비스에 포함되는 애플리케이션들은 도메인을 공유하는 등의 다른 애플리케이션의 특정 부분를 공유해야 하는 상황이 발생한다.
- 위 상황에서 최악의 경우에는 구성하는 각 애플리케이션 마다 동일한 코드를 복/붙해야 하는 상황이 발생한다.
- 이런 상황을 해결할 수 있는 방법이 `Gradle Multi-Project`  이다.
- 간단한 예제 애플리케이션을 구성해서 `Gradle Multi-Project` 에 대한 기본적인 구성과 빌드에 대해 알아본다.


## Mutli-Project 만들기

### Root 프로젝트 생성
- `Intellij` 를 기준으로 `Gradle` 기반 `Multi-Project` 를 구성하는 방법에 대해 알아본다.
- `File -> New -> Project ...` 를 눌러 아래와 같이 새로운 `Gradle` 프로젝트를 생성한다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-1.png)
	
- `GroupId`, `ArtifactId`, `Version` 을 입력해 준다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-2.png)

- `Use auto-import` 를 사용할 경우 체크하고 다음으로 넘어간다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-3.png)
	
- 생성할 프로젝트의 이름과 경로를 설정하고 생성을 완료한다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-4.png)
	
### Sub Project 생성
- `Sub Project` 는 간단하게 공통 코드가 있는 `core` 프로젝트, 조회관련 API 를 제공하는 `web-read` 프로젝트, 생성및 업데이트 API 를 제공하는 `web-save` 프로젝트로 구성한다.
- 프로젝트의 구조를 정리하면, 실행되는 애플리케이션은 `web-read` 와 `web-save` 이고 두 프로젝트(애플리케이션)는 `core` 프로젝트에 대해 의존성을 가지는 구조이다.
- `Root Project` 에서 `Sub Project` 는 프로젝트 이름에서 오른쪽 클릭을 해서 `New -> Module` 을 눌러 생성할 수 있다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-5.png)
	
- 새로운 `Gradle` 프로젝트를 눌러 생성한다.

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-6.png)
	
- `Sub Project` 에서 사용할 `ArtifactId` 인 `core` 를 입력한다. 

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-7.png)
	
- `Sub Project` 에서 사용할 모듈 이름을 입력하고 경로 확인 후 생성을 마친다.
	
	![그림 1]({{site.baseurl}}/img/java/practice-gradle-multiproject-8.png)
	
- `web-read`, `web-save` 또한 동일하게 생성해준다.

### Multi-Project Gradle 설정
- `Root Project`, `Sub Project` 까지 모두 생성한 프로젝트의 구조는 아래와 같다.

	```bash
	│  build.gradle
	│  gradlew
	│  gradlew.bat
	│  settings.gradle
	│
	├─.gradle
	│  └─이하 생략
	│
	├─core
	│  │  build.gradle
	│  │
	│  └─src
	│      ├─main
	│      │  ├─java
	│      │  └─resources
	│      └─test
	│          ├─java
	│          └─resources
	├─gradle
	│  └─wrapper
	│          gradle-wrapper.jar
	│          gradle-wrapper.properties
	│
	├─web-read
	│  │  build.gradle
	│  │
	│  └─src
	│      ├─main
	│      │  ├─java
	│      │  └─resources
	│      └─test
	│          ├─java
	│          └─resources
	└─web-save
		│  build.gradle
		│
		└─src
			├─main
			│  ├─java
			│  └─resources
			└─test
				├─java
				└─resources
	```  
	
- `settings.gradle` 파일에서 현재 프로젝트의 구조를 설정 할 수 있다. `Intellij` 를 통해 생성했다면 아래와 같이 설정돼 있다.

	```groovy
	rootProject.name = 'gradlemultiproject'
	include 'core'
	include 'web-read'
	include 'web-save'
	```  
	
- `Root Porject` 에 있는 `build.gradle` 에서 프로젝트 전체에 대한 의존성, 프로젝트, 빌드에 대한 설정을 할 수 있다.
	- 각 `Sub Project` 에도 `build.gradle` 이 존재하는데 해당 파일은 `Sub Project` 에만 해당하는 설정을 할 수 있다.
	
	```groovy
	plugins {
		id 'io.spring.dependency-management' version '1.0.8.RELEASE'
		id 'org.springframework.boot' version '2.2.1.RELEASE'
		id 'java'
	}
	
	// 실행 가능한 Jar 파일 비활성화
	bootJar {
		enabled = false
	}
	
	// 모든 프로젝트에 적용
	allprojects {
		repositories {
			mavenCentral()
		}
	}
	
	// 서브 프로젝트에 적용
	subprojects {
		apply plugin: 'java'
		group 'com.windowforsun'
		version '1.0-SNAPSHOT'
		sourceCompatibility = 1.8
	
		// 공통으로 사용되는 의존성
		dependencies {
			compileOnly 'org.projectlombok:lombok'
			annotationProcessor 'org.projectlombok:lombok'
			testCompile group: 'junit', name: 'junit', version: '4.12'
			testImplementation group: 'org.hamcrest', name: 'hamcrest-all', version: '1.3'
		}
	
		configurations {
			compileOnly {
				extendsFrom annotationProcessor
			}
		}
	
		test {
			useJUnitPlatform()
		}
	}
	```  
	- `root` 프로젝트의 경우 실행 가능한 `jar` 파일 생성이 필요하지 않기 때문에 비활성화 시킨다.
	- `allprojects` 를 통해 모든 프로젝트에서 적용되는 설정을 할 수 있다.
	- `subprojects` 에서는 `sub` 프로젝트에서만 적용되는 설정을 할 수 있다.
	
- 공통으로 사용되는 코드가 위치하는 `core` 프로젝트의 `build.gradle` 은 아래와 같다.

	```groovy
	plugins {
		id 'io.spring.dependency-management'
		id 'org.springframework.boot'
	}
	
	// 실행가능한 Jar 파일 비활성화
	bootJar {
		enabled = false
	}
	
	// 외부에서 의존성을 통해 사용가능한 Jar 파일 활성화
	jar {
		enabled = true
	}
	
	dependencies {
		compile('org.springframework.boot:spring-boot-starter-data-jpa')
		runtime('com.h2database:h2')
		testImplementation('org.springframework.boot:spring-boot-starter-test') {
				exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
		}
	}
	```  
	
	- `core` 프로젝트 또한 실행 가능한 `jar` 파일은 필요하지 않기 때문에 비활성화 시키고, 대신 외부에서 의존성을 추가해서 사용할 수 있도록 `jar` 파일을 활성화 한다.

- 실제로 애플리케이션으로 실행되는 `web-read` 프로젝트와 `web-save` 프로젝트의 `build.gradle` 은 아래와 같다.

	```groovy
	plugins {
        id 'io.spring.dependency-management'
        id 'org.springframework.boot'
        id 'com.palantir.docker' version '0.22.1'
    }
    
    dependencies {
        implementation project(':core')
        implementation 'org.springframework.boot:spring-boot-starter-web'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
                exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }
	```  
	
	- 웹 애플리케이션을 기반으로 하기 때문에 `core` 프로젝트와 필요한 의존성을 추가하고 있다.
	
	
---
## Reference
[Authoring Multi-Project Builds](https://docs.gradle.org/current/userguide/multi_project_builds.html)  
[Building a Multi-Module Spring Boot Application with Gradle](https://reflectoring.io/spring-boot-gradle-multi-module/)  
