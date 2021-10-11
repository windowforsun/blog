--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Data MongoDB Cascading Embedded Documents "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data MongoDB 에서 Embedded Documents 를 사용하는 방법과 Cascading 동작을 구현하는 방법에 대해서 알아보자'
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
    - MongoDBEventListener
    - DBRef
    - Cascade
toc: true
use_math: true
---  

> [Spring Data MongoDB 소개]({{site.baseurl}}{% link _posts/spring/2021-09-27-spring-concept-spring-data-mongodb-intro.md %})
을 기반으로 테스트 예제 코드를 진행한다. 

## Embedded Documents and Cascading
`Spring Data MongoDB` 의 기본적인 매핑에서는 도큐먼트간의 `parent-child` 관계가 있을 때 이를 매핑 시키거나, 
`Embedded Document` 를 별도의 컬렉션에 저장하는 기능은 제공하지 않는다.  

이번 포스트에서는 `@DBRef` 와 `Spring Data MongoDB` 의 `Event Listener`, `Custom Annotation` 을 사용해서 
위와 같이 `Embedded Document` 가 있을 때 이를 하나의 도큐먼트에 저장하는게 아니라, 
별도의 도큐먼트에 저장하고 분리된 두 도큐먼트를 매핑해서 읽어올 수 있도록 구현하는 방법에 대해 알아본다.  

`Embedded Doucment` 구조를 사용할때 아무설정도 하지 않은 상태에서는 아래와 같이 컬렉션의 특정 필드에 `Embedded Document` 내용이 저장된다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class NonCascadeTest extends MongoDbTest {

    @Document
    @Data
    public static class NoCascadeUser {
        @Id
        private String id;
        private String name;
        private Integer age;
        private NoCascadeEmail email;
    }

    @Document
    @Data
    private static class NoCascadeEmail {
        @Id
        private String id;
        private String value;
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @BeforeEach
    public void setUp() {
        this.mongoTemplate.dropCollection(NoCascadeUser.class);
    }

    @Test
    public void givenNoCascadingEmbeddedDocument_whenInsert_thenEmbeddedDocumentsSavedInRootDocumentMongoDB() throws Exception {
        // given
        NoCascadeEmail email = new NoCascadeEmail();
        email.setValue("my@email.com");
        NoCascadeUser user = new NoCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);
        
        // when
        Container.ExecResult actualUserExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.noCascadeUser.find({'_id' : ObjectId('" + user.getId() + "')}).pretty();"
        );

        // then
        System.out.println(actualUserExec.getStdout());
        assertThat(user.getId(), notNullValue());
        assertThat(email.getId(), nullValue());

        JsonContent<Object> actual = this.jsonTester.from(actualUserExec.getStdout());
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$.name")
                .isEqualTo("myName");
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$.age")
                .isEqualTo(20);
        Assertions.assertThat(actual)
                .extractingJsonPathValue("$.email.value")
                .isEqualTo("my@email.com");
    }

    @Test
    public void givenNoCascadingEmbeddedDocument_whenInsert_thenEmbeddedDocumentsSavedInRootDocument() {
        // given
        NoCascadeEmail email = new NoCascadeEmail();
        email.setValue("my@email.com");
        NoCascadeUser user = new NoCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        NoCascadeUser actual = this.mongoTemplate.findById(user.getId(), NoCascadeUser.class);

        // then
        assertThat(actual.getId(), not(emptyOrNullString()));
        assertThat(actual.getName(), is("myName"));
        assertThat(actual.getEmail(), notNullValue());
        assertThat(actual.getEmail().getId(), nullValue());
        assertThat(actual.getEmail().getValue(), is("my@email.com"));
    }
}
```  

<details><summary>db.noCascadeUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163f0dc3c0c5d1e73c6d99b"),
	"name" : "myName",
	"age" : 20,
	"email" : {
		"value" : "my@email.com"
	},
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.NonCascadeTest$NoCascadeUser"
}
```  

</div>
</details>

`User` 클래스의 `Email` 도큐먼트의 정보는 `User` 컬렉션의 `email` 필드에 함께 저장되는 것을 확인 할 수 있다.  

### @DBRef
`@DBRef` 를 사용하면 `Entity Class` 를 로드할 때 선언된 필드에 저장된 키와 도큐먼크 정보를 사용해서, 
실제론 별도의 컬렉션된 저장된 데이터를 하나의 도큐먼트로 매핑된 상태로 객체를 얻어올 수 있다.  

간단한 사용예시는 아래와 같다.  

```java
@Document
@Data
public static class DBRefUser {
	@Id
	private String id;
	private String name;
	private Integer age;

	@DBRef
	private DBRefEmail email;
}

@Document
@Data
public static class DBRefEmail {
	@Id
	private String id;
	private String value;
}
```  

`User` 도큐먼트의 `email` 필드에 `Email` 도큐먼크의 데이터가 저장되는 것이 아니라, 
`Email` 도큐먼트에 대한 컬렉션 및 아이디에 대한 정보만 저장된다. 
그리고 실제 데이터는 `Email` 컬렉션에 저장된다. 
아래는 실제 수행한 테스트 코드이다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class DBRefEmbeddedDocumentTest extends MongoDbTest {
    @Document
    @Data
    public static class DBRefUser {
        @Id
        private String id;
        private String name;
        private Integer age;

        @DBRef
        private DBRefEmail email;
    }

    @Document
    @Data
    public static class DBRefEmail {
        @Id
        private String id;
        private String value;
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Test
    public void givenDBRefDocument_whenInsert_thenInsertedSeparatedCollectionsInMongoDB() throws Exception {
        // given
        DBRefEmail email = new DBRefEmail();
        email.setValue("my@email.com");
        email = this.mongoTemplate.insert(email);
        DBRefUser user = new DBRefUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualUserExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.dBRefUser.find({'_id' : ObjectId('" + user.getId() + "')}).pretty();"
        );
        Container.ExecResult actualEmailExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.dBRefEmail.find({'_id' : ObjectId('" + email.getId() + "')}).pretty();"
        );

        // then
        System.out.println(actualUserExec.getStdout());
        System.out.println(actualEmailExec.getStdout());
        JsonContent<Object> actualUser = this.jsonTester.from(actualUserExec.getStdout());
        JsonContent<Object> actualEmail = this.jsonTester.from(actualEmailExec.getStdout());
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.name")
                .isEqualTo("myName");
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.email")
                .asString()
                .contains("dBRefEmail");
        Assertions.assertThat(actualEmail)
                .extractingJsonPathValue("$.value")
                .isEqualTo("my@email.com");
    }


    @Test
    public void givenDBRefDocument_whenInsertAndFind_thenLoadWithEmbeddedDocumentInEntityClass() throws Exception {
        // given
        DBRefEmail email = new DBRefEmail();
        email.setValue("my@email.com");
        email = this.mongoTemplate.insert(email);
        DBRefUser user = new DBRefUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        DBRefUser actual = this.mongoTemplate.findById(user.getId(), DBRefUser.class);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(user.getId()));
        assertThat(actual.getName(), is("myName"));
        assertThat(actual.getEmail(), notNullValue());
        assertThat(actual.getEmail().getId(), is(email.getId()));
        assertThat(actual.getEmail().getValue(), is("my@email.com"));
    }
}
```  

<details><summary>db.dBRefUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163f4a484f8042140b75308"),
	"name" : "myName",
	"age" : 20,
	"email" : DBRef("dBRefEmail", ObjectId("6163f4a484f8042140b75307")),
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.DBRefEmbeddedDocumentTest$DBRefUser"
}
```  

</div>
</details>

<details><summary>db.dBRefEmail.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163f4a484f8042140b75307"),
	"value" : "my@email.com",
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.DBRefEmbeddedDocumentTest$DBRefEmail"
}
```  

</div>
</details>


### Basic Cascade Save
`@DBRef` 를 사용하면 `Embedded Doucment` 를 별도의 컬렉션으로 저장할 수 있는 부분을 먼저 알아봤다. 
하지만 `Embedded Document` 의 데이터는 직접 저장을 해줘야 정상적으로 매핑이 가능하다.  

이번에는 `Spring Data MongoDB` 의 이벤트 리스너를 사용해서 특정 `Entity Class` 에 대해서 
`Cascading` 동작 즉 부모 도큐먼트를 저장하면 자동으로 `Enbedded Document` 까지 저장될 수 있도록 구현을 해본다.  

구현 방법은 `MongoDBEventListener` 에서 `onBeforeSave` 이벤트를 사용한다. 
부모 도큐먼트를 저장하기 전에 `Embedded Document` 의 필드가 `null` 이 아니라면, 
`Embedded Documents` 에 대해서 `save()` 동작을 수행해 주는 방식이다.  

```java
public static class UserCascadeSaveMongoEventListener extends AbstractMongoEventListener<Object> {
	@Autowired
	private MongoOperations mongoOperations;

	@Override
	public void onBeforeConvert(BeforeConvertEvent<Object> event) {
		Object source = event.getSource();

		if((source instanceof BasicCascadeUser) && ((BasicCascadeUser)source).getEmail() != null) {
			mongoOperations.save(((BasicCascadeUser)source).getEmail());
		}
	}
}
```  

선언한 `MongoDBEventListener` 를 빈으로 등록해 주면 된다. 

```java
@Bean
public UserCascadeSaveMongoEventListener userCascadeSaveMongoEventListener() {
	return new UserCascadeSaveMongoEventListener();
}
```  

위 이벤트 리스터를 사용하면 `User` 를 저장할때 `email` 필드가 비어있지 않다면,
`Embedded Document` 인 `Email` 컬렉션에 먼저 저장하고 나서 `User` 를 저장한다. 
아래는 실제로 테스트한 코드이다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class BasicCascadeTest extends MongoDbTest {

    @Document
    @Data
    public static class BasicCascadeUser {
        @Id
        private String id;
        private String name;
        private Integer age;

        @DBRef
        private BasicCascadeEmail email;

    }

    @Document
    @Data
    private static class BasicCascadeEmail {
        @Id
        private String id;
        private String value;
    }

    public static class UserCascadeSaveMongoEventListener extends AbstractMongoEventListener<Object> {
        @Autowired
        private MongoOperations mongoOperations;

        @Override
        public void onBeforeConvert(BeforeConvertEvent<Object> event) {
            Object source = event.getSource();

            if((source instanceof BasicCascadeUser) && ((BasicCascadeUser)source).getEmail() != null) {
                mongoOperations.save(((BasicCascadeUser)source).getEmail());
            }
        }
    }

    @TestConfiguration
    public static class MongoConfig {
        @Bean
        public UserCascadeSaveMongoEventListener userCascadeSaveMongoEventListener() {
            return new UserCascadeSaveMongoEventListener();
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;


    @BeforeEach
    public void setUp() {
        this.mongoTemplate.dropCollection(BasicCascadeUser.class);
        this.mongoTemplate.dropCollection(BasicCascadeEmail.class);
    }

    @Test
    public void givenCascadingEmbeddedDocument_whenInsert_thenInsertedSeparatedCollectionsInMongoDB() throws Exception {
        // given
        BasicCascadeEmail email = new BasicCascadeEmail();
        email.setValue("my@email.com");
        BasicCascadeUser user = new BasicCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualUserExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.basicCascadeUser.find({'_id' : ObjectId('" + user.getId() + "')}).pretty();"
        );
        Container.ExecResult actualEmailExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.basicCascadeEmail.find({'_id' : ObjectId('" + email.getId() + "')}).pretty();"
        );

        // then
        System.out.println(actualUserExec.getStdout());
        System.out.println(actualEmailExec.getStdout());
        assertThat(user.getId(), not(emptyOrNullString()));
        assertThat(email.getId(), not(emptyOrNullString()));

        JsonContent<Object> actualUser = this.jsonTester.from(actualUserExec.getStdout());
        JsonContent<Object> actualEmail = this.jsonTester.from(actualEmailExec.getStdout());
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.name")
                .isEqualTo("myName");
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.email")
                .asString()
                .contains("basicCascadeEmail");
        Assertions.assertThat(actualEmail)
                .extractingJsonPathValue("$.value")
                .isEqualTo("my@email.com");

    }

    @Test
    public void givenCascadingEmbeddedDocument_whenInsertAndFind_thenLoadWithEmbeddedDocumentInEntityClass() {
        // given
        BasicCascadeEmail email = new BasicCascadeEmail();
        email.setValue("my@email.com");
        BasicCascadeUser user = new BasicCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        BasicCascadeUser actualUser = this.mongoTemplate.findById(user.getId(), BasicCascadeUser.class);
        BasicCascadeEmail actualEmail = this.mongoTemplate.findById(email.getId(), BasicCascadeEmail.class);

        // then
        assertThat(actualUser, notNullValue());
        assertThat(actualUser.getId(), is(user.getId()));
        assertThat(actualUser.getName(), is("myName"));
        assertThat(actualUser.getEmail(), notNullValue());
        assertThat(actualUser.getEmail().getId(), is(email.getId()));
        assertThat(actualUser.getEmail().getValue(), is("my@email.com"));

        assertThat(actualEmail, notNullValue());
        assertThat(actualEmail.getId(), is(actualUser.getEmail().getId()));
        assertThat(actualEmail.getValue(), is("my@email.com"));
    }
}
```  

<details><summary>db.basicCascadeUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163f853b0713375df9be6d7"),
	"name" : "myName",
	"age" : 20,
	"email" : DBRef("basicCascadeEmail", ObjectId("6163f853b0713375df9be6d6")),
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.BasicCascadeTest$BasicCascadeUser"
}
```  

</div>
</details>

<details><summary>db.basicCascadeEmail.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163f853b0713375df9be6d6"),
	"value" : "my@email.com",
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.BasicCascadeTest$BasicCascadeEmail"
}
```  

</div>
</details>


### Generic Cascade Save
바로 위에서 알아본 `Basic Cascade Save` 는 특정 `Entity Class` 에 대해서만 동작이 가능한 
`Event Listener` 를 만들어 사용하는 방법이였다. 
이러한 구조는 관련 동작을 필요로하는 `Entity Class` 가 늘어나게 되면 해당하는 클래스와 작업도 늘어나기 때문에 이를 
`Custom Annotation` 을 사용해서 공통지어 일반화 시켜본다.  

먼저 `CasCade Save` 용으로 사용할 `@CascadeSave` 를 아래와 같이 선언해준다.  

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public static @interface CascadeSave {

}
```  

`MongoDBEventistener` 를 아래와 같이 현재 저장하는 `Entity Class` 가 `Cascade Save` 동작을 
수행해야 하는지에 대한 판단을 위해 `Reflection` 콜백인 `CascadeCallback` 을 호출한다.  


```java
public static class CascadeSaveMongoEventListener extends AbstractMongoEventListener<Object> {
	@Autowired
	private MongoOperations mongoOperations;

	@Override
	public void onBeforeConvert(BeforeConvertEvent<Object> event) {
		final Object source = event.getSource();
		ReflectionUtils.doWithFields(source.getClass(), new CascadeCallback(source, this.mongoOperations, true));
	}
}
```  

이벤트 리스너에서 호출한 `Reflection` 콜백은 아래와 같다. 
콜백에서는 `@DBRef` 와 `@CascadeSave` 가 모두 선언된 필드인지 검사하고, 
충족한다면 필드에 대해서 별도로 `save()` 를 호출해서 저장한다.  

```java
public static class CascadeCallback implements ReflectionUtils.FieldCallback {
	private Object source;
	private MongoOperations mongoOperations;

	public CascadeCallback(final Object source, final MongoOperations mongoOperations) {
		this.source = source;
		this.mongoOperations = mongoOperations;
	}

	@Override
	public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
		ReflectionUtils.makeAccessible(field);

		if(field.isAnnotationPresent(DBRef.class) && field.isAnnotationPresent(CascadeSave.class)) {
			final Object fieldValue = field.get(this.source);

			if(fieldValue != null) {
				final FieldCallback callback = new FieldCallback();
				ReflectionUtils.doWithFields(fieldValue.getClass(), callback);
				this.mongoOperations.save(fieldValue);
			}
		}
	}
}
```  

`CascadeCallback` 을 보면 `FieldCallback` 을 다시 호출하는 부분이 있다. 
이는 `@DBRef` 와 `@CascadeSave` 가 모두 선언된 즉 자동으로 저장할 `Embedded Document` 에 해당하는 
`Entity Class` 에 대해서 `@Id` 가 존재하는지 등에 대한 검사를 수행할 수 있다. 
지금은 `@Id` 가 선언된 필드가 있는지만 검사해서 체크하고 있다.  

```java
@Getter
public static class FieldCallback implements ReflectionUtils.FieldCallback {
	private boolean idFound;

	@Override
	public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
		ReflectionUtils.makeAccessible(field);

		if(field.isAnnotationPresent(Id.class)) {
			idFound = true;
		}
	}
}
```  

설명하면서 나열한 설정을 실제로 테스트하면 아래와 같다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class GenericCascadeTest extends MongoDbTest {
    @Document
    @Data
    public static class GenericCascadeUser {
        @Id
        private String id;
        private String name;
        private Integer age;

        @DBRef
        @CascadeSave
        private GenericCascadeEmail email;
    }

    @Document
    @Data
    public static class GenericCascadeEmail {
        @Id
        private String id;
        private String value;
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    public static @interface CascadeSave {

    }

    public static class CascadeSaveMongoEventListener extends AbstractMongoEventListener<Object> {
        @Autowired
        private MongoOperations mongoOperations;

        @Override
        public void onBeforeConvert(BeforeConvertEvent<Object> event) {
            final Object source = event.getSource();
            ReflectionUtils.doWithFields(source.getClass(), new CascadeCallback(source, this.mongoOperations));
        }
    }

    @Getter
    public static class CascadeCallback implements ReflectionUtils.FieldCallback {
        private Object source;
        private MongoOperations mongoOperations;

        public CascadeCallback(final Object source, final MongoOperations mongoOperations) {
            this.source = source;
            this.mongoOperations = mongoOperations;
        }

        @Override
        public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
            ReflectionUtils.makeAccessible(field);

            if(field.isAnnotationPresent(DBRef.class) && field.isAnnotationPresent(CascadeSave.class)) {
                final Object fieldValue = field.get(this.source);

                if(fieldValue != null) {
                    final FieldCallback callback = new FieldCallback();
                    ReflectionUtils.doWithFields(fieldValue.getClass(), callback);
                    this.mongoOperations.save(fieldValue);
                }
            }
        }
    }

    @Getter
    public static class FieldCallback implements ReflectionUtils.FieldCallback {
        private boolean idFound;

        @Override
        public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
            ReflectionUtils.makeAccessible(field);

            if(field.isAnnotationPresent(Id.class)) {
                idFound = true;
            }
        }
    }

    @TestConfiguration
    public static class MongoConfig {
        @Bean
        public CascadeSaveMongoEventListener cascadeSaveMongoEventListener () {
            return new CascadeSaveMongoEventListener();
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Test
    public void givenGenericCascadeSave_whenInsert_thenInsertedSeparatedCollectionsInMongoDB() throws Exception {
        // given
        GenericCascadeEmail email = new GenericCascadeEmail();
        email.setValue("my@email.com");
        GenericCascadeUser user = new GenericCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualUserExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.genericCascadeUser.find({'_id' : ObjectId('" + user.getId() + "')}).pretty();"
        );
        Container.ExecResult actualEmailExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.genericCascadeEmail.find({'_id' : ObjectId('" + email.getId() + "')}).pretty();"
        );

        // then
        System.out.println(actualUserExec.getStdout());
        System.out.println(actualEmailExec.getStdout());
        assertThat(user.getId(), not(emptyOrNullString()));
        assertThat(email.getId(), not(emptyOrNullString()));

        JsonContent<Object> actualUser = this.jsonTester.from(actualUserExec.getStdout());
        JsonContent<Object> actualEmail = this.jsonTester.from(actualEmailExec.getStdout());
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.name")
                .isEqualTo("myName");
        Assertions.assertThat(actualUser)
                .extractingJsonPathValue("$.email")
                .asString()
                .contains("genericCascadeEmail");
        Assertions.assertThat(actualEmail)
                .extractingJsonPathValue("$.value")
                .isEqualTo("my@email.com");
    }

    @Test
    public void givenGenericCascadeSave_whenInsertAndFind_thenLoadWithEmbeddedDocumentInEntityClass() {
        // given
        GenericCascadeEmail email = new GenericCascadeEmail();
        email.setValue("my@email.com");
        GenericCascadeUser user = new GenericCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);

        // when
        GenericCascadeUser actualUser = this.mongoTemplate.findById(user.getId(), GenericCascadeUser.class);
        GenericCascadeEmail actualEmail = this.mongoTemplate.findById(email.getId(), GenericCascadeEmail.class);

        // then
        System.out.println(actualUser);
        System.out.println(actualEmail);
        assertThat(actualUser, notNullValue());
        assertThat(actualUser.getId(), is(user.getId()));
        assertThat(actualUser.getName(), is("myName"));
        assertThat(actualUser.getEmail(), notNullValue());
        assertThat(actualUser.getEmail().getId(), is(email.getId()));
        assertThat(actualUser.getEmail().getValue(), is("my@email.com"));

        assertThat(actualEmail, notNullValue());
        assertThat(actualEmail.getId(), is(actualUser.getEmail().getId()));
        assertThat(actualEmail.getValue(), is("my@email.com"));
    }

    @Test
    public void givenGenericCascade_whenUpdate_thenUpdatedAllDocuments() {
        GenericCascadeEmail email = new GenericCascadeEmail();
        email.setValue("my@email.com");
        GenericCascadeUser user = new GenericCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);
        user.setName("myName2");
        email.setValue("my2@email.com");
        user = this.mongoTemplate.save(user);

        // when
        GenericCascadeUser actualUser = this.mongoTemplate.findById(user.getId(), GenericCascadeUser.class);
        GenericCascadeEmail actualEmail = this.mongoTemplate.findById(email.getId(), GenericCascadeEmail.class);

        // then
        assertThat(actualUser, notNullValue());
        assertThat(actualUser.getId(), is(user.getId()));
        assertThat(actualUser.getName(), is("myName2"));
        assertThat(actualUser.getEmail(), notNullValue());
        assertThat(actualUser.getEmail().getId(), is(email.getId()));
        assertThat(actualUser.getEmail().getValue(), is("my2@email.com"));

        assertThat(actualEmail, notNullValue());
        assertThat(actualEmail.getId(), is(actualUser.getEmail().getId()));
        assertThat(actualEmail.getValue(), is("my2@email.com"));
    }
}
```  

<details><summary>db.genericCascadeUser.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163fd68b8196454705d44db"),
	"name" : "myName",
	"age" : 20,
	"email" : DBRef("genericCascadeEmail", ObjectId("6163fd68b8196454705d44da")),
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.GenericCascadeTest$GenericCascadeUser"
}
```  

</div>
</details>

<details><summary>db.genericCascadeEmail.find({'_id' : ObjectId('{_id}')}) 출력</summary>
<div markdown="1">

```json
{
	"_id" : ObjectId("6163fd68b8196454705d44da"),
	"value" : "my@email.com",
	"_class" : "com.windowforsun.mongoexam.intromongo.cascade.GenericCascadeTest$GenericCascadeEmail"
}
```  

</div>
</details>


### Generic Cascade Remove
위에서 설명한 부분까지하면 `Embedded Document` 에 대해서 `Save`, `Update` 에 대한 동작은 
모두 `Cascading` 하게 가능하다. 
하지만 `Remove`(삭제)에 대한 동작은 좀더 고려해야할 부분이 많기 때문에 상황에 따라서 잘 사용해야 한다.  

만약 `Embedded Document` 와 1:1 관계라면 아래 설명할 `CascadeRemove` 가 관리에 도움이 될 수 있겠지만, 
1:N 관계라면 A, B가 동일한 `Embedded Document` 를 사용할때 A 도큐먼트 삭제로 인해 B 도큐먼트의 `Embedded Document` 까지 삭제될 수 있기 때문이다.  

이후 부터 설명하는 `CascadeRemove` 는 1:1 관계에서 사용할 수 있는 간단한 방식이다. 
`CascadeSave` 도 함께 구현된 코드로 `CascadeSave` 에서 `onBeforeDelete` 이벤트에 대한 부분과 
`CascadeCallback` 의 판별 조건에 대한 부분이 추가 및 변경 돼었다.  


```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class GenericCascadeTest extends MongoDbTest {
    @Document
    @Data
    public static class GenericCascadeUser {
        @Id
        private String id;
        private String name;
        private Integer age;

        @DBRef
        @CascadeSave
        @CascadeRemove
        private GenericCascadeEmail email;
    }

    @Document
    @Data
    public static class GenericCascadeEmail {
        @Id
        private String id;
        private String value;
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    public static @interface CascadeSave {

    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    public static @interface CascadeRemove {
    
    }
    
    public static class CascadeSaveMongoEventListener extends AbstractMongoEventListener<Object> {
        @Autowired
        private MongoOperations mongoOperations;
        
        @Override
        public void onBeforeConvert(BeforeConvertEvent<Object> event) {
            final Object source = event.getSource();
            ReflectionUtils.doWithFields(source.getClass(), new CascadeCallback(source, this.mongoOperations, true));
        }

        @Override
        public void onBeforeDelete(BeforeDeleteEvent<Object> event) {
            final org.bson.Document document = event.getDocument();
            final Object source = this.mongoOperations.findById(document.get("_id"), event.getType());
            ReflectionUtils.doWithFields(source.getClass(), new CascadeCallback(source, this.mongoOperations, false));
        }
    }

    @Getter
    public static class CascadeCallback implements ReflectionUtils.FieldCallback {
        private Object source;
        private MongoOperations mongoOperations;
        private boolean isSave;

        public CascadeCallback(final Object source, final MongoOperations mongoOperations, boolean isSave) {
            this.source = source;
            this.mongoOperations = mongoOperations;
            this.isSave = isSave;
        }

        @Override
        public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
            ReflectionUtils.makeAccessible(field);

            if(field.isAnnotationPresent(DBRef.class)) {
                final Object fieldValue = field.get(this.source);

                if(fieldValue != null) {
                    final FieldCallback callback = new FieldCallback();
                    ReflectionUtils.doWithFields(fieldValue.getClass(), callback);
                    if(this.isSave && field.isAnnotationPresent(CascadeSave.class)) {
                        this.mongoOperations.save(fieldValue);
                    } else if(!this.isSave && field.isAnnotationPresent(CascadeRemove.class)) {
                        this.mongoOperations.remove(fieldValue);
                    }
                }

            }
        }
    }

    @Getter
    public static class FieldCallback implements ReflectionUtils.FieldCallback {
        private boolean idFound;

        @Override
        public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
            ReflectionUtils.makeAccessible(field);

            if(field.isAnnotationPresent(Id.class)) {
                idFound = true;
            }
        }
    }

    @TestConfiguration
    public static class MongoConfig {
        @Bean
        public CascadeSaveMongoEventListener cascadeSaveMongoEventListener () {
            return new CascadeSaveMongoEventListener();
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;


    @Test
    public void givenGenericCascade_whenDelete_thenRemovedAllDocuments() throws Exception {
        GenericCascadeEmail email = new GenericCascadeEmail();
        email.setValue("my@email.com");
        GenericCascadeUser user = new GenericCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);
        this.mongoTemplate.remove(user);

        // when
        Container.ExecResult actualUserExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.genericCascadeUser.find({'_id' : ObjectId('" + user.getId() + "')}).pretty();"
        );
        Container.ExecResult actualEmailExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.genericCascadeEmail.find({'_id' : ObjectId('" + email.getId() + "')}).pretty();"
        );

        // then
        assertThat(actualUserExec.getStdout(), emptyOrNullString());
        assertThat(actualEmailExec.getStdout(), emptyOrNullString());
    }

    @Test
    public void givenGenericCascadeRemove_whenDelete_thenAllEntityClassIsNull() {
        // given
        GenericCascadeEmail email = new GenericCascadeEmail();
        email.setValue("my@email.com");
        GenericCascadeUser user = new GenericCascadeUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmail(email);
        user = this.mongoTemplate.insert(user);
        this.mongoTemplate.remove(user);

        // when
        GenericCascadeUser actualUser = this.mongoTemplate.findById(user.getId(), GenericCascadeUser.class);
        GenericCascadeEmail actualEmail = this.mongoTemplate.findById(email.getId(), GenericCascadeEmail.class);

        // then
        assertThat(actualUser, nullValue());
        assertThat(actualEmail, nullValue());
    }
}
```  



---
## Reference
[Custom Cascading in Spring Data MongoDB](https://www.baeldung.com/cascading-with-dbref-and-lifecycle-events-in-spring-data-mongodb)  
[Spring Data MongoDB cascade save on DBRef objects](https://www.javacodegeeks.com/2013/11/spring-data-mongodb-cascade-save-on-dbref-objects.html)  