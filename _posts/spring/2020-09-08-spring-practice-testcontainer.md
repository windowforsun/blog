--- 
layout: single
classes: wide
title: "[Spring 실습] TestContainers"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - TestContainers
toc: true
use_math: true
---  

## TestContainers
`TestContainers` 는 `JUnit` 테스르를 지원한다. 
그리고 테스트를 수행하며 필요한 다양한 요소들을 도커 컨테이너를 통해 실행할 수 있도록하는 경량화 `Java` 라이브러리이다.   

`TestContainers` 를 사용하면 아래와 같은 장점이 있다. 
- `Data access layer 통합 테스트` : `TestContainers` 는 `MySQL`, `PostgresSQL` 등 다양한 데이터 저장소를 지원한다. 
그리고 간단하게 데이터 저장소를 컨테이너화 해서 테스트시에 사용할 수 있고, 
테스트 시점에 데이터를 일관된 상태로 관리하는 것도 가능하다. 
- `Application 통합 테스트` : 애플리케이션 통합 테스트를 수행하기 위해서는 의존성을 갖는 다양한 요소들이 필요하다. 
`TestContainers` 를 사용하면 이러한 구성요소들을 도커 컨테이너로 실행할 수 있다. 
- [GenericContainer](https://www.testcontainers.org/features/creating_container/)
 를 사용하면 보다 커스텀하고 다양한 컨테이너를 구성해 테스트환경에 알맞는 테스트를 수행할 수 있다. 

`TestContainers` 의 `core` 의존성은 일반적인 컨테이너 구성과 `docker-compose` 를 지원한다. 

```groovy
testCompile "org.testcontainers:testcontainers:1.14.3"
```  

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.14.3</version>
    <scope>test</scope>
</dependency>
```  

추가로 `Module` 로 제공되는 의존성을 사용하면 보편적으로 자주 사용되는 이미지를 보다 간단하게 사용해볼 수 있다.  

## 간단 사용법
`TestContainers` 를 사용할때 `JUnit4`, `Junit5` 에 따라 사용법에 차이가 있기 때문에 나눠서 설명한다. 
테스트에서 사용하는 도커 컨테이너는 `Redis` 를 이용한다. 
그리고 `lettuce` 를 사용해서 해당 컨테이너에 접속해 간단한 테스트를 수행해본다.  

간단 사용법에서는 `TestContainers 1.14.x` 버전을 사용한다. 

### JUnit4
사용한 실제 `build.gradle` 파일 내용은 아래와 같다. 

```groovy
plugins {
    id 'java'
}

group 'com.windowforsun.quickjunit4'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    compile 'io.lettuce:lettuce-core:5.3.3.RELEASE'
    compile group: 'org.slf4j', name: 'slf4j-api', version: '1.7.25'
    testCompile group: 'org.slf4j', name: 'slf4j-log4j12', version: '1.7.25'

    testCompile "org.testcontainers:testcontainers:1.14.3"
}
```  

로그관련 설정 파일인 `log4j.properties` 내용은 아래와 같다. 

```properties
log4j.rootLogger=INFO, stdout

log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[ %d{yyyy-MM-dd HH:mm:ss} %-5p %x ] %-25C{1} :%5L - %m%n
```  

테스트 코드는 아래와 같다. 

```java
public class Junit4SimpleTest {
    private RedisClient redisClient;
    private StatefulRedisConnection<String, String> redisConnection;
    private RedisCommands<String, String> syncCommands;
    @Rule
    public GenericContainer redis = new GenericContainer("redis:5.0-alpine")
            .withExposedPorts(6379);

    @Before
    public void setUp() {
        String address = this.redis.getHost();
        int port = redis.getFirstMappedPort();

        this.redisClient = RedisClient.create("redis://" + address + ":" + port);
        this.redisConnection = this.redisClient.connect();
        this.syncCommands = this.redisConnection.sync();
    }

    @After
    public void tearDown() {
        this.redisConnection.close();
        this.redisClient.shutdown();
    }

    @Test
    public void set_get() {
        // given
        String key = "test";
        String value = "testValue";
        this.syncCommands.set(key, value);

        // when
        String actual = this.syncCommands.get(key);

        // then
        assertEquals(value, actual);
    }
}
```  

`JUnit` 의 `@Rule` 과 `TestContainers` 객체를 `GenericContainer` 를 사용해서 생성한다.  
그리고 `@Before` 에서는 생성된 컨테이너 정보를 바탕으로 `Redis` 클라이언트 초기화를 수행하고, 
`@After` 에서는 자원 반납 과정을 수행한다.  

실제로 테스트룰 수행하면 아래와 같은 로그 메시지를 통해 `TestContainers` 가 실행되고 테스트가 완료되면 종료되는 것을 확인할 수 있다. 

```bash
[INFO ] DockerClientProviderStrategy :  110 - Loaded org.testcontainers.dockerclient.NpipeSocketClientProviderStrategy from ~/.testcontainers.properties, will try it first
[INFO ] NpipeSocketClientProviderStrategy :   31 - Accessing docker with local Npipe socket (npipe:////./pipe/docker_engine)
[INFO ] DockerClientProviderStrategy :  119 - Found Docker environment with local Npipe socket (npipe:////./pipe/docker_engine)
[INFO ] DockerClientFactory       :  150 - Docker host IP address is localhost
[INFO ] DockerClientFactory       :  157 - Connected to docker: 
  Server Version: 19.03.12
  API Version: 1.40
  Operating System: Docker Desktop
  Total Memory: 25563 MB
[INFO ] DockerClientFactory       :  169 - Ryuk started - will monitor and terminate Testcontainers containers on JVM exit
[INFO ] DockerClientFactory       :  180 - Checking the system...
[INFO ] DockerClientFactory       :  237 - ✔︎ Docker server version should be at least 1.6.0
[INFO ] DockerClientFactory       :  237 - ✔︎ Docker environment should have more than 2GB free disk space
[INFO ] GenericContainer          :  357 - Creating container for image: redis:5.0-alpine
[INFO ] GenericContainer          :  418 - Starting container with ID: a84e73be6c603565b4472a063e00b2aaac88d8e510ce6d354b22d6f069c68d57
[INFO ] GenericContainer          :  422 - Container redis:5.0-alpine is starting: a84e73be6c603565b4472a063e00b2aaac88d8e510ce6d354b22d6f069c68d57
[INFO ] GenericContainer          :  476 - Container redis:5.0-alpine started in PT5.494S

Process finished with exit code 0
```  

### Junit5
`Junit5`(`Jupiter`) 를 사용한 방법은 다른 부분은 모두 동일하기 때문에 `build.gradle` 파일과 테스트 코드에 대해서만 설명한다. 

```groovy
plugins {
    id 'java'
}

group 'com.windowforsun.quickjunit5'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    compile 'io.lettuce:lettuce-core:5.3.3.RELEASE'
    compile group: 'org.slf4j', name: 'slf4j-api', version: '1.7.25'
    testCompile group: 'org.slf4j', name: 'slf4j-log4j12', version: '1.7.25'

    testCompile "org.junit.jupiter:junit-jupiter-api:5.6.2"
    testCompile "org.junit.jupiter:junit-jupiter-params:5.6.2"
    testRuntimeOnly "org.junit.jupiter:junit-jupiter-engine:5.6.2"
    testCompile "org.testcontainers:testcontainers:1.14.3"
    testCompile "org.testcontainers:junit-jupiter:1.14.3"
}
```  

```java
@Testcontainers
public class Junit5SimpleTest {
    private RedisClient redisClient;
    private StatefulRedisConnection<String, String> redisConnection;
    private RedisCommands<String, String> syncCommands;

    @Container
    public GenericContainer redis = new GenericContainer("redis:5.0-alpine")
            .withExposedPorts(6379);

    @BeforeEach
    public void setUp() {
        String address = this.redis.getHost();
        int port = this.redis.getFirstMappedPort();

        this.redisClient = RedisClient.create("redis://" + address + ":" + port);
        this.redisConnection = this.redisClient.connect();
        this.syncCommands = this.redisConnection.sync();
    }

    @AfterEach
    public void tearDown() {
        this.redisConnection.close();
        this.redisClient.shutdown();
    }

    @Test
    public void set_get() {
        // given
        String key = "test";
        String value = "testValue";
        this.syncCommands.set(key, value);

        // when
        String actual = this.syncCommands.get(key);

        // then
        assertEquals(value, actual);
    }
}
```  

`Junit5` 는 먼저 클래스 레벨에 `@TestContainers` 를 선언하고, `TestContainers` 객체에 `@Container` 와 `GenericContainer` 를 사용해서 객체를 생성한다. 
그리고 `Junit4` 와 비슷한 방식으로 `@BeforeEach` 에서 초기화 수행, `@AfterEach` 에서 자원 반납을 수행한다.  


## 컨테이너 만들기
이후 예제부터는 `TestContainers 1.15.x` 버전을 사용하고, `Junit4` 를 사용해서 진행한다. 
기본적으로 컨테이너는 `GenericContainer` 를 사용해서 생성할 수 있다. 
`GenericContainer` 생성자에 도커 이미지 이름을 전달하는 방식으로 모든 이미지를 사용해서 테스트 환경을 구성할 수 있다. 

```java
new GenericContainer(DockerImageName.parse("aline:latest"));
```  

이후에 살펴보겠지만 추가 모듈로 제공되는 `GenericContainer` 하위 클래스를 사용할때는 기본 이미지와 태그설정이 있기 때문에 파라미터를 통해 전달하지 않고 사용할 수 있다. 

```java
new MySQLContainer();
```  

그리고 `1.15` 버전부터 이미지 이름을 생성자로 전달할때는 `DockerImageName` 혹은 `RemoteDockerImage` 를 사용하는 것을 권장한다. 
문자열 이미지이름을 전달하는 것보다 보다 명시적으로 이미지의 이름과 태그를 사용할 수 있다.  

컨테이너를 생성할때 `@Rule`, `@ClassRule` 을 사용해서 테스트 환경에 맞게 구성할 수 있다. 
`@Rule` 의 경우 테스트 메소드 단위로 컨테이너가 생성 및 삭제되고, 
`@ClassRule` 은 테스트 클래스 단위로 컨테이너가 생성 및 삭제 된다. 

```java
public class CreateContainerEveryTest {
    public static final DockerImageName IMAGE = DockerImageName.parse("alpine:latest");

    @Rule
    public GenericContainer alpine = new GenericContainer(IMAGE)
            .withCommand("/bin/sh", "-c",
                    "while true; do echo \"is running\"; done");

    @Test
    public void checkCreated() {
        assertTrue(this.alpine.isCreated());
    }

    @Test
    public void checkRunning() {
        assertTrue(this.alpine.isRunning());
    }
}
```  

로그를 확인하면 컨테이너가 2번 실행되고 종료된 것을 확인 할 수 있다. 

```
[INFO ] GenericContainer          :  360 - Creating container for image: alpine:latest
[INFO ] GenericContainer          :  421 - Starting container with ID: 8b9158926351a74543443b991e188503efef23eec1fccc1e6437598a032ccbb0
[INFO ] GenericContainer          :  425 - Container alpine:latest is starting: 8b9158926351a74543443b991e188503efef23eec1fccc1e6437598a032ccbb0
[INFO ] GenericContainer          :  478 - Container alpine:latest started in PT5.513S
[INFO ] GenericContainer          :  360 - Creating container for image: alpine:latest
[INFO ] GenericContainer          :  421 - Starting container with ID: a189dab40c6a3f931e0dc79308c53ba1ac377a58a663742aae168d2ddac05da3
[INFO ] GenericContainer          :  425 - Container alpine:latest is starting: a189dab40c6a3f931e0dc79308c53ba1ac377a58a663742aae168d2ddac05da3
[INFO ] GenericContainer          :  478 - Container alpine:latest started in PT0.611S
```  

```java
public class CreateContainerOnceTest {
    public static final DockerImageName IMAGE = DockerImageName.parse("alpine:latest");

    @ClassRule
    public static GenericContainer alpine = new GenericContainer(IMAGE)
            .withCommand("/bin/sh", "-c",
                    "while true; do echo \"is running\"; done");

    @Test
    public void checkCreated() {
        assertTrue(alpine.isCreated());
    }

    @Test
    public void checkRunning() {
        assertTrue(alpine.isRunning());
    }
}
```  

로그를 확인하면 컨테이너 실행큰 테스트 실행 전 한번만 수행된다. 

```
[INFO ] GenericContainer          :  360 - Creating container for image: alpine:latest
[INFO ] GenericContainer          :  421 - Starting container with ID: cbdf2eb427281eefddd2bf4b751dc9e57c8557173fc037649185449627f30a46
[INFO ] GenericContainer          :  425 - Container alpine:latest is starting: cbdf2eb427281eefddd2bf4b751dc9e57c8557173fc037649185449627f30a46
[INFO ] GenericContainer          :  478 - Container alpine:latest started in PT5.572S
```  

## 컨테이너 통신
`TestContainers` 에서 실행 중인 컨테이너와 통신하기 위해서는 해당 컨테이너의 호스트명과 포트번호를 알아야 한다. 
그렇지 않다면 테스트에서 사용할 `Docker Network` 를 생성해서 통신하는 방법도 가능하다.  

먼저 컨테이너를 생성할때 `withExposePorts()` 메소드를 사용해서 컨테이너에서 오픈할 포트를 설정한다. 
그리고 `getMappedPort()` 메소드를 사용하거나, 
오픈한 포트가 한개인 경우 `getFirstMappedPort()` 를 사용해서 컨테이너에서 오픈한 포트와 포워딩된 호스트 포트를 얻을 수 있다.  

그리고 컨테이너의 호스트는 `getHost()` 메소드를 사용해서 얻을 수 있다.  

### 호스트와 통신하기


### 다른 컨테이너와 통신하기



























































---
## Reference
[Testcontainers](https://www.testcontainers.org/)  

