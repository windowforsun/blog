--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Spring Session"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Spring Session 을 저장하고 관리하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Spring Session
    - Redis
---  

# 목표
- Spring Session 으로 Session 을 관리한다.
- Spring Session 에 대한 정보를 메모리가 아닌, Redis 에 저장한다.

# 방법
- Spring Boot 에서 Redis 를 사용하는 환경을 만들어준다.
- Spring Session 의 저장소를 Redis 로 설정해준다.
- Redis 를 Session 저장소로 사용할 경우 서버가 다운되더라도 세션 정보는 유지된다.

## Spring Session
- HttpSession
	- 웹 컨테이너에 의존적이지 않으면서 헤더에 세션 아이디를 제공한다.
- WebSocket
	- WebSocket 메세지로 HttpSession 을 유지할 수 있다.
- WebSession
	- SpringWebFlux 의 WebSession 을 애플리케이션 컨테이터를 의존하지 않는 방법으로 제공한다.

## Spring Session Clustering
- 웹서비스를 제공하는 기본적인 Client-Server 의 구조는 아래와 같다.
	- 기본적인 구조에서 Session 정보는 Server Memory 상에 저장되어 있다.
	
	![그림 1]({{site.baseurl}}/img/spring/practice-springbootspringsessionredis-1.png)

- Load Balance 를 사용해서 서버를 증설하게 되면 아래와 같은 구조가 된다.
	- 아래 구조는 A 클라이언트의 세션이 T서버에 있을 때 S서버에서는 T서버에 있는 A 클라이언트 세션의 정보를 알 수 없고, 두 세션은 일치하지 않는다.
	
	![그림 2]({{site.baseurl}}/img/spring/practice-springbootspringsessionredis-2.png)
	
- 위처럼 서버가 증설된 상태에서 세션을 공유하는 방법 중 하나는 Redis 를 세션의 저장소로 사용하는 것이다.

	![그림 3]({{site.baseurl}}/img/spring/practice-springbootspringsessionredis-3.png)
	
- Redis 와 같은 공용 세션 저장소를 사용하지 않고, Server 간 Session 을 주고받을 수도 있다.
- 주의 해야할 점은 트래픽 증가로 인한 Server 의 확장과 저장소의 확장은 비례하지 않을 수 있다. 
- Redis Session 저장소 증설을 위해 Clustering 을 하면 아래 구조처럼 가능하다.

	![그림 4]({{site.baseurl}}/img/spring/practice-springbootspringsessionredis-4.png)
	
# 예제
## 프로젝트 구조

![그림 5]({{site.baseurl}}/img/spring/practice-springbootspringsessionredis-5.png)

## pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <packaging>jar</packaging>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-spring-session</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>RELEASE</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```  

## Spring Session Redis 설정하기
- Java 설정 파일을 통해 설정하는 방법은 아래와 같다.

	```java
	@Configuration
	@EnableRedisHttpSession
	public class RedisSessionConfiguration {
	
	}
	```  
	
- `application.properties` 또는 `application.yaml` 를 통해서도 설정 가능하다.

	```yaml
	spring:
	  # Redis 접속 설정
      redis:
        host: localhost
        port: 6379
        timeout: 1m
      # Sping Session 설정
      session:
        store-type: redis
        redis:
          namespace: windowforsun:session
    
    # Session timeout 시간
    server:
      servlet:
        session:
          timeout: 20s
	```  
	
- `spring.session.store-type` 으로 Spring Session 저장소 선택 가능하다.
	- JDBC
	- Redis
	- Hazelcast
	- MongoDB
- `spring.session.redis.namespace` 를 통해 Redis 에 Session 이 저장될때 앞에 붙은 Prefix 와 같은 문자열을 설정할 수 있다.
	- 기본값은 spring:session 이다.
- `server.servlet.session.timeout` 으로 세션의 타임아웃 시간을 설정할 수 있다.
	- s : 초
	- m : 분
	- h : 시
	
## Controller

```java
@RestController
public class SampleController {
    private HttpSession httpSession;

    @Autowired
    public SampleController(HttpSession httpSession) {
        this.httpSession = httpSession;
    }

    @GetMapping
    public String getUid() {
        return this.httpSession.getId();
    }
}
```  

- `HttpSession` 객체는 객체 변수로 `@Autowired` 가능하고 아래 처럼 메서드의 매개변수로도 가능하다.

	```java	
    @GetMapping
    public String getUid(@Autowired HttpSession httpSession) {
        httpSession.setMaxInactiveInterval(this.sessionTimeout);
        return this.httpSession.getId();
    }
	```  

## 테스트 코드

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)	// TestRestTemplate
public class SampleControllerTest_RestTemplate {
    private String url = "http://localhost:8080/";
    @Value("${server.servlet.session.timeout}")
    private String sessionTimeout;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void testGetSessionUid() {
        URI uri = URI.create(this.url);
        ResponseEntity<String> result = this.doRequest(restTemplate, uri);
        String sessionUid = result.getBody();

        Assert.assertTrue(sessionUid.length() > 0);
    }

    @Test
    public void testGetSessionUid_Twice_SameSessionUid() {
        URI uri = URI.create(this.url);
        ResponseEntity<String> result = this.doRequest(restTemplate, uri);
        String sessionUid1 = result.getBody();
        String cookie = result.getHeaders().getFirst("Set-Cookie");

        result = this.doRequest(restTemplate, uri, cookie);
        String sessionUid2 = result.getBody();

        Assert.assertTrue(sessionUid1.length() > 0);
        Assert.assertTrue(sessionUid2.length() > 0);
        Assert.assertEquals(sessionUid1, sessionUid2);
    }

    @Test
    public void testGetSessionUid_Twice_NotSameSessionUid() {
        URI uri = URI.create(this.url);
        ResponseEntity<String> result = this.doRequest(restTemplate, uri);
        String sessionUid1 = result.getBody();

        result = this.doRequest(restTemplate, uri);
        String sessionUid2 = result.getBody();

        Assert.assertTrue(sessionUid1.length() > 0);
        Assert.assertTrue(sessionUid2.length() > 0);
        Assert.assertNotEquals(sessionUid1, sessionUid2);
    }

    @Test
    public void testSessionTimeout_Reset() throws Exception{
        URI uri = URI.create(this.url);
        ResponseEntity<String> result = this.doRequest(restTemplate, uri);
        String sessionUid1 = result.getBody();
        String cookie = result.getHeaders().getFirst("Set-Cookie");

        int timeout = Integer.parseInt(this.sessionTimeout.substring(0, this.sessionTimeout.length() - 1));
        Thread.sleep(timeout * 1010l);

        result = this.doRequest(restTemplate, uri, cookie);
        String sessionUid2 = result.getBody();

        Assert.assertTrue(sessionUid1.length() > 0);
        Assert.assertTrue(sessionUid2.length() > 0);
        Assert.assertNotEquals(sessionUid1, sessionUid2);
    }

    private ResponseEntity<String> doRequest(TestRestTemplate restTemplate, URI uri) {
        HttpHeaders headers = new HttpHeaders();
        RequestEntity<Object> request = new RequestEntity<>(headers, HttpMethod.GET, uri);

        return restTemplate.exchange(request, String.class);
    }

    private ResponseEntity<String> doRequest(TestRestTemplate restTemplate, URI uri, String cookie) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Cookie", cookie);
        RequestEntity<Object> request = new RequestEntity<>(headers, HttpMethod.GET, uri);

        return restTemplate.exchange(request, String.class);
    }
}
```  

- 테스트를 위해 `TestRestTemplate` 를 사용했고, `exchange` 메서드에 쿠기정보를 담은 헤더를 넣어 요청을 수행하고 있다.
- `exchange` 메서드를 사용하고 있기 때문에 테스트를 위해서는 `url` 에 해당되는 서버를 켜두어야 테스트가 가능하다.


---
## Reference
[Spring boot session example using redis](https://javadeveloperzone.com/spring-boot/spring-boot-session-example-using-redis/)   
[Guide to Spring Session](https://www.javadevjournal.com/spring/spring-session/)   
[Spring Boot + Session Management Hello World Example](https://www.javainuse.com/spring/springboot_session)   
[Spring Boot - Redis를 활용한 Session Clustering](https://heowc.tistory.com/30)   
[spring boot session redis and nginx](http://wonwoo.ml/index.php/post/960)   
[스프링 세션](http://arahansa.github.io/docs_spring/session.html)   
[Spring Session Design Pattern](https://brunch.co.kr/@springboot/114)   
