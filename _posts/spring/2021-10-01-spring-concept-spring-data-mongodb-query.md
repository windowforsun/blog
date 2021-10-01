--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Data MongoDB Queries"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data MongoDB 에서 쿼리를 수행하는 다양한 방법과 그 차이에 대해서 알아보자'
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
    - QueryDSL
    - QueryMethods
    - Querying Documents
toc: true
use_math: true
---  

## Spring Data MongoDB Queries
[Spring Data MongoDB 소개]({{site.baseurl}}{% link _posts/spring/2021-09-27-spring-concept-spring-data-mongodb-intro.md %})
에서는 `Spring Data MongoDB` 를 구성하는 방법과 간단한 사용법에 대해서 알아 보았다. 
이번에는 `Spring Data MongoDB` 에는 다양한 방식을 사용해서 `MongoDB` 에 쿼리를 수행하는 방법에 대해 알아보고자 한다.  
대표적으로 `Query, Criteria` 를 사용하는 방법과 `Auto-generated Query Method(Repository)`, `QueryDSL` 이 있다.  

테스트를 위한 `Spring Data MongoDB` 설정은 `Spring Data MongoDB 소개` 를 참고한다.  


### Querying Documents
`Spring Data` 를 사용해서 `MongoDB` 에 쿼리를 수행하는 방법 중 아주 일반적인 방법으로 `Query` 와 `Criteria` 를 사용하는 방법이다. 
소개하는 방법 중 가장 `native operations` 와 가까운 방법이다.  

`Querying Documents` 의 테스트 메소드가 있는 `DocumentQueryTest` 클래스의 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class DocumentQueryTest extends MongoDbTest {

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
    
    // Test Methods ..
}
```  


#### Is
`is` 는 특정 필드의 값이 조건의 값과 동일한지 판별한다. 

```java
@Test
public void givenExistsUser_whenFindCriteriaIsMyNameByName_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").is("myName"));
	List<User> actual = mongoTemplate.find(query, User.class);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getId(), is(user_1.getId()));
	assertThat(actual.get(0).getName(), is("myName"));
}
```  

#### Regex
`regex` 는 정규식을 사용해서 조건에 판별식을 사용할 수 있다. 
아래는 특정 문자로 시작하거나 끝나는 정규식의 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindCriteriaRegexStartWithMyByName_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").regex("^my"));
	List<User> actual = this.mongoTemplate.find(query, User.class);

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName1"));
	assertThat(actual.get(1).getName(), is("myName2"));
}

@Test
public void givenExistsUser_whenFindCriteriaRegexEndWithNameByName_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("name").regex("Name$"));
	List<User> actual = this.mongoTemplate.find(query, User.class);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("yourName"));
}
```  

#### Lt and Gt
`lt` 는 `Less than`(이하), `gt` 는 `Greater than`(이상) 의 약자와 의미를 가지는 조건이다. 
아래는 나이 필드 `age` 에 `30 < age < 50` 조건을 사용한 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindGt30Lt50ByAge_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	Query query = new Query();
	query.addCriteria(Criteria.where("age").gt(30).lt(50));
	List<User> actual = this.mongoTemplate.find(query, User.class);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getAge(), is(40));
}
```  

#### Sort
`sort` 는 조건의 결과를 다시 특정 필드를 기준으로 정렬한 결과를 리턴해준다. 
아래는 `name` 필드 기준 내림차순으로 정렬한 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindSortDescByName_thenFoundSortedUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("myName2");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName3");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	Query query = new Query();
	query.with(Sort.by(Sort.Direction.DESC, "name"));
	List<User> actual = this.mongoTemplate.find(query, User.class);

	// then
	assertThat(actual, hasSize(3));
	assertThat(actual.get(0).getName(), is("myName3"));
	assertThat(actual.get(1).getName(), is("myName2"));
	assertThat(actual.get(2).getName(), is("myName1"));
}
```  

#### Pageable
`pageable` 은 `Pageable` 객체를 사용해서 쿼리 결과를 페이징해서 받아 볼 수 있다.  
아래는 총 3개의 데이터를 페이지 사이즈를 2로 설정해서 2번 조회한 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindByPageable_thenFoundPaginationUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("myName2");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName3");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	Pageable pageable = PageRequest.of(0, 2);
	Query firstPageQuery = new Query();
	firstPageQuery.with(pageable);
	List<User> actualFirstPage = this.mongoTemplate.find(firstPageQuery, User.class);
	pageable = PageRequest.of(1, 2);
	Query secondPageQuery = new Query();
	secondPageQuery.with(pageable);
	List<User> actualSecondPage = this.mongoTemplate.find(secondPageQuery, User.class);

	// then
	assertThat(actualFirstPage, hasSize(2));
	assertThat(actualFirstPage.get(0).getName(), is("myName1"));
	assertThat(actualFirstPage.get(1).getName(), is("myName2"));
	assertThat(actualSecondPage, hasSize(1));
	assertThat(actualSecondPage.get(0).getName(), is("myName3"));
}
```  

### QueryMethods(MongoRepository)
`QueryMethods` 는 `MongoRepository` 를 사용한 방법으로 `Querying Documents` 에 비해서 좀더 간편하게 쿼리 조건을 작성할 수 있다. 
쿼리를 작성하는 방법은 인터페이스에 정해진 규칙에 맞춰 메소드 이름을 정의하는 방법을 사용한다.  

`QueryMethods` 를 사용하기 위해서는 도큐먼트 모델을 정의해야 하고, `MongoRepository` 를 상속한 인터페이스 정의가 필요하다. 
아래는 테스트에서 사용하는 `MongoRepository` 를 상속한 `UserQueryMethodRepository` 인터페이스의 내용이다.  

```java
public interface UserQueryMethodsRepository extends MongoRepository<User, String> {
	List<User> findByName(String name);

	List<User> findByAge(int age);

	List<User> findByNameStartingWith(String regexp);

	List<User> findByNameEndingWith(String regexp);

	List<User> findByAgeBetween(int gt, int lt);

	List<User> findByNameLike(String name);

	List<User> findByOrderByAgeDesc();

	List<User> findByNameLikeOrderByAgeDesc(String name);
}
```  

`QueryMethods` 의 테스트 메소드가 있는 `QueryMethodsTest` 테스트 클래스 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class QueryMethodsTest extends MongoDbTest {

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private UserQueryMethodsRepository userQueryMethodsRepository;

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
        isStop = false;
    }
    
    // Test Methods ..
}
```  

#### FindBy*
`findBy*` 은 특정 필터링 조건을 바탕으로 쿼리를 수행할 때 사용한다. 
메소드에 사용할 필드 이름과 조건을 작성해 주면된다.  

메소드|설명
---|---
List<User> findByName(String name)|`name` 필드가 인자의 값과 일치한 결과
List<User> findByNameAndAge(String name, int age)|`name` 필드와 `age` 가 모두 인자의 값과 일치한 결과
List<User> findByNameOrAge(String name, int age)|`name` 필드와 `age` 중 인자의 값과 일치한 결과


```java
@Test
public void givenExistsUser_whenFindByName_thenFoundUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByName("myName");

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("myName"));
}

@Test
public void givenNotExistsUser_whenFindByAge_thenNotFoundUser() {
	// given
	User user = new User();
	user.setAge(100);
	user = this.mongoTemplate.insert(user, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByAge(99);

	// then
	assertThat(actual, hasSize(0));
}
```  

#### StartingWith, endingWith
`startingWith` 은 필드의 값이 특정 문자열로 시작하는지, `endingWith` 는 필드의 값이 특정 문자열로 끝나는지에 대한 조건이다.  

```java
@Test
public void givenExistsUser_whenFindByNameStartingWithMy_thenFoundStartingWithUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByNameStartingWith("my");

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName1"));
	assertThat(actual.get(1).getName(), is("myName2"));
}

@Test
public void givenExistsUser_whenFindByNameEndingWithName_thenFoundEndingWithUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByNameEndingWith("Name");

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("yourName"));
}
```  

#### Between
`between` 은 특정 `min`, `max` 값의 사이 값에 대한 조건이다. (`min < X < max`)
따로 설정햐진 않지만 비슷한 연산으로는 아래와 같은 것들이 있다.  

연산|설명
---|---
GreaterThan|초과
GreaterThanEqual|이상
LessThan|미만
LessThanEqual|이하

`between` 을 다르게 표현하면 아래와 같다.  

```java
List<User> findByAgeGreaterThanAndAgeLessThan(int gt, int lt);
```  

```java
@Test
public void givenExistsUser_whenFindByAgeBetween30_50_thenFoundBetweenUser() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByAgeBetween(30, 50);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getAge(), is(40));
}

@Test
public void givenExistsUser_whenFindByAgeBetween40_50_thenFoundEmpty() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByAgeBetween(40, 50);

	// then
	assertThat(actual, hasSize(0));
}
```  

#### Like and OrderBy
`like` 는 문자열 연산으로 특정 문자열 포함여부 등을 판별 할 수 있다. 
아래는 `name` 필드에 `yN` 이라는 문자 포함여부에 대한 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindByNameLike_ContainsYN_thenFoundLikeUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByNameLike("yN");

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName1"));
	assertThat(actual.get(1).getName(), is("myName2"));
}
```  

#### OrderBy 
`orderBy` 는 특정 필드를 기준으로 결과 값을 정렬 할 수 있다. 
아래는 `age` 필드를 기준으로 결과값을 내림차순 정렬하는 테스트이다.  

```java
@Test
public void givenExistsUser_whenFindByOrderByAgeDesc_thenFoundAgeDescUser() {
	// given
	User user_1 = new User();
	user_1.setAge(40);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(10);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(20);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByOrderByAgeDesc();

	// then
	assertThat(actual, hasSize(3));
	assertThat(actual.get(0).getAge(), is(40));
	assertThat(actual.get(1).getAge(), is(20));
	assertThat(actual.get(2).getAge(), is(10));
}
```  

그리고 `like` 와 함께 `orderBy` 를 사용해서 아래와 같이 조건으로 필터링된 쿼리의 결과를 정렬할 수 있다.  

```java
@Test
public void givenExistsUser_whenFindByNameLikeOrderByAgeDesc_thenFoundNameLikedAndAgeDescUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2.setAge(10);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3.setAge(40);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userQueryMethodsRepository.findByNameLikeOrderByAgeDesc("yN");

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName2"));
	assertThat(actual.get(0).getAge(), is(40));
	assertThat(actual.get(1).getName(), is("myName1"));
	assertThat(actual.get(1).getAge(), is(20));
}
```  


### Json QueryMethods
`MongoDB` 의 쿼리는 `CLI` 를 사용할 때 제공하는 메소드와 `Json` 으로 작성된 조건을 사용한다. 
`Spring Data MongoDB` 에서도 이와 같이 `Json` 조건을 `QueryMethod` 에 `@Query` `Annotation` 을 선언해서 `Native Query` 를 작성할 수 있도록 제공하고 있다.  

아래는 `MongoRepository` 의 구현체인 `UserJsonQueryMethodsRepository` 인터페이스에 `@Query` 를 사용해서 `Json` 조건을 작성한 내용이다.  

```java
public interface UserJsonQueryMethodsRepository extends MongoRepository<User, String> {
    @Query("{'name' : ?0}")
    List<User> findUsersByName(String name);

    @Query("{'name': {$regex : ?0}}")
    List<User> findUsersByRegexpName(String regexp);

    @Query("{'age' : {$gt : ?0, $lt : ?1}}")
    List<User> findUsersByAgeBetween(int gt, int lt);
}
```  

`QueryMethods` 에서는 메소드 이름이 곧 쿼리를 결정 짓는 요소였지만, 
`Json QueryMethods` 에서는 메소드 이름은 관련 없고, `@Query` 에 작성된 `Json` 조건만 쿼리를 결정하게 된다.  

`Json QueryMethods` 를 테스트하는 `JsonQueryMethodsTest` 클래스의 내용은 아래와 같다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class JsonQueryMethodsTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private UserJsonQueryMethodsRepository userJsonQueryMethodsRepository;

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
    
    // Test Methods ..
}
```  

#### FindBy
특정 필드의 값과 일치하는 도큐먼트를 찾는 쿼리로 아래는 `name` 필드와 일치하는 결과를 리턴 조건이다.  

```java
@Query("{'name' : ?0}")
List<User> findUsersByName(String name);
```  

```java
@Test
public void givenExistsUser_whenFindUsersByName_thenFoundUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user, "user");

	// when
	List<User> actual = this.userJsonQueryMethodsRepository.findUsersByName("myName");

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("myName"));
}
```  

#### Regular Expression
특정 필드의 대한 조건에 정규식(`regexp`)을 사용하는 방법으로 `MongoDB` 에서는 `$regex` 를 사용한다. 
아래는 `name` 필드에 정규식 조건을 적용한 예시이다.  

```java
@Query("{'name': {$regex : ?0}}")
List<User> findUsersByRegexpName(String regexp);
```  

```java
@Test
public void givenExistsUser_whenFindUsersByRegexpName_StartWithMy_thenFoundRegexpUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userJsonQueryMethodsRepository.findUsersByRegexpName("^my");

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName1"));
	assertThat(actual.get(1).getName(), is("myName2"));
}

@Test
public void givenExistsUser_whenFindUsersByRegexpName_EndWithName_thenFoundRegexpUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userJsonQueryMethodsRepository.findUsersByRegexpName("Name$");

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("yourName"));
}
```  

#### Gt, Lt
필드 값에 대해서 `$gt`(`GreaterThan`), `$lt`(`LessThan`) 을 사용해서 초과 미만 연산을 수행할 수 있다. 
아래는 `$gt`, `$lt` 를 사용해서 `between` 연산을 구현한 예시이다.  

```java
@Query("{'age' : {$gt : ?0, $lt : ?1}}")
List<User> findUsersByAgeBetween(int gt, int lt);
```  

```java
@Test
public void givenExistsUser_whenFindUsersByAgeBetween30_50_thenFoundBetweenUser() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userJsonQueryMethodsRepository.findUsersByAgeBetween(30, 50);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getAge(), is(40));
}

@Test
public void givenExistsUser_whenFindUsersByAgeBetween40_50_thenEmpty() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	List<User> actual = this.userJsonQueryMethodsRepository.findUsersByAgeBetween(40, 50);

	// then
	assertThat(actual, hasSize(0));
}
```  

### QueryDSL Queries
`QueryDSL` 을 사용하면 `MongoRepository` 와 매핑되는 모델을 사용해서 타입 세이프한 복잡한 쿼리를 작성할 수 있다. 
위에서 알아본 `Json QueryMethods` 의 경우 사용자가 자유롭게 쿼리는 작성할 수 있지만 조건 필드에 대한 타입이 모호하다는 단점이 있었다. 
만약 타입이 잘못 됐다면 컴파일 시점 이후에 이슈 파악이 가능할 것이다. 
하지만 `QueryDSL` 은 타입 세이프한 동적 쿼리 작성을 컴파일 전에도 알수 있어 복잡한 쿼리 작성에 도움이 된다.  

`Spring Data Mongo` 에서 `QueryDSL` 을 사용하기 위해서는 `build.gradle` 에 추가 설정이 필요하다.  

```groovy
plugins {
	// querydsl 추가
	id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}

dependencies {
	// querydsl 추가
	implementation 'com.querydsl:querydsl-mongodb:4.3.1'
	annotationProcessor(
		"com.querydsl:querydsl-apt:4.3.1",
	)
}


//querydsl 추가 시작
def querydslSrcDir = "$buildDir/generated/querydsl"

querydsl {
	springDataMongo = true
	querydslSourcesDir = querydslSrcDir
}

sourceSets {
	main {
		java {
			srcDirs = ['src/main/java', querydslSrcDir]
		}
	}
}

compileQuerydsl{
	options.annotationProcessorPath = configurations.querydsl
}

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
	querydsl.extendsFrom compileClasspath
}
```  

`build.gradle` 설정이 정상적으로 완료 됐는지 테스트 하기 위해서 아래 명령어를 사용해서 `QueryDSL` 에서 사용할 `QClass` 를 생성한다.  

```bash
$ ./gradlew compileQuerydsl

# or if multi module 

$ ./gradlew :intro-mongodb:compileQuerydsl
```  

명령이 정상으로 수행되고 `QClass` 가 생성됐는지는 아래 경로에서 `QUser`, `QEamilAddress` 클래스가 생성된 것으로 확인 가능하다.  

```bash
$ cd <ProjectRoot>/build/generated/querydsl/com/windowforsun/mongoexam/intromongo/model
$ tree .
.
├── QEmailAddress.java
└── QUser.java
```  

생성된 클래스 중 `QUser` 클래스 내용만 확인하면 아래와 같이 모델에 작성된 필드의 이름과 타입 등의 정보가 작성된 클래스이다.  

```java
@Generated("com.querydsl.codegen.EntitySerializer")
public class QUser extends EntityPathBase<User> {

    private static final long serialVersionUID = 1565382861L;

    private static final PathInits INITS = PathInits.DIRECT2;

    public static final QUser user = new QUser("user");

    public final NumberPath<Integer> age = createNumber("age", Integer.class);

    public final QEmailAddress emailAddress;

    public final StringPath id = createString("id");

    public final StringPath name = createString("name");

    public final NumberPath<Integer> yearOfBirth = createNumber("yearOfBirth", Integer.class);

    public QUser(String variable) {
        this(User.class, forVariable(variable), INITS);
    }

    public QUser(Path<? extends User> path) {
        this(path.getType(), path.getMetadata(), PathInits.getFor(path.getMetadata(), INITS));
    }

    public QUser(PathMetadata metadata) {
        this(metadata, PathInits.getFor(metadata, INITS));
    }

    public QUser(PathMetadata metadata, PathInits inits) {
        this(User.class, metadata, inits);
    }

    public QUser(Class<? extends User> type, PathMetadata metadata, PathInits inits) {
        super(type, metadata, inits);
        this.emailAddress = inits.isInitialized("emailAddress") ? new QEmailAddress(forProperty("emailAddress")) : null;
    }

}
```  

그리고 `QueryDSL` 을 사용하기 위해 `MongoRepository` 와 추가로 `QuerydslPredicateExecutor` 인터페이스를 상속한 `UserQueryDslRepository` 는 아래와 같다.  

```java
public interface UserQueryDslRepository extends MongoRepository<User, String>, QuerydslPredicateExecutor<User> {

}
```  

아래는 `QueryDSL` 의 테스트 메소드가 있는 `QueryDslRepositoryTest` 클래스의 내용이다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class QueryDslRepositoryTest extends MongoDbTest {

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private UserQueryDslRepository userQueryDslRepository;

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
    
    // Test Mehtods ..
}
```  

#### Eq
`eq` 는 특정 필드의 값과 동일한지 판별하는 조건이다. 
아래는 `name` 필드가 `myName` 인 조건을 테스트하는 코드이다.  

```java
@Test
public void givenExistsUser_whenFindAllByEqName_thenFoundUser() {
	// given
	User user = new User();
	user.setName("myName");
	user = this.mongoTemplate.insert(user, "user");

	// when
	QUser qUser = new QUser("user");
	Predicate predicate = (Predicate) qUser.name.eq("myName");
	List<User> actual = (List<User>) this.userQueryDslRepository.findAll(predicate);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getId(), is(user.getId()));
	assertThat(actual.get(0).getName(), is("myName"));
}
```  

#### StartingWith and EndingWith
`startingWith` 는 특정 문자열로 시작, `endingWith` 은 특정 문자열로 끝나는 조건이다. 
아래는 `name` 필드가 `my` 로 시작하는 것과 `Name` 으로 끝나는 조건에 대한 테스트 이다.  

```java
@Test
public void givenExistUser_whenFindAllStartWithName_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2.setAge(10);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3.setAge(40);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	QUser qUser = new QUser("user");
	Predicate predicate = qUser.name.startsWith("my");
	List<User> actual = (List<User>) this.userQueryDslRepository.findAll(predicate);

	// then
	assertThat(actual, hasSize(2));
	assertThat(actual.get(0).getName(), is("myName1"));
	assertThat(actual.get(1).getName(), is("myName2"));
}

@Test
public void givenExistUser_whenFindAllEndWithName_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setName("myName1");
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setName("yourName");
	user_2.setAge(10);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setName("myName2");
	user_3.setAge(40);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	QUser qUser = new QUser("user");
	Predicate predicate = qUser.name.endsWith("Name");
	List<User> actual = (List<User>) this.userQueryDslRepository.findAll(predicate);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getName(), is("yourName"));
}
```  

#### Between
`between` 은 `Gte`(`GreaterThanEqual`), `Lte`(`LessThanEqual`) 을 모두 포함하는 조건으로 `min <= X <= max` 를 의미한다. 
아래는 `age` 의 값이 20, 40, 80으로 있을 때 `30 <= age <= 50`, `40 <= age <=50` 에 대한 테스트 이다.  

```java
@Test
public void givenExistsUser_whenFindAllBetweenAge_30_50_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	QUser qUser = new QUser("user");
	Predicate predicate = qUser.age.between(30, 50);
	List<User> actual = (List<User>) this.userQueryDslRepository.findAll(predicate);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getAge(), is(40));
}

@Test
public void givenExistsUser_whenFindAllBetweenAge_40_50_thenFoundUser() {
	// given
	User user_1 = new User();
	user_1.setAge(20);
	user_1 = this.mongoTemplate.insert(user_1, "user");
	User user_2 = new User();
	user_2.setAge(40);
	user_2 = this.mongoTemplate.insert(user_2, "user");
	User user_3 = new User();
	user_3.setAge(80);
	user_3 = this.mongoTemplate.insert(user_3, "user");

	// when
	QUser qUser = new QUser("user");
	Predicate predicate = qUser.age.between(40, 50);
	List<User> actual = (List<User>) this.userQueryDslRepository.findAll(predicate);

	// then
	assertThat(actual, hasSize(1));
	assertThat(actual.get(0).getAge(), is(40));
}
```




---
## Reference
[A Guide to Queries in Spring Data MongoDB](https://www.baeldung.com/queries-in-spring-data-mongodb)  