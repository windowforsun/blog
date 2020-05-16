--- 
layout: single
classes: wide
title: "[Spring 실습] Gradle Docker 빌드와 이미지 생성"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Gradle 을 사용해서 Spring 애플리케이션을 Docker 이미지로 빌드하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Java
    - Gradle
    - Spring Boot
    - Jib
    - Docker
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
- 실제로는 설치된 `Docker daemon` 을 사용해서 빌드작업을 수행한다.

### Dockerfile

```dockerfile
FROM openjdk:8-jre-alpine
ARG DEPENDENCY=target/dependency
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","com.windowforsun.gradledockerbuild.GradledockerbuildApplication"]
```  

- `Dockerfile` 은 프로젝트 루트에 위치시킨다.
- `COPY ${JAR_FILE} app.jar` 과 같이 `jar` 파일을 복사해서 이미지를 구성할 수도 있다. 하지만 이러한 방식은 `Docker` 이미지를 계속해서 여러 버전으로 생성할 때 `Docker Layer Caching` 을 잘 활용하지 못하는 방법이다.
- 위와 같은 이유로 `jar` 파일을 복사해서 이미지를 빌드하는 것이 아니라, 빌드된 파일에서 앱 구성에 필요한 의존성을 분리해서 `Dockerizing` 을 수행한다. 
- `openjdk:8-jdk-alpine` 이미지를 사용하고, 빌드로 만들어진 의존성관련 파일을 복사하고 `ENTRYPOINT` 를 지정해 주고 있다.
- 이전 이미지 버전에서 캐시된 `layer` 를 재사용하고 변경된 부분만 추가 `layer` 가 생성된다는 장점이 있다.

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

// for palantir/gradle-docker
task unpack(type: Copy) {
    dependsOn bootJar
    from(zipTree(tasks.bootJar.outputs.files.singleFile))
    into("build/dependency")
}
docker {
    name "windowforsun/gradletest-simple"
    tag 'projectVersion', "${name}:${project.version}"
    tag 'buildVersion', "${name}:${BUILD_VERSION}"
    copySpec.from(tasks.unpack.outputs).into("dependency")
    buildArgs(['DEPENDENCY' : "dependency"])
}
```  

- `com.palantir.docker` 플러그인을 추가해 준다.
- `ext` 변수를 사용해서 빌드 버전인 `BUILD_VERSION` 을 시간을 기준으로 생성해 준다.
- `unpack` 에서는 빌드로 생성된 `jar` 파일에서 의존성 파일을  추출해 복사한다.
- `docker` 에서는 이미지 이름, 태그 등 필요한 설정을 하고 `unpack` 에서 추출한 의존성 파일을 `Dockerfile` 에서 사용할 수 있도록 복사한다.
	- `tag` 를 통해 이미지에 태깅을 별도로 할 수 있는데 형식은 `tag <tag-name>, <imagename:tagname>` 형식이다. `tag` 부분은 `dockerPush` 명령어에서만 태깅작업을 수행한다.

### 빌드
- 빌드 명령어는 `./gradlew docker` 와 `./gradlew dockerPush` 로 할 수 있다.
	- `docker` 은 단순히 로컬에 빌드만 수행한다. 이경우 태깅은 기본적으로 `latest` 가 붙게 된다.
	- `dockerPush` 는 `docker` 를 수행해서 로컬에서 빌드를 수행하고 그 결과물에 대해서 `tag` 에 설정된 이름 대로 태깅을 한 수 저장소에 푸쉬까지 한다.
- `docker` 를 통해 빌드하면 이미지는 아래와 같이 `latest` 태그의 이미지만 생성되는 것을 확인 할 수 있다.

	```bash
	$ ./gradlew docker
    > Task :dockerClean
    > Task :compileJava UP-TO-DATE
    > Task :processResources UP-TO-DATE
    > Task :classes UP-TO-DATE
    > Task :bootJar UP-TO-DATE
    > Task :unpack UP-TO-DATE
    > Task :dockerPrepare
    > Task :docker
    
    BUILD SUCCESSFUL in 46s
    7 actionable tasks: 3 executed, 4 up-to-date
	```  
	
	```bash
	$ docker image ls | grep gradletest
    windowforsun/gradletest-simple   latest              bc3b4101c47e        30 seconds ago      102MB
	```  
	
- `dockerPush` 를 통해 빌드하면 아래와 같이 `latest` 뿐만아니라 `tag` 를 통해 설정한 이미지까지 생성되고, 모든 이미지는 저장소에 푸쉬 된다.

	```bash
	$ ./gradlew dockerPush
	> Task :dockerClean
	> Task :compileJava UP-TO-DATE
	> Task :processResources UP-TO-DATE
	> Task :classes UP-TO-DATE
	> Task :bootJar UP-TO-DATE
	> Task :unpack UP-TO-DATE
	> Task :dockerPrepare
	> Task :docker
	> Task :dockerTagBuildVersion
	> Task :dockerTagProjectVersion
	> Task :dockerTag
	
	> Task :dockerPush
	The push refers to repository [docker.io/windowforsun/gradletest-simple]
	0b63558fe75e: Preparing
	18beca3d9847: Preparing
	8387800690aa: Preparing
	edd61588d126: Preparing
	9b9b7f3d56a0: Preparing
	f1b5933fe4b5: Preparing
	f1b5933fe4b5: Waiting
	edd61588d126: Layer already exists
	9b9b7f3d56a0: Layer already exists
	f1b5933fe4b5: Layer already exists
	0b63558fe75e: Pushed
	18beca3d9847: Pushed
	8387800690aa: Pushed
	0.0.1-SNAPSHOT: digest: sha256:f163184f8cdb5218ba5d0a80c83b028520bacba2b513e35a88f37064ed643e35 size: 1573
	0b63558fe75e: Preparing
	18beca3d9847: Preparing
	8387800690aa: Preparing
	0b63558fe75e: Layer already exists
	18beca3d9847: Layer already exists
	edd61588d126: Preparing
	9b9b7f3d56a0: Preparing
	f1b5933fe4b5: Preparing
	8387800690aa: Layer already exists
	edd61588d126: Layer already exists
	9b9b7f3d56a0: Layer already exists
	f1b5933fe4b5: Layer already exists
	20200516190636: digest: sha256:f163184f8cdb5218ba5d0a80c83b028520bacba2b513e35a88f37064ed643e35 size: 1573
	0b63558fe75e: Preparing
	18beca3d9847: Preparing
	8387800690aa: Preparing
	edd61588d126: Preparing
	9b9b7f3d56a0: Preparing
	f1b5933fe4b5: Preparing
	0b63558fe75e: Layer already exists
	18beca3d9847: Layer already exists
	8387800690aa: Layer already exists
	edd61588d126: Layer already exists
	9b9b7f3d56a0: Layer already exists
	f1b5933fe4b5: Waiting
	f1b5933fe4b5: Layer already exists
	latest: digest: sha256:f163184f8cdb5218ba5d0a80c83b028520bacba2b513e35a88f37064ed643e35 size: 1573
	
	BUILD SUCCESSFUL in 33s
	10 actionable tasks: 6 executed, 4 up-to-date
	```  
	
	```bash
	$ docker image ls | grep gradletest
	windowforsun/gradletest-simple   0.0.1-SNAPSHOT      bc3b4101c47e        3 minutes ago       102MB
	windowforsun/gradletest-simple   20200516190636      bc3b4101c47e        3 minutes ago       102MB
	windowforsun/gradletest-simple   latest              bc3b4101c47e        3 minutes ago       102MB
	```  
	
- 빌드된 이미지를 아래 명령어를 통해 컨테이너로 실행하고 확인하면 아래와 같이 정상적으로 애플리케이션이 실행되는 것을 확인 할 수 있다.

	```bash
	$ docker run --rm --name gradletest -p 8080:8080 -t windowforsun/gradletest-simple:latest
    
      .   ____          _            __ _ _
     /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
    ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
     \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
      '  |____| .__|_| |_|_| |_\__, | / / / /
     =========|_|==============|___/=/_/_/_/
     :: Spring Boot ::        (v2.2.7.RELEASE)
    
    2020-05-16 00:24:31.181  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : Starting GradletestSimpleApplication on 8fb6b814a797 with PID 1 (/app/classes started by root in /)
    2020-05-16 00:24:31.617  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : No active profile set, falling back to default profiles: default
    2020-05-16 00:24:32.260  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
    2020-05-16 00:24:32.374  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
    2020-05-16 00:24:32.872  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.34]
    2020-05-16 00:24:32.905  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
    2020-05-16 00:24:32.923  INFO 1 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1138 ms
    2020-05-16 00:24:33.014  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
    2020-05-16 00:24:33.223  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
    2020-05-16 00:24:33.315  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : Started GradletestSimpleApplication in 2.372 seconds (JVM running for 2.776)
	```  

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-dockerbuild-1.png)
	
## GoogleContainerTools/jib
- `palantir/gradle-docker` 를 사용한 빌드 방식을 정리하면, `jar` 파일을 만들고 다시 압축을 해제한다음 의존성을 분리하는 등의 순서로 이미지를 생성한다.
- 또한 `palantir/gradle-docker` 는 `Docker daemon` 을 사용하기 때문에 빌드 환경에서 `Docker` 설치는 필수 적이다.
- 다른 방법 중 하나는 `GoogleContainerTools/jib` 을 사용하는 방법으로 `Gradle`, `Maven` 에서 모두 사용가능하고 간단하게 `Docker` 이미지를 생성해주는 도구이다.
- `palantir/gradle-docker` 와 또 다른 점은 `GoogleContainerTools/jib` 은 좀더 `Java Application` 에 최적화된 방식으로 `Dockerizing` 을 수행하기 때문에 별도로 의존성을 분리하는 등의 작업을 필요하지 않다.
- 또한 빌드를 위해 `Docker` 를 설치할 필요도 없고, 별도의 `Dockerfile` 을 작성할 필요도 없다.

### build.gradle

```groovy
plugins {
	.. 생략 ..
	
    // for GoogleContainerTools/jib
    id 'com.google.cloud.tools.jib' version '1.6.0'
}

.. 생략 ..

ext {
    BUILD_VERSION = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
}

// for GoogleContainerTools/jib
jib {
    from {
        image = "openjdk:8-jre-alpine"
    }
    to {
        image = "windowforsun/gradletest-simple"
        tags = ["${project.version}".toString(), "${BUILD_VERSION}".toString()]
        auth {
            username = <repository-account>
            password = <repository-passwd>
        }
    }
    container {
        mainClass = "com.windowforsun.gradletestsimple.GradletestSimpleApplication"
        ports = ["8080"]
        creationTime = Instant.now()
        environment = [
            'ACTIVE_PROFILE' : 'dev'
        ]
    }
}
```  

- `com.google.cloud.tools.jib` 플러그인을 추가해 준다.
- 빌드 버전은 동일하게 현재 날짜와 시간을 기준으로 생성한다.
- `jib` 의 `from` 는 `base image` 에 대한 설정으로 애플리케이션 이미지(`target image`)를 구성할 떄 사용하는 기본 이미지(`base image`) 를 설정한다.
	- 로컬 도커의 이미지를 사용하고 싶은 경우 `docker://openjdk:8-jre-alpine` 라고 작성할 수 있고, 관련 설명은 [여기](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#setting-the-base-image)
	에서 확인 할 수 있다.
- `jib` 의 `to` 는 `target image` 에 대한 설정으로 이미지의 이름, 태그, 저장소의 인증 정보 등을 설정 한다.
	- 현재 `auth` 설정은 `Docker Hub` 의 계정 정보를 직접 입력하고 있지만, [여기](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#using-specific-credentials)
	와 같이 좀 더 다양한 방식으로 설정 가능하다.
- `jib` 의 `container` 는 빌드한 이미지를 실행 할때에 대한 설정으로 메인 클래스, 포트, 볼륨, 환경변수 등을 설정 할 수 있다.
	- `createTime` 이 별도로 지정되지 않은 경우, 이미지 빌드시간이 이상하게 나오는 현상이 있다.

### 빌드
- 빌드 명령어는 `./gradlew jib` 또는 `./gradlew jibDockerBuild` 등으로 할 수 있다.
	- `jib` 는 `target image` 를 빌드하고 태깅 후 설정된 저장소에 푸쉬까지 수행하지만, `Docker daemon` 을 사용하지 않기 때문에 로컬에서는 빌드된 이미지를 확인 할 수 없다
	- `jibDockerBuild` 는 `target image` 를 빌드하고 태깅 까지만 수행하고, `Docker daemon` 을 사용해서 수행하기때문에 로컬에서 확인이 가능하다.
- `./gradlew jibDockerBuild` 로 빌드를 수행하면 `target image` 생성 후 설정된 태깅까지만 수행한다.

	```bash
	$ ./gradlew jibDockerBuild
    > Task :compileJava UP-TO-DATE
    > Task :processResources UP-TO-DATE
    > Task :classes UP-TO-DATE
    Setting image creation time to current time; your image may not be reproducible.
    
    Containerizing application to Docker daemon as windowforsun/gradletest-simple, windowforsun/gradletest-simple:0.0.1-SNAPSHOT, windowforsun/gradletest-simple:20200516202522...
    The base image requires auth. Trying again for openjdk:8-jre-alpine...
    The credential helper (docker-credential-desktop) has nothing for server URL: registry-1.docker.io
    
    Got output:
    
    credentials not found in native keychain
    
    The credential helper (docker-credential-desktop) has nothing for server URL: registry.hub.docker.com
    
    Got output:
    
    credentials not found in native keychain
    
    Executing tasks:
    [=========                     ] 30.0% complete
    > pulling base image manifest
    
    
    Container entrypoint set to [java, -cp, /app/resources:/app/classes:/app/libs/*, com.windowforsun.gradletestsimple.GradletestSimpleApplication]
    
    Built image to Docker daemon as windowforsun/gradletest-simple, windowforsun/gradletest-simple:0.0.1-SNAPSHOT, windowforsun/gradletest-simple:20200516202522
    Executing tasks:
    [==============================] 100.0% complete
    
    
    BUILD SUCCESSFUL in 16s
    3 actionable tasks: 1 executed, 2 up-to-date
	```  
	
	```bash
	$ docker image ls | grep gradletest
    windowforsun/gradletest-simple   0.0.1-SNAPSHOT      31fa83be3806        40 seconds ago      102MB
    windowforsun/gradletest-simple   20200516202522      31fa83be3806        40 seconds ago      102MB
    windowforsun/gradletest-simple   latest              31fa83be3806        40 seconds ago      102MB
	```  
	

- `./gradlew jib` 를 통해 빌드를 수행하면 아래와 같이 `target image` 의 빌드를 수행하고, 설정된 태깅 수행 후 저장소에 푸쉬하게 된다.

	```bash
	$ ./gradlew jib
    > Task :compileJava UP-TO-DATE
    > Task :processResources UP-TO-DATE
    > Task :classes UP-TO-DATE
    Setting image creation time to current time; your image may not be reproducible.
    
    Containerizing application to windowforsun/gradletest-simple, windowforsun/gradletest-simple:0.0.1-SNAPSHOT, windowforsun/gradletest-simple:20200516202739...
    The base image requires auth. Trying again for openjdk:8-jre-alpine...
    The credential helper (docker-credential-desktop) has nothing for server URL: registry-1.docker.io
    
    Got output:
    
    credentials not found in native keychain
    
    The credential helper (docker-credential-desktop) has nothing for server URL: registry.hub.docker.com
    
    Got output:
    
    credentials not found in native keychain
    
    Executing tasks:
    [========                      ] 25.0% complete
    > pulling base image manifest
    > authenticating push to registry-1.docker.io
    
    Container entrypoint set to [java, -cp, /app/resources:/app/classes:/app/libs/*, com.windowforsun.gradletestsimple.GradletestSimpleApplication]
    
    Built and pushed image as windowforsun/gradletest-simple, windowforsun/gradletest-simple:0.0.1-SNAPSHOT, windowforsun/gradletest-simple:20200516202739
    Executing tasks:
    [===========================   ] 90.0% complete
    > launching layer pushers
    
    
    BUILD SUCCESSFUL in 21s
    3 actionable tasks: 1 executed, 2 up-to-date
	```  
	
	```bash
	$ docker image ls | grep gradletest
	REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
	.. NONE ...
	```  
	
- 빌드된 이미지를 아래 명령어를 통해 컨테이너로 실행하고 확인하면 아래와 같이 정상적으로 애플리케이션이 실행되는 것을 확인 할 수 있다.

	```bash
	$ docker run --rm --name gradletest -p 8080:8080 -t windowforsun/gradletest-simple:latest
    
      .   ____          _            __ _ _
     /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
    ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
     \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
      '  |____| .__|_| |_|_| |_\__, | / / / /
     =========|_|==============|___/=/_/_/_/
     :: Spring Boot ::        (v2.2.7.RELEASE)
    
    2020-05-16 02:44:31.681  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : Starting GradletestSimpleApplication on 8fb6b814a797 with PID 1 (/app/classes started by root in /)
    2020-05-16 02:44:31.687  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : No active profile set, falling back to default profiles: default
    2020-05-16 02:44:32.860  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
    2020-05-16 02:44:32.874  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
    2020-05-16 02:44:32.875  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.34]
    2020-05-16 02:44:32.947  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
    2020-05-16 02:44:32.948  INFO 1 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1138 ms
    2020-05-16 02:44:33.194  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
    2020-05-16 02:44:33.440  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
    2020-05-16 02:44:33.447  INFO 1 --- [           main] c.w.g.GradletestSimpleApplication        : Started GradletestSimpleApplication in 2.372 seconds (JVM running for 2.776)
	```  

	![그림 1]({{site.baseurl}}/img/java/practice-gradle-dockerbuild-1.png)

---
## Reference
[Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker/)  
[palantir/gradle-docker](https://github.com/palantir/gradle-docker)  
[GoogleContainerTools/jib](https://github.com/GoogleContainerTools/jib)  