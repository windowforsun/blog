--- 
layout: single
classes: wide
title: "[Spring 실습] TestContainers"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '통합 테스트와 같이 테스트 코드가 추가적인 구성에 대한 의존성을 갖을 때 이를 해결할 수 있는 Testcontainer 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - TestContainers
    - Junit
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
new GenericContainer(DockerImageName.parse("alpine:latest"));
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
로컬에서 실행 중인 도커 데몬을 사용한다면 `localhost` 호스트 주소, 포트포워딩을 통해 컨테이너와 통신을 할 수 있다. 

```java
public class NetworkingToHostTest {
    public static final DockerImageName IMAGE = DockerImageName.parse("nginx:latest");

    @ClassRule
    public static GenericContainer nginx = new GenericContainer(IMAGE)
            .withExposedPorts(80);

    private RestTemplate restTemplate = new RestTemplate();
    private String url;

    @Before
    public void setUp() {
        this.url = "http://" + nginx.getHost() + ":";
    }

    @Test
    public void check_getFirstMappedPort() throws Exception {
        // given
        this.url += nginx.getFirstMappedPort();;

        // when
        String actual = this.restTemplate.getForObject(this.url, String.class);

        // then
        Assert.assertTrue(actual.contains("Welcome to nginx!"));
    }

    @Test
    public void check_getMappedPort() {
        // given
        this.url += nginx.getMappedPort(80);

        // when
        String actual = this.restTemplate.getForObject(this.url, String.class);

        // then
        Assert.assertTrue(actual.contains("Welcome to nginx!"));
    }
}
```  

예제 테스트는 `Nginx` 를 도커 컨테이너로 올리고 `RestTemplate` 을 사용해서 `Nginx` 의 기본 페이지 접속 테스트를 수행한다. 
`withExposedPorts()` 를 사용해서 컨테이너에 80 포트를 호스트 포트와 포워딩 한다. 
그리고 호스트와 포워딩된 실제 포트는 `getFirstMappedPort()`, `getMappedPort()` 메소드를 통해 값을 조회할 수 있다. 


### 다른 컨테이너와 통신하기
컨테이너간 통신을 해야할 경우 도커에서는 `Network` 를 생성하고 이를 각 컨테이너에 설정하는 방법으로 가능하다. 
`Testcontainer` 또한 동일한 방법으로 네트워크를 구성해서 구성한 컨테이너간 통신을 수행할 수 있다. 

```java
public class NetworkingToContainerTest {
    public static Network NETWORK = Network.newNetwork();
    public static String NGINX_HOST = "nginx";

    @ClassRule
    public static GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
            .withNetwork(NETWORK)
            .withNetworkAliases(NGINX_HOST);

    @ClassRule
    public static GenericContainer netshoot = new GenericContainer(DockerImageName.parse("nicolaka/netshoot"))
        .withNetwork(NETWORK)
        .withCommand("/bin/sh", "-c",
                "while true; do echo \"is running\"; done");

    @Test
    public void check_ContainerNetwork() throws Exception {
        // when
        Container.ExecResult actual = netshoot.execInContainer("/bin/bash", "-c", "curl http://" + NGINX_HOST + ":80");

        // then
        String stdOut = actual.getStdout();
        Assert.assertTrue(stdOut.contains("Welcome to nginx!"));
    }
}
```  

테스트에서 `Nginx` 와 `netshoot` 이라는 네트워크관련 라이브러리들이 설치된 컨테이너를 생성한다. 
그리고 `Network.newNetwork()` 로 도커 네트워크를 생성하고, 
요청을 받아야하는 `Nginx` 에는 `.withNetworkAliases()` 메소드로 호스트 이름으로 사용할 별칭을 지정한다. 
`netshoot` 컨테이너에서 `curl` 명령을 사용해서 `Nginx` 에 요청을 보내면 기본 페이지가 조회되는 것을 확인 할 수 있다.  


## 명령 실행
도커 컨테이너는 컨테이너 실행 시점(`CMD`, `ENTRYPOINT`)에 시행될 명령을 지정할 수 있고, 
컨테이너 실행 도중에도 명령을 `-it` 옵션으로 실행할 수 있다. 
또한 컨테이너에서 사용될 환경변수(`ENV`) 를 설정해 컨테이너 런타임 시점에 사용할 수 있다.  

먼저 컨테이너 실행 시점에 실행되는 명령어인 `CMD`, `ENTRYPOINT` 와 매칭되는 동작은 `.withCommand()` 메소드로 지정할 수 있다. 
그리고 컨테이너 실행 중 명령어 실행은 `.execInContainer()` 메소드로 가능하고, 환경 변수 지정은 `.withEnv()` 메소드로 설정 할 수 있다.  

```java
public class CommandTest {
    @Test
    public void executingCommand() throws Exception {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c",
                        "while true; do echo \"is running\"; done");
        alpine.start();
        alpine.execInContainer("/bin/sh", "-c", "echo \"hello\" >> newfile");

        // when
        Container.ExecResult actual = alpine.execInContainer("/bin/sh", "-c", "cat newfile");

        // then
        assertThat(actual.getStdout(), is("hello\n"));
        assertThat(actual.getExitCode(), is(0));
        alpine.stop();
    }

    @Test
    public void startUpCommand() throws Exception {
        // given
        GenericContainer redis = new GenericContainer(DockerImageName.parse("redis:latest"))
                .withCommand("redis-server --port 1234");
        redis.start();

        // when
        Container.ExecResult actual = redis.execInContainer("sh", "-c", "redis-cli -p 1234 info");

        // then
        assertThat(actual.getStdout(), containsString("tcp_port:1234"));
        assertThat(actual.getExitCode(), is(0));
        redis.stop();
    }


    @Test
    public void environment_EnvMap() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c",
                        "while true; do echo \"is running\"; done")
                .withEnv("HELLO", "WORLD");
        alpine.start();

        // when
        Map<String, String> actual = alpine.getEnvMap();

        // then
        assertThat(actual, hasEntry("HELLO", "WORLD"));
        alpine.stop();
    }

    @Test
    public void environment_Stdout() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c",
                        "while true; do echo \"is running\"; done")
                .withEnv("HELLO", "WORLD");
        alpine.start();

        // when
        Container.ExecResult actual = alpine.execInContainer("/bin/sh", "-c", "echo $HELLO");

        // then
        assertThat(actual.getStdout(), is("WORLD\n"));
        assertThat(actual.getExitCode(), is(0));
        alpine.stop();
    }
}
```  

## 파일 및 볼륨
도커에서는 호스트의 파일을 이미지 빌드시점에 `COPY`, `ADD` 로 복사하거나, 
컨테이너 실행시점에 `volume` 을 사용해서 마운트 할 수 있다. 
이미지 빌드시점은 나중에 설명하는 `Dockerfile` 부분에서 알아보고, 
컨테이너 실행 시점에 볼륨 마운트에 대해 알아본다.  

`Testcontainer` 에서는 `Java` 클래스경로와 호스트 경로에 대한 방법으로 볼륨 마운트를 지원한다. 
클래스경로로 지정은 `.withClasspathResourceMapping()` 메소드를 사용하고, 
호스트 경로에 대한 지정은 `.withCopyFileToContainer()` 메소드를 사용한다.  

```java
public class VolumeTest {
    @Test
    public void classPathResourceMapping() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c",
                        "while true; do echo \"is running\"; sleep 1; done")
                .withClasspathResourceMapping("host-file", "/host-file", BindMode.READ_ONLY);
        alpine.start();

        // when
        Container.ExecResult actual = alpine.execInContainer("/bin/sh", "-c", "cat /host-file");

        // then
        assertThat(actual.getStdout(), is("this is host file"));
        alpine.stop();
    }

    @Test
    public void copyFileToContainer() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c",
                        "while true; do echo \"is running\"; sleep 1; done")
                .withCopyFileToContainer(MountableFile.forClasspathResource("host-file"), "/host-file");
        alpine.start();

        // when
        Container.ExecResult actual = alpine.execInContainer("/bin/sh", "-c", "cat /host-file");

        // then
        assertThat(actual.getStdout(), is("this is host file"));
        alpine.stop();
    }
}
```  

프로젝트 `main` 디렉토리 하위에 있는 `resources` 에 아래 내용이 있는 `host-file` 이 있다. 

```
this is host file
```  

그리고 테스트에서 `.withClasspathResourceMapping()` 와 `.withCopyFileToContainer()` 를 사용해서 컨테이너에 마운트하고, 
내용을 조회하는 방법으로 테스트를 수행하고 있다.  


## 컨테이너 시작 및 준비 대기
컨테이너가 실행 되면서 초기에 실행이 되야하는 수행으로 실제 요청에 대한 처리나 불가할 수 있다. 
이런 상황에서 컨테이너 완전히 처리가능한 상태가 될때까지 대기하는 것을 `Wait Strategies` 라고 한다. 
그리고 데몬의 성격을 띄지 않고, 몇가지 수행을 하고 마치는 성격의 컨테이너의 경우에는 컨테이너가 종료될때 까지 기다려야 할 수도 있다. 
이런 상황에서 컨테이너가 실행되고 완전히 종료될 때까지 대기하는 것을 `Startup Startegies` 라고 한다.  

### Wait Strategies
컨테이너가 완전히 처리가능한 상태가 될때까지 대기는 `.waitingFor()` 메소드를 사용해서, 
상황에 따라 적절한 방법으로 구성할 수 있다. 
만약 웹 서버라면 `.forHttp()` 를 통해 특정 경로의 상태값 등을 사용할 수 있고, 
이미지를 빌드한 `Dockerfile` 에 `HEALTHCHECK` 가 있다면 `.forHealthcheck()` 로 지정가능하다. 
그리고 컨테이너가 실행되며 출력하는 특정 로그를 사용하는 `.forLogMessage()` 도 사용가능하다. 
또한 각 방법들은 타임아웃을 설정할 수 있다. 

```java
public class CheckContainerWaitingForTest {
    private RestTemplate restTemplate = new RestTemplate();

    @Test
    public void forHttp_Success() {
        // given
        String path = "/";
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHttp(path));
        nginx.start();
        String url = "http://" + nginx.getHost() + ":" + nginx.getFirstMappedPort();

        // when
        String actual = this.restTemplate.getForObject(url, String.class);

        // then
        assertThat(actual, containsString("Welcome to nginx!"));
    }

    @Test(expected = ContainerLaunchException.class)
    public void forHttp_Timeout_Status() {
        // given
        String path = "/";
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHttp(path).forStatusCode(111).withStartupTimeout(Duration.ofMillis(3000)));

        // when
        nginx.start();
    }

    @Test(expected = ContainerLaunchException.class)
    public void forHttp_Timeout_Path() {
        // given
        String path = "/timeout";
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHttp(path).withStartupTimeout(Duration.ofMillis(3000)));

        // when
        nginx.start();
    }

    @Test(expected = ContainerLaunchException.class)
    public void forHttp_Tls_Timeout() {
        // given
        String path = "/timeout";
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHttp(path).usingTls().withStartupTimeout(Duration.ofMillis(3000)));

        // when
        nginx.start();
    }

    @Test
    public void forHealthcheck_Success() {
        GenericContainer nginx = new GenericContainer(new ImageFromDockerfile()
                .withFileFromClasspath("Dockerfile", "/success-hc-dockerfile"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHealthcheck().withStartupTimeout(Duration.ofMillis(3000)));
        nginx.start();
        String url = "http://" + nginx.getHost() + ":" + nginx.getFirstMappedPort();

        // when
        String actual = this.restTemplate.getForObject(url, String.class);

        // then
        assertThat(actual, containsString("Welcome to nginx!"));
    }


    @Test(expected = ContainerLaunchException.class)
    public void forHealthcheck_Timeout() {
        GenericContainer nginx = new GenericContainer(new ImageFromDockerfile()
                .withFileFromClasspath("Dockerfile", "/fail-hc-dockerfile"))
                .withExposedPorts(80)
                .waitingFor(Wait.forHealthcheck().withStartupTimeout(Duration.ofMillis(3000)));

        // when
        nginx.start();
    }

    @Test
    public void forLogMessage_Success() {
        // given
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forLogMessage(".* Configuration complete; ready for start up.*", 1));
        nginx.start();
        String url = "http://" + nginx.getHost() + ":" + nginx.getFirstMappedPort();

        // when
        String actual = this.restTemplate.getForObject(url, String.class);

        // then
        assertThat(actual, containsString("Welcome to nginx!"));
    }

    @Test(expected = ContainerLaunchException.class)
    public void forLogMessage_Timeout() {
        // given
        GenericContainer nginx = new GenericContainer(DockerImageName.parse("nginx:latest"))
                .withExposedPorts(80)
                .waitingFor(Wait.forLogMessage("Configuration complete; ready for start up", 1)
                .withStartupTimeout(Duration.ofMillis(3000)));

        // when
        nginx.start();
    }
}
```  

`Dockerfile` 에 설정된 `HEALTHCHECK` 를 사용하는 `.forHealthcheck()` 테스트는 `success-hc-dockerfile` 과 `fail-hc-dockerfile` 을 직접 빌드해서 사용하고 그 내용은 아래와 같다. 
- `success-hc-dockerfile`

    ```dockerfile
    FROM nginx:latest
    
    HEALTHCHECK --interval=2s --timeout=2s --retries=3 \
        CMD curl -f http://localhost
    ```  

- `fail-hc-dockerfile`

    ```dockerfile
    FROM nginx:latest
    
    HEALTHCHECK --interval=2s --timeout=2s --retries=3 \
        CMD curl -f http://localhost/fail
    ```  

### Startup Strategies
컨테이너가 실행되고 종료하기 까지 대기하는 것은 `.withStartupCheckStrategy()` 를 사용해서 상황에 맞게 설정할 수 있다. 
`IsRunningStartupStrategy` 는 기본으로 설정되는 전략이다. 
그리고 `OneShotStartupCheckStrategy` 는 실행되고 종료되는 컨테이너를 위한 전략으로 종료는 `exit code 0` 으로 판별한다. 
`IndefiniteWaitOneShotStartupCheckStrategy` 는 컨테이너가 실행되고 종료되는 타임아웃을 지정하지 않는 전략으로, 
컨테이너 수행 시간이 길고 예측할 수 없는 경우에 사용할 수 있다. 
`MinimumDurationRunningStartupCheckStrategy` 는 컨테이너가 실행되고 종료되는 최소 시간을 지정할 수 있는 전략이다.   

```java
public class CheckContainerStartUpTest {
    @Test
    public void isRunningStartupCheckStrategy_Success() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withStartupCheckStrategy(new IsRunningStartupCheckStrategy());

        alpine.start();
    }

    @Test
    public void oneShotStartupCheckStrategy_Success() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "echo \"is running\"")
                .withStartupCheckStrategy(
                        new OneShotStartupCheckStrategy().withTimeout(Duration.ofSeconds(3))
                );

        alpine.start();
    }

    @Test(expected = ContainerLaunchException.class)
    public void oneShotStartupCheckStrategy_Timeout() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "while true; do echo \"is running\"; sleep 1; done")
                .withStartupCheckStrategy(
                        new OneShotStartupCheckStrategy().withTimeout(Duration.ofSeconds(3))
                );

        alpine.start();
    }

    @Test
    public void indefiniteWaitOneShotStartupCheckStrategy() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "sleep 5; echo hello")
                .withStartupCheckStrategy(
                        new IndefiniteWaitOneShotStartupCheckStrategy()
                );

        alpine.start();
    }


    @Test
    public void minimumDurationRunningStartupCheckStrategy_Success() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "sleep 10; echo \"is running\"")
                .withStartupCheckStrategy(
                        new MinimumDurationRunningStartupCheckStrategy(Duration.ofSeconds(2))
                );

        alpine.start();
    }

    @Test(expected = ContainerLaunchException.class)
    public void minimumDurationRunningStartupCheckStrategy_Fail() {
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "echo \"is running\"")
                .withStartupCheckStrategy(
                        new MinimumDurationRunningStartupCheckStrategy(Duration.ofSeconds(3))
                );

        alpine.start();
    }
}
```  

## 컨테이너 로그
기본적으로 컨테이너에서 생성하는 로그는 `getLogs()` 메소드로 그 시점까지 로그를 얻을 수 있다. 
그리고 `followOutput()` 메소드와 `Slf4j` 를 사용해서 컨테이너 로그를 콘솔 로그나 다른 곳으로 남길 수도 있다. 

```java
public class ContainerLogTest {
    @Test
    public void simpleLog() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "sleep 3; echo \"is stdout\"; >&2 echo \"is stderr\"; " +
                        "sleep 3; echo \"not contain\"");

        // when
        alpine.start();
        Thread.sleep(3500);

        // then
        String allLog = alpine.getLogs();
        String stdoutLog = alpine.getLogs(OutputFrame.OutputType.STDOUT);
        String stderrLog = alpine.getLogs(OutputFrame.OutputType.STDERR);
        assertThat(allLog, allOf(containsString("is stdout"),
                containsString("is stderr"),
                not(containsString("not contain"))));
        assertThat(stdoutLog, allOf(containsString("is stdout"),
                not(containsString("not contain"))));
        assertThat(stderrLog, containsString("is stderr"));
    }

    @Test
    public void streamLog() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "while true; do echo \"is running\"; sleep 1; done");
        Logger logger = LoggerFactory.getLogger("container-logger");
        Slf4jLogConsumer logConsumer = new Slf4jLogConsumer(logger);

        // when
        alpine.start();
        alpine.followOutput(logConsumer);
        Thread.sleep(3500);

        // then
        // Container logs will print on console
    }

    @Test
    public void toStringConsumer() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "while true; do echo \"is running\"; sleep 1; done");
        ToStringConsumer toStringConsumer = new ToStringConsumer();

        // when
        alpine.start();
        alpine.followOutput(toStringConsumer);
        Thread.sleep(3500);

        // then
        String utf8String = toStringConsumer.toUtf8String();
        assertThat(utf8String, containsString("is running"));
    }

    @Test
    public void waitingConsumer() throws Exception {
        // given
        GenericContainer alpine = new GenericContainer(DockerImageName.parse("alpine:latest"))
                .withCommand("/bin/sh", "-c", "echo \"start\"; while true; do echo \"is running\"; sleep 1; done");
        WaitingConsumer waitingConsumer = new WaitingConsumer();
        ToStringConsumer toStringConsumer = new ToStringConsumer();

        // when
        alpine.start();
        Consumer<OutputFrame> composedConsumer = toStringConsumer.andThen(waitingConsumer);
        alpine.followOutput(composedConsumer);
        waitingConsumer.waitUntil(outputFrame ->
            outputFrame.getUtf8String().contains("is running"), 3, TimeUnit.SECONDS);

        // then
        String utf8String = toStringConsumer.toUtf8String();
        assertThat(utf8String, allOf(
                containsString("start"),
                containsString("is running")
        ));
    }
}
```  

## 커스텀 이미지 빌드(Dockerfile)
필요에 따라 미리 빌드된 이미지를 사용하지 않고, 
테스트 시점에 필요한 이미지를 빌드해서 사용해야할 수도 있다. 
`Testcontainer` 는 이를 `ImageFromDockerfile` 을 통해 `Dockerfile` 과 동일한 방식으로 이미지를 빌드할 수 있도록 제공한다. 
실제 파일로 작성된 `Dockerfile` 을 사용해서 빌드를 수행할 수도 있고, 
`DockerfileBuilder` 에서 제공하는 메소드를 사용해서, `Dockerfile` 을 코드상에서 만들어 사용할 수도 있다. 

```java
public class BuildImageTest {
    @Test
    public void imageFromDockerfile_Dockerfile() throws Exception {
        // given
        GenericContainer custom = new GenericContainer(
                new ImageFromDockerfile()
                        .withFileFromString("/from-string", "this is from string")
                        .withFileFromClasspath("Dockerfile", "/test-dockerfile")
        ).withCommand("/bin/sh", "-c", "while true; do echo \"is running\"; sleep 1; done");

        // when
        custom.start();

        // then
        Container.ExecResult fromStringContent = custom.execInContainer("/bin/sh", "-c", "cat /from-string");
        Container.ExecResult dockerFileContent = custom.execInContainer("/bin/sh", "-c", "cat /dockerfile-file");
        assertThat(fromStringContent.getStdout(), is("this is from string"));
        assertThat(dockerFileContent.getStdout(), is("this is dockerfile\n"));
    }

    @Test
    public void imageFromDockerfile_DSL() throws Exception {
        // given
        GenericContainer custom = new GenericContainer(
                new ImageFromDockerfile()
                        .withDockerfileFromBuilder(dockerfileBuilder -> {
                            dockerfileBuilder
                                    .from("alpine:latest")
                                    .copy("from-string", "/from-string")
                                    .run("echo \"this is dockerfile\" >> dockerfile-file");
                        })
                        .withFileFromString("/from-string", "this is from string")
        ).withCommand("/bin/sh", "-c", "while true; do echo \"is running\"; sleep 1; done");

        // when
        custom.start();

        // then
        Container.ExecResult fromStringContent = custom.execInContainer("/bin/sh", "-c", "cat /from-string");
        Container.ExecResult dockerFileContent = custom.execInContainer("/bin/sh", "-c", "cat /dockerfile-file");
        assertThat(fromStringContent.getStdout(), is("this is from string"));
        assertThat(dockerFileContent.getStdout(), is("this is dockerfile\n"));
    }
}
```  

`.withFileFromString()` 메소드는 도커 빌드 컨텍스트에 문자열을 통해 파일을 쓰는 동작을 수행한다. 
그리고 `.withFileFromClasspath()` 메소드는 `Java` 클래스패스에 있는파일을 도커 빌드 컨텍스트로 복사하는 명령이다.  

현재 클래스패스 하위에 `test-dockerfile` 내용은 아래와 같다. 

```dockerfile
FROM alpine:latest

RUN echo "this is dockerfile" >> /dockerfile-file
COPY from-string /from-string
```  


---
## Reference
[Testcontainers](https://www.testcontainers.org/)  

