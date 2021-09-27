--- 
layout: single
classes: wide
title: "[Spring 개념] "
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
    - Spring Boot
    - Spring Data MongoDB
    - 
toc: true
use_math: true
---  

## Spring Data MongoDB
`Spring Boot` 에서 `MongoDB` 의 연동은 `Spring Data MongoDB` 의존성을 사용하는 방법이 있다. 
`Spring Data MongoDB` 의존성에는 `MongoTemplate`, `MongoRepository` 두 가지 방법으로 사용할 수 있는데 그 설명은 아래와 같다.  

구분|설정
---|---
MongoTemplate|`Spring` 의 표준 템플릿 패턴을 따르고 기본적인 `Persistence Engine` 을 즉시 사용할 수 있는 `API` 제공한다. (모델 클래스 정의 불필요)
MongoRepository|`Spring Data` 중심 접근 방식을 따르고, 모든 `Spring Data` 프로젝트에서 일반적으로 사용되는 패턴을 기반으로 한다. `POJO` 를 기반으로 좀 더 유연하고 복잡한 API 를 제공하낟.  

어느 하나만 고정해서 사용하기 보다는 각 특성을 파악하고 목적에 알맞는 것을 선택해서 사용하는 것이 좋을 것 같다.

### 의존성 추가
`Spring Data MongoDB` 사용을 위해서 아래 의존성을 `build.gradle` 에 추가가 필요하다.  

```groovy
plugins {
	id 'io.spring.dependency-management' version '1.0.8.RELEASE'
	id 'org.springframework.boot' version '2.4.2'
	id 'java'
}

dependencies {
	// for Spring Data MongoDB
	implementation group: 'org.springframework.boot', name: 'spring-boot-starter-data-mongodb'
	implementation group: 'org.springframework.boot', name: 'spring-boot-starter-test'
}
```  

### MongoDB 테스트 환경
`Spring Data MongoDB` 를 연동하고 테스트하기 위해서는 `MongoDB` 에 대한 테스트 환경이 필요하다. 
간단하게 `MongoDB` 테스트 환경을 구성할 수 있는 2가지 방법에 대해서 알아본다.  

모두 `Docker` 를 기반으로 `MongoDB` 를 구동시키는 방법인데 약간에 차이가 있어서 나눠서 설명한다.   

#### docker-compose.yaml 사용
`docker-compose.yaml` 을 사용하는 방법은 로컬에 테스트 용도로 사용할 `MongoDB` 컨테이너 한개를 구동 시켜놓은 채로 계속해서 사용하는 방법이다.  
프로젝트 루트 경로에 `docker-compose.yaml` 이름과 아래와 같은 내용으로 파일을 생성한다.  

```yaml
version: '3.7'

services:
  mongodb:
    image: mongo:4.4.4
    ports:
      - "27017:27017"
```  

계정정보 등은 `environment` 를 사용해서 설정할 수 있지만 간단한 구성을 위해서 아무 설정도 하지 않은 상태이다. 
그리고 아래 명령어로 실행하면 로컬 `27017` 포트로 접속할 수 있는 `MongoDB` 컨테이너가 실행 된다.  

```bash
$ docker-compose up -d
or
$ docker-compose up --build
Creating intro-mongodb_mongodb_1 ... done
Attaching to intro-mongodb_mongodb_1

.. 생략 ..
```  

#### TestContainers 사용
`TestContainers` 를 사용하는 방법은 로컬에 계속해서 테스트용으로 사용할 `MongoDB` 컨테이너를 실행해 두는 것이 아니라, 
테스트 코드가 수행되기 전에 사용할 `MongoDB` 컨테이너를 실행하고 테스트가 완료되면 컨테이너를 종료하는 방법이다. 
말 그대로 테스트 수행을 위한 환경을 `Docker` 를 기반으로 구성할 수 있다.  

`TestContainers` 를 사용하기 위해서는 아래와 같은 의존성을 `build.gradle` 에 추가가 필요하다.  

```groovy
dependencies {
	compile "org.testcontainers:mongodb:1.15.2"
    testCompile "org.testcontainers:mongodb:1.15.2"
	compile "org.testcontainers:junit-jupiter:1.15.2"
    testCompile "org.testcontainers:junit-jupiter:1.15.2"
}
```  

그리고 `TestContainers` 를 사용해서 `MongoDB` 컨테이너를 실행하는 클래스는 아래와 같다.  

```java
@ActiveProfiles("test")
@Testcontainers
@Slf4j
public class MongoDbTest {
    @Container
    public static MongoDBContainer MONGO_DB = new MongoDBContainer(
            DockerImageName.parse("mongo:4.4.4")
    );

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        // yaml 쪽에 설정이 필요한 경우 
//        registry.add("spring.data.mongodb.host", MONGO_DB::getHost);
//        registry.add("spring.data.mongodb.port", MONGO_DB::getFirstMappedPort);
        log.info("mongodb test container info host : {}, port : {}", MONGO_DB.getHost(), MONGO_DB.getFirstMappedPort());
    }
}
```  

이후 테스트 클래스에서 `MongoDbTest` 클래스를 상속 받아 테스트를 진행하는 방법으로 사용된다.  

```java

@SpringBootTest
@ExtendWith(SpringExtension.class)
public class SomeMongoDbTest extends MongoDbTest {
    // ...
}
```  

### Spring Data MongoDB 설정
`Spring Boot` 프로젝트에서 `Spring Data MongoDB` 를 설정하는 방법 중 `Java Configuration` 을 사용해서 설정하는 방법과 `application.yaml` 을 통해 설정하는 방법에 대해서 알아본다.  

#### Java Configuration
`Java Config` 로 설정하는 예시는 `MongoClient` 와 `MongoTemplate` 에 해당하는 빈을 선언해 설정하는 방식이다.  

```java
@Configuration
public class MongoConfig {
    // ex) mongodb://<username>:<password>@<dbhost>:<port>/<databasename>
    // or
    // ex) mongodb://<dbhost>:<port>/<databasename>
    public static String mongoConnectionString = "";
    public static String dbName = "";

    @Bean
    public MongoClient mongo() {
        ConnectionString connectionString = new ConnectionString(mongoConnectionString);
        MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
                .applyConnectionString(connectionString)
                .build();

        return MongoClients.create(mongoClientSettings);
    }

    @Bean
    public MongoTemplate mongoTemplate() throws Exception {
        return new MongoTemplate(mongo(), dbName);
    }
}
```  

#### application.yaml
`application.yaml` 파일에 설정해서 자동 설정을 사용하는 방법은 아래와 같다.  

```yaml
spring:
  data:
    mongodb:
      #uri: mongodb://<dbhost>:<port>/<databasename>
      # or
      uri: mongodb://<username>:<password>@<dbhost>:<port>/<databasename>
      database: <databasename> 
```  

#### MongoRepository
`MongoRepository` 를 사용하기 위해서는 아래와 같이 각 `Document` 클래스에 해당하는 `Repository` 클래스 생성이 필요하다.  

```java
public interface MyTestRepository extends MongoRepository<MyTest, String> {
    
}
```  


### MongoDB 예제 코드

#### Model
간단한 테스트를 위해 `MongoDB` 의 `Collection` 과 매핑되는 `User` 라는 모델 클래스를 아래와 같이 정의 한다.  

```java
@Document
@Getter
@Setter
@ToString
public class User {
    @Id
    private String id;
    private String name;
    private Integer age;

    private EmailAddress emailAddress;
    private Integer yearOfBirth;
}

@Document
@Getter
@Setter
@ToString
public class EmailAddress {
	@Id
	private String id;
	private String value;
}
```  

`@Document` `Annotation` 을 선언해 주면 해당 클래스는 `MongoDB` 의 `Collection` 와 같은 테이블 역할을 하게 된다.  



#### Repository
아래는 `User` 모델을 사용해서 `MongoDB` 의 `user` 이름을 갖는 `Collection` 에 연산을 수행할 수 있는 `Repository` 클래스이다.  

```java
public interface UserRepository extends MongoRepository<User, String> {
}
```  


### MongoTemplate 사용 하기
`MongoTemplate` 에서 제공하는 `API` 중 간단하게 `CRUD` 에 해당하는 부분만 살펴 본다.  

#### Insert
`insert` 는 `Collection` 에 존재하지 않는 새로운 데이터를 추가하는 것을 의미한다. 
만약 기존에 `insert` 된 데이터를 다시 수행하면 `DuplicateKeyException` 예외가 발생한다.  

```java
@Test
public void givenNewUser_whenInsert_thenCreatedUserId() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.mongoTemplate.insert(user, "user");

	// then
	assertThat(actual.getId(), not(emptyOrNullString()));
}


@Test
public void givenNewUser_whenInsert_thenExistsDb() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.mongoTemplate.insert(user, "user");

	// then
	User afterUser = this.mongoTemplate.findById(actual.getId(), User.class);
	assertThat(actual.getId(), is(afterUser.getId()));
}


@Test
public void givenExistUser_whenInsert_thenThrowsDuplicateKeyException() {
	// given
	User user = new User();
	user.setName("myName");
	this.mongoTemplate.insert(user, "user");

	// when
	user.setName("myName2");
	Executable executable = () -> this.mongoTemplate.insert(user, "user");

	// then
	assertThrows(DuplicateKeyException.class, executable);
}
```  

### save










---
## Reference
