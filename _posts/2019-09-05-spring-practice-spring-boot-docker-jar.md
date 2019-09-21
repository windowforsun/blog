--- 
layout: single
classes: wide
title: "[Docker 실습] Spring Boot Jar Local 환경 구성"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 를 통해 Spring Boot 프로젝트를 빌드 및 실행 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Practice
    - Spring
    - Redis
    - SpringBoot
---  

# 환경
- Docker
- Spring Boot
- Maven
- Redis

# 목표
- Docker 를 사용해서 Spring Boot Local 환경을 구성한다.
- Spring Boot 는 Embedded Tomcat 을 사용하고 jar 로 패키징 한다.
- 빌드 및 패키지는 Maven 을 사용하고 이또한 Docker 에서 수행할 수 있도록 구성한다.

# 방법
- Spring Boot, Redis 로 구성된 총 2개의 Docker Container 를 사용한다.
- Dockerfile 에는 각 Container 에 대한 설정을 기술한다.
- docker-compose 를 통해 각 Container 를 연동 및 빌드 한다.

# 예제
- 예제의 전체 디렉토리 구조는 아래와 같다.

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-jar-2.png)

## 간단한 Spring Boot 프로젝트 만들기
### 프로젝트 구조

![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-jar-1.png)

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-docker-jar-2</name>
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
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
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

### application.yml

```yaml
spring:
  redis:
    host: redis
    port: 6377
```  

### HelloController

```java
@RestController
public class HelloController {
    @GetMapping("/")
    public String root() {
        return "hello ~ spring boot docker jar";
    }

    @GetMapping("/echo/{str}")
    public String echoGet(@PathVariable String str){
        return str;
    }

    @PostMapping("/echo")
    public String echoPost(@RequestBody String str) {
        return str;
    }
}
```  

### RedisConfig

```java
@Configuration
public class RedisConfig {
    @Value("${spring.redis.port}")
    private int redisPort;
    @Value("${spring.redis.host}")
    private String redisHost;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(this.redisHost);
        redisStandaloneConfiguration.setPort(this.redisPort);

        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);
        return lettuceConnectionFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(this.redisConnectionFactory());
        redisTemplate.setDefaultSerializer(new StringRedisSerializer());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        return redisTemplate;
    }
}
```  

### Data

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@RedisHash("Data")
public class Data {
    @Id
    private long id;
    @Indexed
    private long firstIndex;
    @Indexed
    private long secondIndex;
    private String str;
}
```  

### DataRepository

```java
@Repository
public interface DataRepository extends CrudRepository<Data, Long> {
}
```  

### DataController

```java
@RestController
@RequestMapping("/data")
public class DataController {
    @Autowired
    private DataRepository dataRepository;

    @PostConstruct
    public void setUp() {
        this.dataRepository.save(
                Data.builder()
                        .id(1)
                        .str("1")
                        .firstIndex(1)
                        .secondIndex(1)
                        .build()
        );
    }

    @PostMapping
    public Data create(@RequestBody Data data) {
        return this.dataRepository.save(data);
    }

    @PutMapping("/{id}")
    public Data update(@PathVariable long id, @RequestBody Data data) {
        Data result = null;

        if(this.dataRepository.existsById(id)) {
            result = this.dataRepository.save(data);
        }

        return result;
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable long id) {
        this.dataRepository.deleteById(id);
    }

    @GetMapping("/{id}")
    public Data read(@PathVariable long id) {
        return this.dataRepository.findById(id).orElse(null);
    }
}
```  

## Docker Container 구성하기

![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-jar-2.png)

- `docker/redis` 디렉토리는 Redis 관련 Docker Container 구성 및 Redis 설정 파일이 있다.
- `docker/web` 디렉토리는 Maven 기반 Spring Boot 프로젝트 빌드 및 실행하는 Container 구성 파일이 있다.
- `docker/docker-compose.yml` 은 위 2개 Container 를 빌드 및 구동 시키고, Container 관련 설정이 기술되어 있다.

### Web Container
- Web Container 는 Spring Boot 프로젝트를 빌드 및 실행하는 Container 이다.
- `docker/web/Dockerfile`

	```
	### BUILD image
	FROM maven:3-jdk-11 as builder
	# create app folder for sources
	RUN mkdir -p /build
	WORKDIR /build
	COPY pom.xml /build
	#Download all required dependencies into one layer
	RUN mvn -B dependency:resolve dependency:resolve-plugins
	#Copy source code
	COPY src /build/src
	# Build application
	#RUN mvn package
	RUN mvn package -DskipTests	
	
	### Rumtime Image
	FROM openjdk:11-slim as runtime
	EXPOSE 8080
	#Set app home folder
	ENV APP_HOME /app
	#Possibility to set JVM options (https://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
	ENV JAVA_OPTS=""
	#Create base app folder
	RUN mkdir $APP_HOME
	#Create folder to save configuration files
	RUN mkdir $APP_HOME/config
	#Create folder with application logs
	RUN mkdir $APP_HOME/log
	VOLUME $APP_HOME/log
	VOLUME $APP_HOME/config
	WORKDIR $APP_HOME
	#Copy executable jar file from the builder image
	COPY --from=builder /build/target/*.jar app.jar
	ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar" ]
	#Second option using shell form:
	#ENTRYPOINT exec java $JAVA_OPTS -jar app.jar $0 $@
	```  
	
- Web Container 는 Maven 을 통해 빌드하는 이미지와 빌드된 이미지를 Java 명령어를 통해 실행시키는 이미지로 구성되어 있다.
- Maven 빌드시 Maven 설정을 변경해야 할경우 `FROM` 아래부분에 아래 코드를 추가 해준다.

	```
	#Copy Custom Maven settings
	COPY settings.xml /root/.m2/
	```  
	
	- `settings.xml` 파일에 Maven 설정이 기술한다.
	
### Redis Container
- `docker/redis/Dockerfile`

	```
	FROM redis:latest
	
	COPY ./redis/redis.conf /usr/local/etc/redis/redis.conf
	CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
	```  
	
- `redis.conf` 파일은 [여기](http://download.redis.io/redis-stable/redis.conf) 에서 받아 수정해 사용한다.


### docker-compose

```yaml
version: '3.7'

services:
  web:
    build:
      context: ./../
      dockerfile: docker/web/Dockerfile
    ports:
      - "8080:8080"
    networks:
      - web-redis
    depends_on:
      - redis
    restart: on-failure
  redis:
    build:
      context: .
      dockerfile: redis/Dockerfile
    ports:
      - "6377:6377"
    networks:
      - web-redis
    volumes:
      - "./../redis-data:/data"

networks:
  web-redis:
```  

- Web Container 는 `docker/web/Dockerfile` 을 통해 빌드하고, Redis Container 는 `docker/redis/Dockerfile` 을 통해 빌드한다.
- Web Container 와 Redis Container 에 대한 network 는 `web-redis` 를 사용한다.
- Redis 의 데이터는 Docker 를 실행하는 머신에서 `redis-data` 의 디렉토리에 저장해 영속성을 보장하도록 한다.
- Web Container 는 8080 포트, Redis Container 는 6377 포트를 호스트 머신과 포트포워딩 한다.

## 실행
- `docker` 디렉토리로 이동해서 `docker-compose up --build` 명령어를 수행하면 `docker-compose` 파일을 기반으로 Container 를 빌드한다.
	
	```
	$ docker-compose up --build
	Found orphan containers (docker_jar-build_1) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
	Building redis
	Step 1/3 : FROM redis:latest
	 ---> 63130206b0fa
	Step 2/3 : COPY ./redis/redis.conf /usr/local/etc/redis/redis.conf
	 ---> Using cache
	 ---> b3e6abca2cfb
	Step 3/3 : CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
	 ---> Using cache
	 ---> 95c3cc96d39d
	
	Successfully built 95c3cc96d39d
	Successfully tagged docker_redis:latest
	Building web
	Step 1/19 : FROM maven:3-jdk-11 as builder
	 ---> e941463218b9
	Step 2/19 : RUN mkdir -p /build
	 ---> Using cache
	 ---> e207dfbe6f8c
	Step 3/19 : WORKDIR /build
	 ---> Using cache
	 ---> 2504a5ec952c
	Step 4/19 : COPY pom.xml /build
	 ---> Using cache
	 ---> 22e0ce6a0009
	Step 5/19 : RUN mvn -B dependency:resolve dependency:resolve-plugins
	 ---> Using cache
	 ---> dabc47659eb6
	Step 6/19 : COPY src /build/src
	 ---> 8c4f05bfd5e0
	Step 7/19 : RUN mvn package -DskipTests
	 ---> Running in 2e80f121f932

	생략 ...
	
	redis_1  | 1:C 06 Sep 2019 10:05:54.915 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
    redis_1  | 1:C 06 Sep 2019 10:05:54.915 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=1, just started
    redis_1  | 1:C 06 Sep 2019 10:05:54.915 # Configuration loaded
    redis_1  |                 _._
    redis_1  |            _.-``__ ''-._
    redis_1  |       _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
    redis_1  |   .-`` .-```.  ```\/    _.,_ ''-._
    redis_1  |  (    '      ,       .-`  | `,    )     Running in standalone mode
    redis_1  |  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6377
    redis_1  |  |    `-._   `._    /     _.-'    |     PID: 1
    redis_1  |   `-._    `-._  `-./  _.-'    _.-'
    redis_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|
    redis_1  |  |    `-._`-._        _.-'_.-'    |           http://redis.io
    redis_1  |   `-._    `-._`-.__.-'_.-'    _.-'
    redis_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|
    redis_1  |  |    `-._`-._        _.-'_.-'    |
    redis_1  |   `-._    `-._`-.__.-'_.-'    _.-'
    redis_1  |       `-._    `-.__.-'    _.-'
    redis_1  |           `-._        _.-'
    redis_1  |               `-.__.-'
    redis_1  |
    redis_1  | 1:M 06 Sep 2019 10:05:54.918 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
    redis_1  | 1:M 06 Sep 2019 10:05:54.918 # Server initialized
    redis_1  | 1:M 06 Sep 2019 10:05:54.918 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
    redis_1  | 1:M 06 Sep 2019 10:05:54.919 * DB loaded from disk: 0.002 seconds
    redis_1  | 1:M 06 Sep 2019 10:05:54.919 * Ready to accept connections
    web_1    |
    web_1    |   .   ____          _            __ _ _
    web_1    |  /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
    web_1    | ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
    web_1    |  \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
    web_1    |   '  |____| .__|_| |_|_| |_\__, | / / / /
    web_1    |  =========|_|==============|___/=/_/_/_/
    web_1    |  :: Spring Boot ::        (v2.1.8.RELEASE)
    web_1    |
    web_1    | 2019-09-06 10:05:59.000  INFO 6 --- [           main] c.e.d.SpringBootDockerJar2Application    : Starting SpringBootDockerJar2Application v0.0.1-SNAPSHOT on 3a46b1aa18c0 with PID 6 (/app/app.jar started by root in /app)
    web_1    | 2019-09-06 10:05:59.016  INFO 6 --- [           main] c.e.d.SpringBootDockerJar2Application    : No active profile set, falling back to default profiles: default

	생략 ..
	```  
	
	- 출력 결과는 Docker Image 다운, Container 설정, Spring Boot 빌드에 따라 상이할수 있다.

- 실행 결과는 아래와 같다.

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-jar-4.png)

	![그림 1]({{site.baseurl}}/img/docker/practice-spring-boot-docker-jar-5.png)




---
## Reference
[Spring Boot: Run and Build in Docker](https://dzone.com/articles/spring-boot-run-and-build-in-docker)   
[Docker Compose wait for container X before starting Y](https://stackoverflow.com/questions/31746182/docker-compose-wait-for-container-x-before-starting-y)   
[Control startup and shutdown order in Compose](https://docs.docker.com/compose/startup-order/)   
[Dockerize a Spring Boot application](https://thepracticaldeveloper.com/2017/12/11/dockerize-spring-boot/)   
[IntelliJ 에서 JAR 만들기](https://www.hyoyoung.net/100)   
[[IntelliJ] 실행 가능한 Jar 파일 생성 시 'xxx.jar에 기본 Manifest 속성이 없습니다.' 오류증상](http://1004lucifer.blogspot.com/2016/01/intellij-jar-xxxjar-manifest.html)   
[docker-compose - ADD failed: Forbidden path outside the build context](https://stackoverflow.com/questions/54287298/docker-compose-add-failed-forbidden-path-outside-the-build-context)   
[redis - Docker Hub](https://hub.docker.com/_/redis)   
[redis - Docker Hub](https://hub.docker.com/_/redis)   
[Setting up a Redis test environment using Docker Compose](https://cheesyprogrammer.com/2018/01/04/setting-up-a-redis-test-environment-using-docker-compose/)   
