--- 
layout: single
classes: wide
title: "[Jetbrains] Intellij Gradle 프로젝트 만들기"
header:
  overlay_image: /img/jetbrains-bg.jpg
excerpt: 'Intellij 에서 Gradle 프로젝트를 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Jetbrains
tags:
  - Jetbrains
  - Intellij
  - Java
  - Gradle
---  

## 환경
- Intellij
- Java 8
- Gradle

## Gradle 프로젝트 생성
- File -> New -> Project ...

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-1.png)

- Gradle -> Java

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-2.png)

- GroupId, ArtifactId, Version 을 적어 준다.
	- GroupId 는 생성하는 프로젝트를 고유하게 식별하는 것으로 Domain Name 이 적합하며 Package 명명 규칙을 따른다.
	- ArtifactId 는 생성하는 프로젝트 제품의 이름으로, 버전 정보를 생략한 jar 파일 이름이고 프로젝트 이름과 동일하게 설정한다.
	- Version 은 개발용은 SNAPSHOT, 배포용은 RELEASE 로 지정하며 숫자와 점을 이용해서 표현한다. (1.0.1, 1.1.1)

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-3.png)

- 프로젝트 생성 설정

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-4.png)

- 프로젝트 경로 설정

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-5.png)

- 프로젝트가 생성되면 아래와 같은 디렉토리 구조가 나온다.

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-6.png)

- Gradle Sync 성공할 때까지 기다린다.

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-7.png)

- 개발을 위한 main, test 등의 기본 디렉토리 구조 생성
	- File -> Settings.. -> Gradle 검색 -> Create directories for empty content roots automatically 선택
	
![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-8.png)
	
- 생성된 기본 디렉토리 구조

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-9.png)

- Junit 테스트를 위해 java 디렉토리에 `com.java.exam` 패키지에 아래 클래스를 만든다.

```java
package com.java.exam;

public class Calculator {
    public int plus(int a, int b) {
        return a + b;
    }

    public int minus(int a, int b) {
        return a - b;
    }
}
```  

- test 디렉토리에도 동일한 패키지를 만들고 아래와 같은 테스트 클래스를 생성한다.

```java
public class CalculatorTest {
    private Calculator calculator;

    @Before
    public void setUp() {
        this.calculator = new Calculator();
    }

    @Test
    public void plus() {
        int result = this.calculator.plus(10, 1);

        Assert.assertThat(result, is(11));
    }

    @Test
    public void minus(){
        int result = this.calculator.minus(10, 1);

        Assert.assertThat(result, is(9));
    }
}
```  

- 프로젝트 구조

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-10.png)

- Junit 테스트 결과

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-11.png)

---
## Reference
