--- 
layout: single
classes: wide
title: "[Jetbrains] Intellij Spring Gradle 프로젝트 만들기"
header:
  overlay_image: /img/jetbrains-bg.jpg
excerpt: 'Intellij 에서 Spring Gradle 프로젝트를 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Jetbrains
tags:
  - Jetbrains
  - Intellij
  - Spring MVC
  - Spring
  - Java
  - Gradle
---  

## 환경
- Intellij
- Java 8
- Spring 4
- Gradle

## 필요한 선행 작업
- [Intellij Gradle 프로젝트 만들기]({{site.baseurl}}{% link _posts/2019-05-14-jetbrains-gradleproject.md %})

## Gradle 프로젝트에 Spring Framework 추가하기
- 프로젝트 오른쪽 클릭 -> Add Framework Support ...

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-12.png)

- Web Application 체크 후 아래로 스크롤

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-13.png)

- Spring -> Spring MVC 체크

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-14.png)

- Gradle 프로젝트에 Spring Framework 가 추가된 디렉토리 구조

![spring gradle new project 1]({{site.baseurl}}/img/jetbrains/spring-gradle-newproject-15.png)

- Gradle Spring Framework 프로젝트 관련 .gitignore

```
### JAVA ###
# Compiled class file
*.class

# Log file
*.log

# BlueJ files
*.ctxt

# Mobile Tools for Java (J2ME)
.mtj.tmp/

# Package Files #
*.jar
*.war
*.nar
*.ear
*.zip
*.tar.gz
*.rar

# virtual machine crash logs, see http://www.java.com/en/download/help/error_hotspot.xml
hs_err_pid*


### Gradle ###
.DS_Store
.gradle
/build/
/out/
/target/

# Ignore Gradle GUI config
gradle-app.setting

# Avoid ignoring Gradle wrapper jar file (.jar files are usually ignored)
!gradle-wrapper.jar
!gradle/wrapper/gradle-wrapper.jar

# Cache of project
.gradletasknamecache

# # Work around https://youtrack.jetbrains.com/issue/IDEA-116898
# # gradle/wrapper/gradle-wrapper.properties


### STS ###
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
bin/


### IntelliJ IDEA ###
.idea
*.iws
*.iml
*.ipr
https://gmlwjd9405.github.io/2018/10/24/intellij-springmvc-gradle-setting.html
```  

---
## Reference
