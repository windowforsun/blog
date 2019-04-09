--- 
layout: single
classes: wide
title: "[Jenkins] Jenkins Github Maven 빌드 및 Tomcat 배포하기"
header:
  overlay_image: /img/jenkins-bg.jpg
excerpt: 'Jenkins 로 Github 에 있는 Maven 프로젝트를 Tomcat 서버에 배포하자'
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
- Spring 4
- Git 2.18
- Tomcat 8.5
- Jenkins 2.164
- Maven 3.6.0


## Tomcat 배포 계정 및 manager 설정

- <Tomcat 설치 경로>/conf/tomcat-users.xml 파일을 연다.

```
[root@windowforsun ~]# vi /usr/local/tomcat8/conf/tomcat-users.xml
```  

- 배포 시에 사용할 manager 계정을 아래와 같이 추가한다.
	- `<user username="유저명" password="비밀번호" roles="접근가능한 role" />`
	
```xml
<tomcat-users>
	<role rolename="manager"/>
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <role rolename="admin"/>
    <user username="admin" password="manager" roles="admin,manager,manager-gui,manager-script,manager-jmx,manager-status"/>
</tomcat-users>
```  

- Tomcat 의 배포는 `서버IP:포트/manager` 에서 관리하는데 해당 페이지 접근관련 설정을 한다.
	- <Tomcat 설치 경로>/webapps/manager/META-INF/context.xml 파일을 연다.

```
[root@windowforsun ~]# vi /usr/local/tomcat8/webapps/manager/META-INF/context.xml
```  

- 아래와 같이 파일을 수정한다.

```xml
<Context antiResourceLocking="false" privileged="true" >
<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```  

- `서버IP:포트/manager` 에 접속해 추가한 계정을 입력하고 접속 여부를 확인한다.

![tomcat manager]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-tomcatmanager-1.png)


## 프로젝트 pom.xml 파일 설정
- 프로젝트의 pom.xml 파일에 Jenkins 배포시에 필요한 부분을 추가해 준다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>firstdeploy</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    
    <properties>
        <org.springframework-version>4.3.18.RELEASE</org.springframework-version>

        <!-- Jenkins에서 사용하는 부분 -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <!--  -->
    </properties>

    <dependencies>
        <!-- Spring core -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${org.springframework-version}</version>
        </dependency>
        <!-- Spring Web ( Servlet / Anotation ) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${org.springframework-version}</version>
        </dependency>
        <!-- Spring MVC -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${org.springframework-version}</version>
        </dependency>
    </dependencies>

    <!-- Jenkins에서 사용하는 부분 -->
    <profiles>
        <profile>
            <id>production</id>
            <build>
                <resources>
                    <resource>
                        <directory>${project.basedir}/src/main/resources</directory>
                        <excludes>
                            <exclude>**/*.java</exclude>
                        </excludes>
                    </resource>
                </resources>

                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-resources-plugin</artifactId>
                        <configuration>
                            <encoding>UTF-8</encoding>
                        </configuration>
                    </plugin>
                </plugins>
            </build>

            <!--<dependencies>-->
                <!--&lt;!&ndash; Servlet &ndash;&gt;-->
                <!--<dependency>-->
                    <!--<groupId>javax.servlet</groupId>-->
                    <!--<artifactId>javax.servlet-api</artifactId>-->
                    <!--<version>3.0.1</version>-->
                    <!--<scope>provided</scope>-->
                <!--</dependency>-->

                <!--<dependency>-->
                    <!--<groupId>javax.servlet.jsp</groupId>-->
                    <!--<artifactId>jsp-api</artifactId>-->
                    <!--<version>2.0</version>-->
                    <!--<scope>provided</scope>-->
                <!--</dependency>-->
            <!--</dependencies>-->
        </profile>
    </profiles>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <webXml>web/WEB-INF/web.xml</webXml>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.0.0</version>
                <configuration>
                    <warSourceDirectory>web</warSourceDirectory>
                </configuration>
            </plugin>

            <!-- Jenkins에서 사용하는 부분 -->
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <url>http://서버IP:포트/manager/text</url>
                    <path>/firstdeploy</path>
                    <username>admin</username>
                    <password>manager</password>
                </configuration>
            </plugin>
            <!--  -->
        </plugins>
    </build>
</project>
```  


## Github Token 생성

- 배포하고자 하는 프로젝트가 있는 자신의 Github 계정의 Settings 에 들어간다.

![github token settings 클릭]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-1.png) 

- Settings 목록에서 Developer settings 를 클릭한다.

![github token developer settings]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-2.png)

- Developer settings 목록에서 Personal access tokens 를 클릭한다.

![github token personal access tokens]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-3.png)

- Generate new token 을 클릭한다.

![github token generate new token]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-4.png)

- 비밀번호로 인증을 진행하고 token 이름 작성 및 repo, admin:repo_hook 를 체크를 한후 Generate token 를 클릭한다.

![github token create token]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-5.png)

- 생성된 token 를 복사해 둔다.

![github token created token]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-githubtokenl-6.png)


## Jenkins Github 연동
- Jenkins 관리에 들어간다.

![jenkins github jenkins 관리 진입]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-1.png)

- 시스템 관리에 들어간다.

![jenkins github 시스템 관리 진입]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-2.png)

- GitHub 관련 정보를 기입한다.
	- API URL default 값 그대로 사용한다.
	
![jenkins github github 설정]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-3.png)

- Add -> Jenkins 를 눌러 Github Credential 을 추가한다.

![jenkins github add jenkins credential]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-4.png)	

- Kind 부분을 Secret text 로 변경해주고, Secret 부분에는 Github token 값을 넣어 준다.

![jenkins github add credential]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-5.png)

- 새로 추가된 Credential 를 선택한다.

![jenkins github select credential]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-6.png)

- Test connection 을 눌러 연결여부를 확인한다.

![jenkins github test connection]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinsgithub-7.png)


## 플러그인 다운로드

![플러그인 다운로드 jenkins 관리]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-plugininstall-1.png)

![플러그인 다운로드 플러그인 관리]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-plugininstall-2.png)

- 설치가능 > 필터에서 Deploy to container 를 검색해서 플러그인을 설치한다.

![플러그인 다운로드 설치가능 검색]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-plugininstall-3.png)

![플러그인 다운로드 deploy to container]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-plugininstall-4.png)


## Jenkins 배포 환경 설정
- Jenkins 에 접속해 Jenkins 관리에 들어간다.

![jenkins setting jenkins 관리 진입]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinssettingl-1.png) 

- Global Tool Configuration 에 들어간다.

![jenkins setting global tool configuration]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinssettingl-2.png)

- 설치된 JDK 를 설정한다.

![jenkins setting jdk setting]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinssettingl-3.png)

- 설치된 Git 을 설정한다.

![jenkins setting git setting]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinssettingl-4.png)

- 설치된 Mavne 을 설정한다.

![jenkins setting maven setting]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-jenkinssettingl-5.png)

- Apply, Save 를 눌러 적용한다.


## Jenkins 배포 Item 생성
- 새로운 Item 을 클릭한다.

![create item select create item]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-1.png)

- Item 이름을 입력하고 Freestyle project 를 선택 후 OK 를 누른다.

![create item freestyle project ok]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-2.png)

- 배포하고자하는 프로젝트의 Github Repository URL 을 넣어준다.

![create item github url]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-3.png)

- 소스코드 관리에서 Git 을 선택하고 Github Repository URL 을 기입해준다.

![create item git repo url]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-4.png)

- Credential 에서 Add -> Jenkins 를 눌러 Github 계정을 추가한다.

![create item add credential]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-5.png)

![create item add github account]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-6.png)

- 새로 추가한 Github 계정을 눌러 Repository URL 접속을 확인한다.

![create item select github account and connection]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-7.png)

- Build 에서 Add build step 을 눌러 Invoke top-level Maven Project 를 선택한다.

![create item build invoke top-level maven project]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-8.png)

- Jenkins 환경 설정에서 설정한 maven 을 선택하고 Goals 에 `clean package` 를 기입한다.

![create item maven]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-9.png)

- 빌드 후 조치 > 빌드 후 조치 추가 > Deploy war/ear to a container 를 선택한다.

![create item 빌드 후 조치]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-10.png)

- WAR/EAR file 에는 `**/*.war`, Context path 에는 배포하고자 하는 애플리케이션의 context path 를 입력하고, Container > Add Container > Tomcat 8.x 혹은 Tomcat 7.x 를 선택한다.

![create item 빌드 후 조치 war context path]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-11.png)

- Container > Credentials > Add > Jenkins 를 눌러 Tomcat 배포 계정을 추가해 준다.

![create item tomcat credentials 추가 진입]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-12.png)

- Tomcat 에서 추가한 배포 계정의 Username, Password 를 적고 Add 버튼을 누른다.

![create item tomcat 배포 계정 credential 추가]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-13.png)

- Credentials 에 추가한 credential 선택

![create item tomcat credential 선택]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-14.png)

- Tomcat URL 에 현재 배포하고자 하는 서버 도메인 입력을 입력한다.

![create item tomcat 서버 도메인 입력]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-createitem-15.png)

- Apply, Save 눌러 적용 한다.


## 배포하기
- 위에 만든 배포 프로젝트 `firstdeploy` 에 들어가 Build Now 를 눌러 배포를 시작한다.

![deploy build now]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-deploy-1.png)

- Build History 에서 성공 여부를 확인 한다. 빨간색일 경우 배포가 실패한 경우이다.
	- 실패 했거나 배포 로그를 확인 해야 할경우 해당 Build 를 눌르고 Console Output 를 눌러 확인 할 수 있다.

![deploy 배포 성공 확인]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-deploy-2.png)

- 배포한 서버에서 프로젝트의 Context Path 로 접속을 확인 한다. `서버IP:포트/ContextPath`

![deploy 서버에서 페이지 확인]({{site.baseurl}}/img/jenkins/jenkins-gitmaventomcatdeploy-deploy-3.png)

---
## Reference
[Tomcat :: 톰캣8 manager 403 access denied 해결 방법](https://hongku.tistory.com/196)  
[[Spring/Jenkins] 젠킨스로 배포하기 - 지속적인 통합 ( CI )](https://victorydntmd.tistory.com/230)  
[gitHub와 Jenkins 연결하기](https://bcho.tistory.com/1237)  
[jenkins tomcat 배포](https://handcoding.tistory.com/25)  
[JENKINS/GITHUB/MAVEN 으로 빌드&배포하기-1/4](https://dukeom.wordpress.com/2017/03/19/jenkinsgithubmaven-%EC%9C%BC%EB%A1%9C-%EB%B9%8C%EB%93%9C%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0-14/)  
[[애플리케이션 배포] 웹 애플리케이션 Tomcat 서버에 배포하기 - 젠킨스(Jenkins) (2)](https://blog.tophoon.com/2018/03/23/deploy-war-to-tomcat-jenkins.html)  
[Maven maven-war-plugin 사용시 Error 처리](https://enosent.tistory.com/49)  
[jenkins tomcat 401 Unauthorized](http://jagdeesh1009.blogspot.com/2016/04/jenkins-tomcat-401-unauthorized-org.html)  
[The username you provided is not allowed to use the text-based Tomcat Manager (error 403) when deploying on remote Tomcat8 using Jenkins](https://stackoverflow.com/questions/41675813/the-username-you-provided-is-not-allowed-to-use-the-text-based-tomcat-manager-e)  
[13. Jenkins 와 gitlab 연동 및 Tomcat배포 자동화](https://wikidocs.net/16281)  
