--- 
layout: single
classes: wide
title: "메이븐(Maven) 개요"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Maven 이란 무엇이고 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Java
    - Maven
---  

# Maven 이란
> Apache Maven 은 Java 용 프로젝트 관리도구로 Apache Ant 의 대안으로 만들어 졌다.  
> Apache License 로 배포되는 오픈 소스 소프트웨어이다.  

- Java 의 프로젝트 빌드, 관리에 사용되는 도구로 개발자들이 전체 개발과정을 한 눈에 알아 볼 수 있다.

## 장점 및 특징
- 빌드 절차의 간소화
- 동일한 빌드 시스템 제공
- 프로젝트 정보 제공
- 라이브러리의 관리 용의
- 프로젝트 작성, 컴파일, 페트스 등 프로젝트 라이프사이클에 포함되는 각 테스트 지원

## 구조
![maven 구조]({{site.baseurl}}/img/java/java-maven-architecture-1.png)

## 디렉토리 구조
![메이블 디렉토리 구조]({{site.baseurl}}/img/java/java-maven-directory.jpg)
- src 밑으로 main 과 test 라는 디렉토리가 존재하고, 각 하위에는 java 와 resources 가 위치한다.
	- src/main/java : 자바 소스 파일이 위치한다.
		- 이 하위에 com.project.protype 와 같은 패키지를 배치한다.
	- src/main/resources : 프로퍼티나 XML 등 리소스 파일이 위치한다.
	- src/main/webapp : Web Project 일 경우 WEB-INF 등 웹 애플리케이션 리소스를 위치 시킨다.
	- src/test/java : JUnit 등의 테스트 파일이 위치한다.
	- src/test/resources : 테스트 시에 필요한 resource 파일이 위치한다.

# Plugin
- 메이븐은 플러그인을 구동해주는 프레임워크(Plugin execution framework)이다. 모든 작업은 플로그인에서 수행한다.
- 플러그인은 다른 산출물(Artifacts)와 같이 저장소에서 관리된다.
- 메이븐은 여러 플러그인으로 구성되어 있으며, 각각의 플러그인은 하나 이상의 Goal(명렁, 작업)을 포함하고 있다.
	- Goal 은 Maven의 실행 단위이다.
![maven plugin goal]({{site.baseurl}}/img/java/java-maven-plugingoal.png)
- 플러그인과 골의 조합으로 실행한다. 
	- mvn <plugin>:<goal> => mvn archetype:generate
- 메이픈은 여러 Goal 을 묶어서 Lifecycle phase 로 만들고 실행한다.
	- mvn <phase> => mvn install

## 플러그인 목록

구분 | Plugin name | 설명
---|---|---
core plugin | clean, compiler, deploy, failsafe, install, resources, site, surefire, verifier | 기본 단계에 해당하는 핵심 플러그인
Packaging types/tools | ear, ejb, jar, rar, war, app-client, shade | 압축 도구
Reporting plugins | changelog, changes, checkstyle, javadoc, pmd, surefire-report | 리포팅 도구
Tools | ant, antruna, archetpye, assembly, dependency, pdf, plugin, repository | 기타 다양한 도구

# Life Cycle
- 메이븐은 프레임워크이기 때문에 동작 방식이 정해져 있다.
- 일련의 단계(Phase)에 연계된 Goal 을 실행하는 과정을 Build 라고 한다.
- 미리 정의되어 있는 Build 들의 순서를 라이프사이클(LifeCycle) 이라고 한다.
- 즉, 미리 정의된 빌드순서를 라이프사이클 이라 하고, 각 빌드 단계를 Phase 라고 한다.

- 일반적으로 메이븐은 3개의 포준 라이프사이클을 제공한다.
![라이프사이클 목록]({{site.baseurl}}/img/java/java-maven-lifecyclelist.png)
	- Clean: 빌드 시 생성되었던 Output(산출물) 을 삭제한다.
		- pre-clean : clean 작업 전에 사전작업
		- clean : 이전 빌드에서 생성된 모든 파일 삭제
		- post-clean : 사후 작업
	- default(Build) : 일반적인 빌드 프로세스(배포 절차, 패키지 타입별로 다르게 정의)를 위한 모델이다.
		- validate : 프로젝트 상태 점검, 빌드에 필요한 정보 존재유무 체크
		- initialize : 빌드 상태를 초기화, 속성 설정, 작업 디렉토리 생성
		- generate-sources : 컴파일에 필요한 소스 생성
		- process-sources : 소스코드를 처리
		- generator-resources : 패키지에 포함될 리소스 생성
		- compile : 프로젝트의 소스코드를 컴파일
		- process-classes : 컴파일 후 후처리
		- generate-test-source : 테스트를 위한 소스코드 생성
		- process-test-source : 테스트 소스코드 처리
		- generate-test-resources : 테스트를 위한 리소스 생성
		- process-test-resources : 테스트 대상 디렉토리에 자원을 복사하고 가공
		- test-compile : 테스트 코드를 컴파일
		- process-test-classes : 컴파일 후 후처리
		- test : 단위 테스트 프레임워크를 이용해 테스트 수행
		- prepare-package : 패키지 생성 전 사전작업
		- package : 개발자가 선택한 war, jar 등의 패키징 수행
		- pre-integration-test : 통합 테스트 사전작업
		- integration-test : 통합 테스트
		- post-integration : 통합 테스트 후 사후 작업
		- verify : 패키지가 품질 기준이 적합한지 검사
		- install : 패키지를 로컬 저장소에 설치
		- deploy : 패키지를 원격 저장소에 배포
	- site : 프로젝트 문서화 절차와 사이트 작성을 수행한다.
		- pre-site : 사전작업
		- site : 사이트문서 생성
		- post-site : 사후작업 및 배포 전 사전작업
		- site-deploy : 생성된 문서를 웹 서버에 배포
	- Build default 라이프 사이클의 주요 Phase
		 - ![default 라이프 사이클의 주요 phase]({{site.baseurl}}/img/java/java-maven-defaultlifecycle.png)

# Dependency
- 개발자는 프로젝트에 사용할 라이프러리를 pom.xml 에 dependency 로 정의만 하면 메이븐이 repository 에서 검색해 자동으로 추가해준다.
- 참조하고 있는 library 까지 모드 찾아 추가해준다.
- 이러한 것들을 **의존성 전이**라고 한다.

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-core</artifactId>
  <version>5.1.5.RELEASE</version>
</dependency>
```  

## 의존관계 제한 기능
- 불필요한 라이브러리 다운로드를 방지하기 위해 추가기능을 제공한다.
	- Dependency mediation : 버전이 다른 두 개의 라이브러리가 동시에 의존 관계에 있을 경우 Maven 은 좀 더 가까운 의존관계에 있는 하나의 버전만 선택한다.
	- Dependency management : 직접 참조하지는 않지만 하위 모듈이 특정 모듈을 참조할 경우, 특정 모듈의 버전을 지정
	- Dependency scope : 현재 Build 단계에 꼭 필요한 모듈만 참조할 수 있도록 참조 범위를 지정
		- compile : 기본값, 모든 classpath 에 추가, 컴파일 및 배포 때 같이 제공
		- provided : 실행 시 외부에서 제공
			- ex) WAS 에서 제공되어 지므로 컴파일 시에는 필요하지만, 베포시에는 빠지는 라이브러리들
		- runtime : 컴파일 시 참조되지 않고 실행 때 참조
		- test : 테스트 시에만 참조
		- system : 저장소에서 관리하지 않고 직접 관리하는 jar 파일을 지정
		- import : 다른 pom 파일 설정을 가져옴, `<dependencyManagement>` 에서만 사용
	- Excluded dependencies : 임의의 모듈에서 참조하는 특정 하위 모듈을 명시적으로 제외처리
	- Optional dependencies : 임의의 모듈에서 Optional 로 참조된 모듈은 상위 모듈이 참조될 때  Optional 모듈은 참조 제외

## 의존 라이브러리 경로
- 로컬 : USER_HOME/.m2/repository 에 저장된다.

# Profile
- 메이븐은 서로 다른 환경에 따라 달라지는 설정을 각각 관리할 수 있는 profile 기능을 제공한다.

# POM.xml
- pom.xml 은 메이븐을 이용하는 프로젝트의 root에 존재하는 xml 파일이다.
- pom 은 프로젝트 객체 모델(Project Object Model) 을 뜻한다.
- 프로젝트 당 1개가 존재한다.
- pom.xml 을 통해 프로젝트의 모든 설정, 의존성을 알 수 있다.

## Element
- `<groupId>` : 프로젝트의 패키지 명칭
- `<artifactId>` : artifact 이름, groupId 내에서 유일해야 한다.

	```xml	
	  <groupId>org.springframework</groupId>
	  <artifactId>spring-core</artifactId>
	```  
	
- `<version>` : artifact 의 현재버전
	 - ex) 1.0-SNAPSHOT
- `<name>` : 애플리케이션 명칭
- `<packaging>` : 패키지 유형 (jar, war 등)
- `<distributionManagement>` : artifact 가 배포 될 저장소 정보와 설정
- `<parent>` : 프로젝트의 계층 정보
- `<dependencies>` : 의존성 정의 영역
- `<repositories>` : 프로젝트의 저장소 설정, 설정 하지 않으면 메이븐 저장소를 사용
- `<build>` : 빌드에 사용할 플러그인 목록을 나열
- `<reporting>` : 리포팅에 사용할 프러그인 목록을 나열
- `<properties>` : pom.xml 에서 사용할 값들을 properties 로 선언하여 사용한다. (주로 버전에 사용)

	```xml
	<!-- property 선언 -->
	<spring-version>5.1.5</spring-version>
	<!-- property 사용 -->
	<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>${spring-version}</version>
    </dependency>
	```  

## 전체적인 구조

```xml
<modelVersion>5.1.5</modelVersion>

<!-- 기본 설정 -->
<groupId></groupId>
<artifactId></artifactId>
<version></version>
<packaging></packaging>
<properties></properties>
<dependencies></dependencies>

<!-- 빌드 설정 -->
<build></build>
<reporting></reporting>

<!-- 프로젝트 관련 추가 정보 -->
<name></name>
<description></description>
<url></url>
<inceptionYear></inceptionYear>
<license></license>
<organization></organization>
<developers></developers>
<contributors></contributors>

<!-- 환경 설정 -->
<issueManagement></issueManagement>
<ciManagement></ciManagement>
<mailingLists></mailingLists>
<scm></scm>
<prerequisites></prerequisites>
<repositories></repositories>
<pluginRepositories></pluginRepositories>
<distributionManagement></distributionManagement>
<profiles></profiles>
```  


---
## Reference
[Maven](http://maven.apache.org/index.html)  
[maven (메이븐 구조, 차이점, 플러그인, 라이프사이클, 의존성, pom.xml)](https://sjh836.tistory.com/131)  
[Maven 을 이용한 프로젝트 생성 및 활용](https://unabated.tistory.com/entry/Maven-%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%9C-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%83%9D%EC%84%B1-%EB%B0%8F-%ED%99%9C%EC%9A%A9)  
[[Maven] 메이븐이란 무엇인가?](https://mangkyu.tistory.com/8)  
[Maven이란?](https://boxfoxs.tistory.com/324)  

