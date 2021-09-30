--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Data MongoDB 소개 및 사용 방법"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data MongoDB 에 대해 알아보고 설정과 간단하게 사용하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Data MongoDB
    - MongoDB
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
        // yaml 에 설정이 필요한 경우 
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


### MongoDB Model
간단한 테스트를 위해 `MongoDB` 의 `user` 라는 `Collection` 의 `Document` 모델 클래스를 아래와 같이 정의 한다.  

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

`@Document` `Annotation` 을 선언해 주면 해당 클래스는 `MongoDB` 의 `Collection` 에 저장되는 `Document` 의 구조를 정의할 수 있다. 
그리고 도큐먼트의 필드 중 아이디에 해당하는 필드에 `@Id` `Annotation` 을 선언해 주면 아이디와 매핑 되게 된다.  




### MongoTemplate 사용 하기
`MongoTemplate` 에서 제공하는 `API` 중 간단하게 `CRUD` 에 해당하는 부분만 살펴 본다. 
앞서 언급한 것과 같이 `MongoTemplate` 은 별도의 `POJO` 선언 없이도 사용이 가능하다. (ex) HashMap)
예제에서는 앞서 선언한 `User` 모델을 사용해서 진행하도록 한다.  

`MongoTemplate` 의 테스트를 수행하는 `MongoTemplateTest` 클래스에서 `Test Method` 를 제외한 나머지 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class MongoTemplateTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;

    @BeforeAll
    public static void init() {
        MongoTemplateConfig.dbName = "test";
        MongoTemplateConfig.mongoConnectionString = new StringBuilder()
                .append("mongodb://")
                .append(MongoDbTest.MONGO_DB.getHost())
                .append(":")
                .append(MongoDbTest.MONGO_DB.getFirstMappedPort())
                .append("/")
                .append(MongoTemplateConfig.dbName)
                .toString();
    }

    @BeforeEach
    public void setUp() {
        this.mongoTemplate.dropCollection(User.class);
    }

    // Test Method ..
}
```  

#### Insert
`insert` 는 `Collection` 에 존재하지 않는 새로운 데이터를 추가하는 것을 의미한다. 
만약 기존에 `insert` 된(`id` 존재) 데이터를 다시 수행하면 `DuplicateKeyException` 예외가 발생한다.  

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

#### Save
`save` 는 `insert-or-update` 라고 할 수 있다. 
만약 `save` 하려는 `id` 가 존재하면 `update` 를 수행하고, 존재하지 않는다면 `insert` 를 수행한다.  

```java
@Test
public void givenNewUser_whenSave_thenCreatedUserId() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.mongoTemplate.save(user, "user");

	// then
	assertThat(actual.getId(), not(emptyOrNullString()));
}

@Test
public void givenExistsUser_whenSave_thenUpdateName() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.save(user, "user");

	// when
	user.setName("myName2");
	User actual = this.mongoTemplate.save(user, "user");

	// then
	assertThat(actual.getId(), is(user.getId()));
	assertThat(actual.getName(), is("myName2"));
}
```  

####  UpdateFirst
`updateFirst` 는 조건이 해당하는 도큐먼트 중 가장 첫번째에 있는 도튜먼트를 업데이트 한다.  

```java
@Test
public void givenTwoUser_whenUpdateFirst_thenFirstUserNameUpdated() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("myName");
	user_2 = this.mongoTemplate.insert(user_2);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	UpdateResult actual = this.mongoTemplate.updateFirst(query, update, User.class);

	// then
	User afterUser_1 = this.mongoTemplate.findById(user_1.getId(), User.class);
	User afterUser_2 = this.mongoTemplate.findById(user_2.getId(), User.class);
	assertThat(actual.getModifiedCount(), is(1L));
	assertThat(afterUser_1.getName(), is("myName1"));
	assertThat(afterUser_2.getName(), is("myName"));
}
```  

#### UpdateMulti
`updateMulti` 는 조건이 해당하는 모든 도큐먼트를 업데이트 한다.  

```java
@Test
public void givenTwoUser_whenUpdateMulti_thenTwoUserNameUpdated() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("myName");
	user_2 = this.mongoTemplate.insert(user_2);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	UpdateResult actual = this.mongoTemplate.updateMulti(query, update, User.class);

	// then
	User afterUser_1 = this.mongoTemplate.findById(user_1.getId(), User.class);
	User afterUser_2 = this.mongoTemplate.findById(user_2.getId(), User.class);
	assertThat(actual.getModifiedCount(), is(2L));
	assertThat(afterUser_1.getName(), is("myName1"));
	assertThat(afterUser_2.getName(), is("myName1"));
}
```  

#### FindAndModify
`findAndModify` 는 실제 동작은 `updateFirst` 와 동일하게 조건에 해당하는 도큐먼트 중 가장 첫번째 도큐먼트를 업데이트 한다. 
그리고 업데이트 전의 도큐먼트 내용을 리턴한다.  

```java
@Test
public void givenSingleUser_whenFindAndModify_thenModified() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	User actual = this.mongoTemplate.findAndModify(query, update, User.class);

	// then
	assertThat(actual.getId(), is(user.getId()));
	assertThat(actual.getName(), is("myName"));
	User afterUser_1 = this.mongoTemplate.findById(user.getId(), User.class);
	assertThat(afterUser_1.getName(), is("myName1"));

}

@Test
public void givenTwoUser_whenFindAndModify_thenModified() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("myName");
	user_2 = this.mongoTemplate.insert(user_2);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	User actual = this.mongoTemplate.findAndModify(query, update, User.class);

	// then
	assertThat(actual.getId(), is(user_1.getId()));
	assertThat(actual.getName(), is("myName"));
	User afterUser_1 = this.mongoTemplate.findById(user_1.getId(), User.class);
	assertThat(afterUser_1.getName(), is("myName1"));
	User afterUser_2 = this.mongoTemplate.findById(user_2.getId(), User.class);
	assertThat(afterUser_2.getName(), is("myName"));
}
```  

#### Upsert
`upsert` 는 조건에 해당하는 도큐먼트가 존재하는 경우 `Update`를 수행하고, 
존재하지 않는다면 `Insert`를 수행한다. 
`save` 와 비슷한 동작이지만, `save` 는 `id` 필드의 존재 여부로 수행되고 `upsert` 는 다양한 조건을 사용할 수 있다.  

```java
@Test
public void whenUpsert_thenCreateUser() {
	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	UpdateResult actual = this.mongoTemplate.upsert(query, update, User.class);

	// then
	assertThat(actual.getModifiedCount(), is(0L));
	String upsertedId = actual.getUpsertedId().asObjectId().getValue().toString();
	assertThat(upsertedId, not(emptyOrNullString()));
	User user = this.mongoTemplate.findById(upsertedId, User.class);
	assertThat(user.getName(), is("myName1"));
}

@Test
public void givenExistsUser_whenUpsert_thenUpdatedUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	Update update = new Update();
	update.set("name", "myName1");
	UpdateResult actual = this.mongoTemplate.upsert(query, update, User.class);

	// then
	assertThat(actual.getModifiedCount(), is(1L));
	User afterUser = this.mongoTemplate.findById(user.getId(), User.class);
	assertThat(afterUser.getName(), is("myName1"));
}
```  


#### Remove
`remove` 는 특정 도큐먼트를 삭제할 수도 있고, 
조건을 사용해서 여러 도큐먼트를 한번에 삭제할 수도 있다.  

```java
@Test
public void givenExistsUser_whenRemove_thenRemovedUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	DeleteResult actual = this.mongoTemplate.remove(user);

	// then
	assertThat(actual.getDeletedCount(), is(1L));
	User afterUser = this.mongoTemplate.findById(user.getId(), User.class);
	assertThat(afterUser, nullValue());
}

@Test
public void givenNotExistsUser_whenRemove_thenDeleteCountIsZero() {
	// given
	User user = new User();
	user.setId("test");
	user.setName("myName");

	 // when
	DeleteResult actual = this.mongoTemplate.remove(user);

	// then
	assertThat(actual.getDeletedCount(), is(0L));
}

@Test
public void givenExistsUser_whenRemoveQuery_thenRemovedUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2);
	User user_3 = new User();
	user_3.setName("myName");
	user_3 = this.mongoTemplate.insert(user_3);

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	DeleteResult actual = this.mongoTemplate.remove(query, User.class);

	// then
	assertThat(actual.getDeletedCount(), is(2L));
	User afterUser_1 = this.mongoTemplate.findById(user_1.getId(), User.class);
	assertThat(afterUser_1, nullValue());
	User afterUser_2 = this.mongoTemplate.findById(user_2.getId(), User.class);
	assertThat(afterUser_2, not(nullValue()));
	User afterUser_3 = this.mongoTemplate.findById(user_3.getId(), User.class);
	assertThat(afterUser_3, nullValue());
}
```  


### MongoRepository 사용 하기
`MongoRepository` 를 사용하면 `MongoTemplate` 보다 좀더 간편하게 쿼리 수행이 가능하다. 
하지만 별도로 `POJO` 를 선언해 줘야 하고, `POJO` 와 매핑되는 `Repository` 또한 선언이 필요하다. 
테스트에서는 아래 `UserRepository` 를 사용한다. 
그리고 테스트는 `MongoRepository` 에서 기본으로 제공하는 몇가지 `API` 에 대해서만 알아본다.  

```java
public interface UserRepository extends MongoRepository<User, String> {
}
```  

> `MongoRepository` 사용을 위해서는 `@EnableMongoRepositories` 선언이 필요하다.  

`MongoRepository` 테스트를 수행하는 `MongoRepositoryTest` 클래스에서 `Test Method` 를 제외한 나머지 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class MongoRepositoryTest extends MongoDbTest {

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private UserRepository userRepository;

    @BeforeAll
    public static void init() {
        MongoTemplateConfig.dbName = "test";
        MongoTemplateConfig.mongoConnectionString = new StringBuilder()
                .append("mongodb://")
                .append(MongoDbTest.MONGO_DB.getHost())
                .append(":")
                .append(MongoDbTest.MONGO_DB.getFirstMappedPort())
                .append("/")
                .append(MongoTemplateConfig.dbName)
                .toString();
    }

    @BeforeEach
    public void setUp() {
        this.mongoTemplate.dropCollection(User.class);
    }
    
    // Test Method ..
}
```  

#### Insert
`insert` 는 `Collection` 에 존재하지 않는 새로운 데이터를 추가하는 것을 의미한다.
만약 기존에 `insert` 된(`id` 존재) 데이터를 다시 수행하면 `DuplicateKeyException` 예외가 발생한다.

```java
@Test
public void givenNewUser_whenInsert_thenCreatedUserId() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.userRepository.insert(user);

	// then
	assertThat(actual.getId(), not(emptyOrNullString()));
}

@Test
public void givenNewUser_whenInsert_thenExistsDb() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.userRepository.insert(user);

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
	Executable executable = () -> this.userRepository.insert(user);

	// then
	assertThrows(DuplicateKeyException.class, executable);
}
```  


#### Save
`save` 는 `insert-or-update` 라고 할 수 있다.
만약 `save` 하려는 `id` 가 존재하면 `update` 를 수행하고, 존재하지 않는다면 `insert` 를 수행한다.  

```java
@Test
public void givenNewUser_whenSave_thenCreatedUserId() {
	// given
	User user = new User();
	user.setName("myName");

	// when
	User actual = this.userRepository.save(user);

	// then
	assertThat(actual.getId(), not(emptyOrNullString()));
}

@Test
public void givenExistsUser_whenSave_thenUpdateName() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.save(user, "user");

	// when
	user.setName("myName2");
	User actual = this.userRepository.save(user);

	// then
	assertThat(actual.getId(), is(user.getId()));
	assertThat(actual.getName(), is("myName2"));
}
```  

#### Delete
`delete` 와 `deleteById` 는 특정 도큐먼트 혹은 도큐먼트에 해당하는 아이디를 사용해서 삭제를 수행할 수 있다.  

```java
@Test
public void givenExistsUser_whenDelete_thenRemovedUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	this.userRepository.delete(user);
	User actual = this.mongoTemplate.findById(user.getId(), User.class);

	// then
	assertThat(actual, nullValue());
}

@Test
public void givenExistsUser_whenDeleteById_thenRemovedUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	this.userRepository.deleteById(user.getId());
	User actual = this.mongoTemplate.findById(user.getId(), User.class);

	// then
	assertThat(actual, nullValue());
}
```  

#### FindOne
`findOne` 은 도큐먼트 객체를 사용해서 일치하는 가장 첫 번째 도큐먼트를 찾을 수 있다. 
도큐먼트 객체에 설정된 필드 값과 동일한 도큐먼트를 찾아 리턴한다.  

```java
@Test
public void givenMultipleUser_whenFindOneByName_thenReturnUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("myName");
	user_2 = this.mongoTemplate.insert(user_2);
	User example = new User();
	example.setName("myName");

	// when
	User actual = this.userRepository.findOne(Example.of(example)).orElse(null);

	// then
	assertThat(actual, notNullValue());
	assertThat(actual.getId(), is(user_1.getId()));
}

@Test
public void givenMultipleUser_whenFindOneByNameAge_thenReturnUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName");
	user_1.setAge(1);
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("myName");
	user_2.setAge(2);
	user_2 = this.mongoTemplate.insert(user_2);
	User example = new User();
	example.setName("myName");
	example.setAge(2);

	// when
	User actual = this.userRepository.findOne(Example.of(example)).orElse(null);

	// then
	assertThat(actual, notNullValue());
	assertThat(actual.getId(), is(user_2.getId()));
}
```  

#### FindById
`findById` 는 도큐먼트 아이디와 일치하는 도큐먼트를 찾아 리턴한다.  

```java
@Test
public void givenExistsUser_whenFindById_thenReturnUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	User actual = this.userRepository.findById(user.getId()).orElse(null);

	// then
	assertThat(actual, notNullValue());
	assertThat(actual.getId(), is(user.getId()));
	assertThat(actual.getName(), is(user.getName()));
}
```  

#### Exists
`exists` 는 `findOne` 과 동일하게 도큐먼트 객체를 사용해서 필드의 값과 일치하는 도큐먼트가 존재하는지 리턴한다.  

```java
@Test
public void givenExistsUser_whenExists_thenReturnTrue() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);
	User example = new User();
	user.setName("myName");

	// when
	boolean actual = this.userRepository.exists(Example.of(example));

	// then
	assertThat(actual, is(true));
}
```  

#### ExistsById
`existsById` 는 특정 도큐먼트 아이디가 존재하는지 리턴한다.  

```java
@Test
public void givenExistsUser_whenExistsById_thenReturnTrue() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user);

	// when
	boolean actual = this.userRepository.existsById(user.getId());

	// then
	assertThat(actual, is(true));
}
```  

#### FindAll With Sort
`findAll` 은 컬렉션에 존재하는 모든 도큐먼트를 가져올 수 있는데, 
`Sort` 를 사용하면 특정 조건을 기준으로 정렬된 리스트로 결과값을 리턴 받을 수 있다.  

```java
@Test
public void givenThreeUser_whenFindAllWithSort_thenSortedDescByName() {
	// given
	User user_1 = new User();
	user_1.setName("1");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("2");
	user_2 = this.mongoTemplate.insert(user_2);
	User user_3 = new User();
	user_3.setName("3");
	user_3 = this.mongoTemplate.insert(user_3);

	// when
	List<User> actual = this.userRepository.findAll(Sort.by(Sort.Direction.DESC, "name"));

	// then
	assertThat(actual, hasSize(3));
	assertThat(actual.get(0).getName(), is("3"));
	assertThat(actual.get(1).getName(), is("2"));
	assertThat(actual.get(2).getName(), is("1"));
}
```  

#### FindAll With Pageable
`findAll` 은 컬렉션 전체 데이터에 대해서 `Pageable` 을 사용해서 페이징 처리 해서 가져올 수 있다.  

```java
@Test
public void givenThreeUser_whenFindAllWithPageable_thenReturnPaginationList() {
	// given
	User user_1 = new User();
	user_1.setName("1");
	user_1 = this.mongoTemplate.insert(user_1);
	User user_2 = new User();
	user_2.setName("2");
	user_2 = this.mongoTemplate.insert(user_2);
	User user_3 = new User();
	user_3.setName("3");
	user_3 = this.mongoTemplate.insert(user_3);
	Pageable firstPageRequest = PageRequest.of(0, 2);
	Pageable secondPageRequest = PageRequest.of(1, 2);

	// when
	List<User> actualFirstPage = this.userRepository.findAll(firstPageRequest).getContent();
	List<User> actualSecondPage = this.userRepository.findAll(secondPageRequest).getContent();

	// then
	assertThat(actualFirstPage, hasSize(2));
	assertThat(actualFirstPage.get(0).getName(), is("1"));
	assertThat(actualFirstPage.get(1).getName(), is("2"));
	assertThat(actualSecondPage, hasSize(1));
	assertThat(actualSecondPage.get(0).getName(), is("3"));
}
```  




---
## Reference
[Introduction to Spring Data MongoDB](https://www.baeldung.com/spring-data-mongodb-tutorial)  