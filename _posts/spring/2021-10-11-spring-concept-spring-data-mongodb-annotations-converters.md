--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Data MongoDB Common Annotations, Converters"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data MongoDB 에서 유용한 Annotation 과 데이터를 변환할 수 있는 Converter 에 대해서 알아보자'
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
    - Converters
    - Transient
    - Field
    - PersistenceConstruct
    - Value
    - MappingMongoConverter
    - MongoCustomConversions
toc: true
use_math: true
---  

> [Spring Data MongoDB 소개]({{site.baseurl}}{% link _posts/spring/2021-09-27-spring-concept-spring-data-mongodb-intro.md %})
을 기반으로 테스트 예제 코드를 진행한다. 

## Spring Data MongoDB Common Annotations
이번에는 몇가지 공통적으로 사용될수 있는 `Annotation` 에 대해 알아본다.  

`Annotation` 에 대해서 테스트 메소드가 있는 테스트 클래스의 내용은 아래와 같다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@ExtendWith(SpringExtension.class)
@AutoConfigureJsonTesters
public class CommonAnnotationsTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    // Test Methods ..
}
```  

### @Transient
`@Transient` 는 `Entity Class` 로 정의된 클래스 필드 중 실제 `MongoDB Document` 에 제외 시키고 싶은 필드에 선언해서 사용한다. 
아래와 같이 제외하고 싶은 변수에 필드 레벨로 선언해주면 된다. 
`yearOfBirth` 필드는 실제로 저장되지 않는 필드이다.  

```java
@Document
@Data
public static class TransientExamUser {
	@Id
	private String id;
	private String name;
	private int age;
	@Transient
	private Integer yearOfBirth;
}
```  

아래는 실제 테스트를 수행한 코드이다.  

```java
@Test
public void givenTransientFieldYearOrBirth_whenInert_thenNotExistsYearOfBirthFieldInMongoDB() throws Exception {
    // given
    TransientExamUser user = new TransientExamUser();
    user.setName("myName");
    user.setAge(20);
    user.setYearOfBirth(2021);
    user = this.mongoTemplate.insert(user);

    // when
    Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
            "mongo",
            "--quiet",
            "--eval",
            "db.getSiblingDB('test'); db.transientExamUser.find({'_id' : ObjectId('" + user.getId() + "')});"
    );

    // then
    System.out.println(actualExec.getStdout());
    JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$._id")
            .isEqualTo("ObjectId(\"" + user.getId() + "\")");
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$.name")
            .isEqualTo(user.getName());
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$.age")
            .isEqualTo(user.getAge());
    Assertions.assertThat(actual)
            .doesNotHaveJsonPathValue("$.yearOfBirth");
}

@Test
public void givenTransientFieldYearOrBirth_whenInertAndFind_thenYearOfBirthFieldIsNull() {
    // given
    TransientExamUser user = new TransientExamUser();
    user.setName("myName");
    user.setAge(20);
    user.setYearOfBirth(2021);
    user = this.mongoTemplate.insert(user);

    // when
    TransientExamUser actual = this.mongoTemplate.findById(user.getId(), TransientExamUser.class);

    // then
    assertThat(actual, notNullValue());
    assertThat(actual.getId(), is(user.getId()));
    assertThat(actual.getAge(), is(user.getAge()));
    assertThat(actual.getYearOfBirth(), nullValue());
}
```  

<details><summary>db.transientExamUser..find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{ 
	"_id" : ObjectId("61631a409891e5713bfffe03"), 
	"name" : "myName", 
	"age" : 20, 
	"_class" : "com.windowforsun.mongoexam.intromongo.commonannotations.CommonAnnotationsTest$TransientExamUser"
}
```  

</div>
</details>

### @Field
`@Field` 는 실제로 저장될 `Document` 에 커스텀한 필드 이름을 지정할 수 있다.  

```java
@Test
public void givenCustomFieldName_whenInsert_thenCustomFieldNameInMongoDB() throws Exception {
    // given
    FieldExamUser user = new FieldExamUser();
    user.setName("myName");
    user.setAge(20);
    user.setYearOfBirth(2021);
    user = this.mongoTemplate.insert(user);

    // when
    Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
            "mongo",
            "--quiet",
            "--eval",
            "db.getSiblingDB('test'); db.fieldExamUser.find({'_id' : ObjectId('" + user.getId() + "')});"
    );

    // then
    System.out.println(actualExec.getStdout());
    JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$._id")
            .isEqualTo("ObjectId(\"" + user.getId() + "\")");
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$.nameField")
            .isEqualTo(user.getName());
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$.ageField")
            .isEqualTo(user.getAge());
    Assertions.assertThat(actual)
            .extractingJsonPathValue("$.yearOfBirth")
            .isEqualTo(user.getYearOfBirth());
}

@Test
public void givenCustomFieldName_whenInsert_thenCustomFieldNameMappingToEntityClassField() throws Exception {
    // given
    FieldExamUser user = new FieldExamUser();
    user.setName("myName");
    user.setAge(20);
    user.setYearOfBirth(2021);
    user = this.mongoTemplate.insert(user);

    // when
    FieldExamUser actual = this.mongoTemplate.findById(user.getId(), FieldExamUser.class);

    // then
    assertThat(actual, notNullValue());
    assertThat(actual.getId(), is(user.getId()));
    assertThat(actual.getAge(), is(user.getAge()));
    assertThat(actual.getYearOfBirth(), is(user.getYearOfBirth()));
}
```  

<details><summary>db.fieldExamUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{ 
	"_id" : ObjectId("61631aba94218d0d3d40b1d0"), 
	"nameField" : "myName", 
	"ageField" : 20, 
	"yearOfBirth" : 2021, 
	"_class" : "com.windowforsun.mongoexam.intromongo.commonannotations.CommonAnnotationsTest$FieldExamUser"
}
```  

</div>
</details>

### @PersistenceConstruct, @Value
`@PersistenceConstruct` 는 `Spring Data MongoDB` 에서 `Entity Class` 의 인스턴스를 생성할때 기본으로 사용할 생성장 메소드를 정의한다. 
그 예시는 아래와 같다. 
`PersistenceConstructorExamUser` 의 인스턴스를 생성할때 3개의 인자를 갖는 생성자를 사용하게 된다.  

```java
@Document
@Data
public static class PersistenceConstructorExamUser {
	@Id
	private String id;
	private String name;
	private int age;
	private Integer yearOfBirth;

	public PersistenceConstructorWithValueExamUser() {

	}

	@PersistenceConstructor
	public PersistenceConstructorExamUser(String name, int age, Integer yearOfBirth) {
		this.name = name;
		this.age = age;
		this.yearOfBirth = yearOfBirth;
	}
}
```  

그리고 `@Value` 는 `SpEL(Spring Expression Language)` 를 사용해서 객체를 구성하기 전에 `MongoDB` 에서 읽은 도큐먼트를 변환하거나, 
기본 값을 설정할 수 있다. 
`MongoDB` 에 저장된 도큐먼트 필드값이 `null` 이거나 불필요한 값일 때, 기본값을 설정하기 위해서는 `@PersistenceConstructor` 와 `@Value` 를 함께 사용해서 구현할 수 있다. 

```java
@Document
@Data
public static class PersistenceConstructorWithValueExamUser {
	@Id
	private String id;
	private String name;
	private int age;
	private Integer yearOfBirth;

	public PersistenceConstructorWithValueExamUser() {

	}

	@PersistenceConstructor
	public PersistenceConstructorWithValueExamUser(@Value("#root.name ?: 'defaultName'") String name, int age, @Value("#root.yearOfBirth ?: 999") Integer yearOfBirth) {
		this.name = name;
		this.age = age;
		this.yearOfBirth = yearOfBirth;
	}
}
```  

위 엔티티 클래스는 `name` 필드가 `null` 값이라면 `defaultName`, `yeaerOfBirth` 필드가 `null` 이라면 `999` 라는 값이 기본으로 설정돼서 최종적으로 클래스 객체가 생성된다.  

```java
@Document
@Data
public static class PersistenceConstructorWithValueExamUser {
    @Id
    private String id;
    private String name;
    private int age;
    private Integer yearOfBirth;

    public PersistenceConstructorWithValueExamUser() {

    }

    @PersistenceConstructor
    public PersistenceConstructorWithValueExamUser(@Value("#root.name ?: 'defaultName'") String name, int age, @Value("#root.yearOfBirth ?: 999") Integer yearOfBirth) {
        this.name = name;
        this.age = age;
        this.yearOfBirth = yearOfBirth;
    }
}

@Test
public void givenNullNameAndYearOfBirth_whenInsert_thenUseEntityInstanceExistsDefaultValueAndIsNullInMongoDB() throws Exception {
    // given
    PersistenceConstructorWithValueExamUser user = new PersistenceConstructorWithValueExamUser();
    user.setAge(20);
    user = this.mongoTemplate.insert(user);

    // when
    PersistenceConstructorWithValueExamUser actual = this.mongoTemplate.findById(user.getId(), PersistenceConstructorWithValueExamUser.class);
    Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
            "mongo",
            "--quiet",
            "--eval",
            "db.getSiblingDB('test'); db.persistenceConstructorWithValueExamUser.find({'_id' : ObjectId('" + user.getId() + "')});"
    );

    // then
    System.out.println(actualExec.getStdout());
    assertThat(actual.getId(), is(user.getId()));
    assertThat(actual.getName(), is("defaultName"));
    assertThat(actual.getAge(), is(user.getAge()));
    assertThat(actual.getYearOfBirth(), is(999));

    JsonContent<Object> actualMongo = this.jsonTester.from(actualExec.getStdout());
    Assertions.assertThat(actualMongo)
            .extractingJsonPathValue("$.age")
            .isEqualTo(20);
    Assertions.assertThat(actualMongo)
            .doesNotHaveJsonPathValue("$.name")
            .doesNotHaveJsonPathValue("$.yearOfBirth");
}
```  

<details><summary>db.persistenceConstructorWithValueExamUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{ 
	"_id" : ObjectId("61631eb472666720001a1ea5"), 
	"age" : 20, 
	"_class" : "com.windowforsun.mongoexam.intromongo.commonannotations.CommonAnnotationsTest$PersistenceConstructorWithValueExamUser"
}
```  

</div>
</details>


## Spring Data MongoDB Converters
다음으로는 `MongoDB` 에 도큐먼트를 저장하고, `MongoDB` 로 부터 도큐먼트를 읽어 올때 `Converter` 를 사용해서 필드 값을 추가, 제거, 수정 등이 가능하다. 
`Converter` 는 `Spring Data MongoDB` 에서 기본으로 제공하는 `MappingMongoConverter` 를 사용하거나, 
사용자가 직접 `Converter` 의 구현체로 정의한 `Custom Converter` 를 사용할 수 있다.  


### MappingMongoConverter
`MappingMongoConverter` 를 사용해서 `Spring Data MongoDB` 가 `MongoDB` 에 실제로 데이터를 저장할 때 자동으로 생성되는 `_class` 필드를 제거해 본다. 
`MappingMongoConverter` 를 생성하고 `DefaultMongoTypeMapper` 를 사용해서 `_class` 필드를 제거는 아래와 같은 빈을 선언해 주면된다.  

```java
@Bean
public MappingMongoConverter mappingMongoConverter() {
	DbRefResolver dbRefResolver = new DefaultDbRefResolver(this.mongoDatabaseFactory);
	MappingMongoConverter converter = new MappingMongoConverter(dbRefResolver, this.mongoMappingContext);
	converter.setTypeMapper(new DefaultMongoTypeMapper(null));

	return converter;
}
```  

실제로 잘 동작하는지 테스트하면 아래와 같다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class MappingMongoConverterTest extends MongoDbTest {
    @Document
    @Data
    public static class NoClassFieldUser {
        @Id
        private String id;
        private String name;
    }

    @TestConfiguration
    @RequiredArgsConstructor
    public static class RemovedClassFieldConverterConfig {
        private final MongoDatabaseFactory mongoDatabaseFactory;
        private final MongoMappingContext mongoMappingContext;

        @Bean
        public MappingMongoConverter mappingMongoConverter() {
            DbRefResolver dbRefResolver = new DefaultDbRefResolver(this.mongoDatabaseFactory);
            MappingMongoConverter converter = new MappingMongoConverter(dbRefResolver, this.mongoMappingContext);
            converter.setTypeMapper(new DefaultMongoTypeMapper(null));

            return converter;
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Test
    public void givenRemoveClassField_whenInsert_thenNotExistsClassFieldInMongoDB() throws Exception {
        // given
        NoClassFieldUser user = new NoClassFieldUser();
        user.setName("myName");
        user = this.mongoTemplate.insert(user);

        // when
        NoClassFieldUser actual = this.mongoTemplate.findById(user.getId(), NoClassFieldUser.class);
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.noClassFieldUser.find({'_id' : ObjectId('" + user.getId() + "')});"
        );
        
        // then
        System.out.println(actualExec.getStdout());
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(user.getId()));
        assertThat(actual.getName(), is(user.getName()));

        JsonContent<Object> actualJson = this.jsonTester.from(actualExec.getStdout());
        Assertions.assertThat(actualJson)
                .hasJsonPathValue("$._id")
                .hasJsonPathValue("$.name")
                .doesNotHaveJsonPathValue("$._class");
    }
    
    @Test
    public void givenRemoveClassField_whenInsert_thenSuccessfullyCreateEntityClass() throws Exception {
        // given
        NoClassFieldUser user = new NoClassFieldUser();
        user.setName("myName");
        user = this.mongoTemplate.insert(user);

        // when
        NoClassFieldUser actual = this.mongoTemplate.findById(user.getId(), NoClassFieldUser.class);
        
        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(user.getId()));
        assertThat(actual.getName(), is(user.getName()));
    }
}
```  

<details><summary>db.noClassFieldUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{ 
	"_id" : ObjectId("61632637d3640816ec40f19b"), 
	"name" : "myName"
}
```  

</div>
</details>


### Custom Converter
`Custom Converter` 는 직접 `Converter` 인터페이스를 구현해서 사용자가 변환 로직을 정의하고, 
이를 `MongoCustomConversions` 객체에 설정하고 빈으로 선언해주는 방법으로 사용된다. 
`Custom Converter` 로는 패스워드 필드를 간단하게 `Base64` 인코딩하고 디코딩하는 예제로 진행한다.  

`EntityClass` 인 `EncodeDecodeUser` 객체에 대해서 인코딩과 디코딩을 수행하는 `Converter` 인터페이스의 구현체는 아래와 같다. 
`MongoDB` 로 저장 할때 `Base64` 로 인코딩하고, 
`MongoDB` 에서 `EntityClass` 로 만들때 디코딩을 수행한다.  

```java
@Component
@WritingConverter
public static class EncodeConverter implements Converter<EncodeDecodeUser, org.bson.Document> {
	@Override
	public org.bson.Document convert(EncodeDecodeUser source) {
		final org.bson.Document dbObject = new org.bson.Document();
		dbObject.put("name", source.getName());
		dbObject.put("password", Base64.getEncoder().encodeToString(source.getPassword().getBytes(StandardCharsets.UTF_8)));

		return dbObject;
	}
}

@Component
@ReadingConverter
public static class DecodeConverter implements Converter<org.bson.Document, EncodeDecodeUser> {
	@Override
	public EncodeDecodeUser convert(org.bson.Document source) {
		EncodeDecodeUser user = new EncodeDecodeUser();
		user.setId(Objects.toString(source.get("_id")));
		user.setName(Objects.toString(source.get("name")));
		byte[] decodeBytes = Base64.getDecoder().decode(Objects.toString(source.get("password")));
		user.setPassword(new String(decodeBytes));

		return user;
	}
}
```  

구현한 `Converter` 는 아래와 같이 `MongoCustomConversions` 빈으로 생성해 줘야 한다.  

```java
@Bean
public MongoCustomConversions mongoCustomConversions() {
	List<Converter<?, ?>> converters = new ArrayList<>();
	converters.add(new EncodeConverter());
	converters.add(new DecodeConverter());

	return new MongoCustomConversions(converters);
}
```  

아래는 실제 테스트 코드이다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class CustomConverterTest extends MongoDbTest {
    @Document
    @Data
    public static class EncodeDecodeUser {
        @Id
        private String id;
        private String name;
        private String password;
    }

    @Component
    @WritingConverter
    public static class EncodeConverter implements Converter<EncodeDecodeUser, org.bson.Document> {
        @Override
        public org.bson.Document convert(EncodeDecodeUser source) {
            final org.bson.Document dbObject = new org.bson.Document();
            dbObject.put("name", source.getName());
            dbObject.put("password", Base64.getEncoder().encodeToString(source.getPassword().getBytes(StandardCharsets.UTF_8)));

            return dbObject;
        }
    }

    @Component
    @ReadingConverter
    public static class DecodeConverter implements Converter<org.bson.Document, EncodeDecodeUser> {
        @Override
        public EncodeDecodeUser convert(org.bson.Document source) {
            EncodeDecodeUser user = new EncodeDecodeUser();
            user.setId(Objects.toString(source.get("_id")));
            user.setName(Objects.toString(source.get("name")));
            byte[] decodeBytes = Base64.getDecoder().decode(Objects.toString(source.get("password")));
            user.setPassword(new String(decodeBytes));

            return user;
        }
    }

    @TestConfiguration
    @RequiredArgsConstructor
    public static class TestConfig {
        private final MongoDatabaseFactory mongoDatabaseFactory;

        @Bean
        public MongoCustomConversions mongoCustomConversions() {
            List<Converter<?, ?>> converters = new ArrayList<>();
            converters.add(new EncodeConverter());
            converters.add(new DecodeConverter());

            return new MongoCustomConversions(converters);
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;


    @Test
    public void givenPasswordEncodeDecode_whenInsert_thenPasswordEncodedInMongoDB() throws Exception {
        // given
        EncodeDecodeUser user = new EncodeDecodeUser();
        user.setName("myName");
        user.setPassword("myPassword");
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.encodeDecodeUser.find({'_id' : ObjectId('" + user.getId() + "')});"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$._id")
                .isEqualTo("ObjectId(\"" + user.getId() + "\")");
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$.name")
                .isEqualTo("myName");
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$.password")
                .isEqualTo(Base64.getEncoder().encodeToString("myPassword".getBytes(StandardCharsets.UTF_8)));
    }

    @Test
    public void givenPasswordEncodeDecode_whenInsert_thenPasswordDecodedInEntityClass() throws Exception {
        // given
        EncodeDecodeUser user = new EncodeDecodeUser();
        user.setName("myName");
        user.setPassword("myPassword");
        user = this.mongoTemplate.insert(user);

        // when
        EncodeDecodeUser actual = this.mongoTemplate.findById(user.getId(), EncodeDecodeUser.class);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(user.getId()));
        assertThat(actual.getName(), is("myName"));
        assertThat(actual.getPassword(), is("myPassword"));
    }
}
```  

<details><summary>db.encodeDecodeUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{ 
	"_id" : ObjectId("6163271c16165470021fb8f0"), 
	"name" : "myName", 
	"password" : "bXlQYXNzd29yZA=="
}
```  

</div>
</details>



---
## Reference
[Spring Data MongoDB – Indexes, Annotations and Converters](https://www.baeldung.com/spring-data-mongodb-index-annotations-converter)  
[Spring boot: MongoDB: @PersistenceConstructor: Constructor to use while instantiating](https://self-learning-java-tutorial.blogspot.com/2019/12/spring-boot-mongodb-persistenceconstruc.html)  
[Use ZonedDateTime in Spring WebFlux (MongoDB Reactive)](https://dev.to/marttp/use-zoneddatetime-in-spring-webflux-mongodb-reactive-1408)  