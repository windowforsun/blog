--- 
layout: single
classes: wide
title: "[Jenkins] Jenkins Spring Boot Tomcat 배포하기"
header:
  overlay_image: /img/jenkins-bg.jpg
excerpt: 'Jenkins 로 Github 에 있는 Spring Boot 프로젝트를 Tomcat 서버에 배포하자'
author: "window_for_sun"
header-style: text
categories :
  - Jenkins
tags:
  - Jenkins
  - Github
  - Maven
  - Tomcat
---  

## 환경
- CentOS 6
- Java 8
- Spring Boot 2.1.6
- Git 2.18
- Tomcat 8.5
- Jenkins 2.164
- Maven 3.6.0

## 사전 참고
- [Intellij Spring Boot 프로젝트 만들기]({{site.baseurl}}{% link _posts/2019-06-23-jetbrains-practice-springboot.md %})
- [Jenkins Github Maven 빌드 및 Tomcat 배포하기]({{site.baseurl}}{% link _posts/2019-04-06-jenkins-practice-springmavengitdeploy.md %})

## Spring Boot 프로젝트 설정
- `pom.xml` 파일에서 `<packaging></packaging>` 부분이 `war` 인지 확인 한다.
	- war : 외부 톰캣 사용
	- jar : Spring Boot 의 내부 톰캣 사용
	
```xml
<packaging>war</packaging>
```  
  	
## Jenkins 배포 Item 생성
- 젠킨스 대시보드 에서 새로운 Item 를 선택한다.

![그림 1]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-1.png)

- 이름을 입력하고 `Freestyle project` 를 눌러준다.

![그림 2]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-1.png)

- 소스코드 관리에서 Git 설정을 해준다.

![그림 2]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-2.png)

- Build 에서 Add build step 을 눌러 Invoke top-level Maven Project 를 선택한다.

![create item build invoke top-level maven project]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-8.png)

- Jenkins 환경 설정에서 설정한 maven 을 선택하고 Goals 에 `clean package` 를 기입한다.

![create item maven]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-9.png)

- 빌드 후 조치 > 빌드 후 조치 추가 > Deploy war/ear to a container 를 선택한다.

![create item 빌드 후 조치]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-10.png)

- WAR/EAR file 에는 `**/*.war`, Context path 에는 배포하고자 하는 애플리케이션의 context path 를 입력하고, Container > Add Container > Tomcat 8.x 혹은 Tomcat 7.x 를 선택한다.
- 배포 계정 정보도 설정후 저장 버튼을 누른다.

![그림 3]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-3.png)

## 배포하기
- 생성 한 `bootdeploy` 아이템에 들어가 `Build Now` 를 누른다.

![그림 4]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-4.png)

- Build History 를 통해 빌드가 성공 했는지 확인한다.

![그림 5]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-5.png)

- 배포한 서버에서 프로젝트의 ContextPath 로 접속해 확인한다. `http://<서버IP>:<포트>:<ContextPath>`

![그림 6]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-6.png)

![그림 7]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-7.png)

![그림 8]({{site.baseurl}}/img/jenkins/practice-springbootdeploy-8.png)



---
## Reference
