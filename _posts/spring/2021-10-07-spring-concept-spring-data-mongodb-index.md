--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Data MongoDB Indexes"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data MongoDB 에서 Index 를 설정하고 생성하는 방법에 대해 알아보자'
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
    - Indexed
    - CompoundIndex
toc: true
use_math: true
---  

## Spring Data MongoDB Indexes
[Spring Data MongoDB 소개]({{site.baseurl}}{% link _posts/spring/2021-09-27-spring-concept-spring-data-mongodb-intro.md %})
을 기반으로 해서 `Spring Data MongoDB` 에서 인덱스를 설정하고 사용하는 방법에 대해 알아보고 테스트 코드를 작성해 본다. 
`Spring Data MongoDB` 에서는 인덱스를 위해 아래와 같은 `Annotation` 을 제공한다.  

Annotation|Desc
---|---
@Indexed|단일 인덱스로 사용할 필드에 선언한다. 
@CompoundIndex|도큐먼트 클래스 레벨에 선언하고, 복합 필드를 사용해서 인덱스를 정의할 때 사용한다. 
@TextIndexed|텍스트 인덱스로 사용할 필드에 선언한다. 
@GeoSpacialIndexed|지리공안 인덱스로 사용할 필드에선언한다. 

테스트 코드에서는 `MongoDB` 에서 특정 컬렉션에 생성된 인덱스 정보를 리스팅해주는 아래 명령어를 사용해서 인덱스가 정상적으로 생성됐는지 확인 한다.  

```bash
db.<collection-name>.getIndexes()
```  


### Index 설정
`Spring Data MongoDB` 에서 인덱스 생성에 대한 아래와 같은 몇가지 설정 방법을 제공한다. 

- `Spring Boot Properties` 를 사용한 자동 설정을 통한 `auto-index`
- `Custom MongoDB Config` 관련 설정 클래스를 바탕으로 `auto-index`
- `Programmatically` 한 방법으로 직접 인덱스 설정 

`Programmatically` 사용하는 방법은 가장 나중에 알아보고, 
우선 `auto-index` 를 활성화 해서 인덱스를 생성해 줄 수 있도록 설정을 해본다.  

먼저 아무런 설정을 하지 않은 상태에서 `@Indexed` 를 필드에 선언하더라도 인덱스는 생성되지 않는다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class NoAutoIndexTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Document
    @Data
    public static class NoAutoIndexUser {
        @Id
        private String id;
        @Indexed
        private String name;
        private Integer age;
    }

    @Test
    public void givenInsertNoIndexEntity_whenMongoShGetIndexes_thenOnlyIdIndex() throws Exception {
        // given
        NoAutoIndexUser user = new NoAutoIndexUser();
        user.setName("myName");
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.noAutoIndexUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_")
                .doesNotContain("name");
        assertThat(actual)
                .hasJsonPath("$[*].key._id");
        assertThat(actual)
                .doesNotHaveJsonPathValue("$[*].key.name");
    }
}
```  

<details><summary>db.noAutoIndexUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[ 
	{ 
		"v" : 2, 
		"key" : { "_id" : 1 }, 
		"name" : "_id_"
	} 
]
```  

</div>
</details>

`ExamUser` 라는 도큐먼트의 `name` 필드에 `@Indexed` 를 선언했지만 `MongoDB` 에서는 아무런 인덱스로 생성되지 않은 것을 확인 할 수 있다.  

#### Spring Boot Properties
`Spring Boot` 의 자동설정 `Properties` 를 사용하는 방법은 아래와 같이 아주 간단하다.  

```yaml
spring:
	data:
		mongodb:
			auto-index-creation: true
```  

위 설정을 사용해서 실제 테스트 코드를 수행하면 아래와 같다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test",
                "spring.data.mongodb.auto-index-creation=true"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class AutoIndexByAutoConfigTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Document
    @Data
    public static class AutoConfigUser {
        @Id
        private String id;
        @Indexed
        private String name;
        private Integer age;
    }

    @Test
    public void givenAutoIndexConfigAndNameIndex_whenInsertEntity_thenCreatedNameIndex() throws Exception {
        // given
        AutoConfigUser user = new AutoConfigUser();
        user.setName("myName");
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.autoConfigUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_", "name");
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key._id")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.name")
                .contains(1);
    }
}
```  

<details><summary>db.autoConfigUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"name" : 1
		},
		"name" : "name"
	}
]
```  

</div>
</details>

`Properties` 에 `spring.data.mongodb.auto-index-creation=true` 을 설정하고 인덱스로 사용하고 싶은 필드에 `@Indexed` 를 선언해주면, 
`MongoDB` 에도 인덱스가 실제로 생긴것을 확인 할 수 있다.  

#### Custom MongoDB Config
다음은 `Spring Boot` 자동 설정을 사용하지 않더라도 `Auto-Index` 를 설정할 수 있는 방법으로, 
아래와 같이 `AbstractMongoClientConfiguration` 의 구현체를 만들어 설정하는 방법이다.  

```java
@Configuration
public class CustomMongoConfig extends AbstractMongoClientConfiguration {

	@Override
	protected boolean autoIndexCreation() {
		return true;
	}
	
	// Other Override Methods ..
}
```  

위와 같이 커스텀한 설정 클래스를 사용한 테스트 코드는 아래와 같다.  

```java
@JsonTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@ExtendWith(SpringExtension.class)
public class AutoIndexByCustomMongoConfigTest extends MongoDbTest {
    @TestConfiguration
    public static class CustomMongoConfig extends AbstractMongoClientConfiguration {
        @Value("${spring.data.mongodb.database}")
        private String database;
        @Value("${spring.data.mongodb.host}")
        private String host;
        @Value("${spring.data.mongodb.port}")
        private int port;

        @Override
        protected String getDatabaseName() {
            return "test";
        }

        @Override
        public MongoClient mongoClient() {
            ConnectionString connectionString = new ConnectionString("mongodb://" + this.host + ":" + this.port + "/" + this.database);
            MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
                    .applyConnectionString(connectionString)
                    .build();

            return MongoClients.create(mongoClientSettings);
        }

        @Override
        protected boolean autoIndexCreation() {
            return true;
        }
    }

    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Document
    @Data
    public static class CustomConfigUser {
        @Id
        private String id;
        @Indexed
        private String name;
        private Integer age;
    }

    @Test
    public void givenAutoIndexConfigAndNameIndex_whenInsertEntity_thenCreatedNameIndex() throws Exception {
        // given
        CustomConfigUser user = new CustomConfigUser();
        user.setName("myName");
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.customConfigUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_", "name");
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key._id")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.name")
                .contains(1);
    }
}
```  

<details><summary>db.customConfigUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"name" : 1
		},
		"name" : "name"
	}
]
```  

</div>
</details>

### CompoundIndex
복합 필드를 사용해서 도큐먼트의 인덱스를 생성할 수 있는 `CompoundIndex` 는 아래와 같이 도튜먼크 클래스 레벨에 선언하는 방법으로 사용 할 수 있다. 
아래는 `Embedded` 도큐먼트인 `EmailAddress` 의 `id` 필드와 `age` 필드를 사용해서 `CompoundIndex` 를 생성한 예제이다.   

```java
@Document
@Data
@CompoundIndexes({
		@CompoundIndex(name = "email_age_index", def = "{'email.id' : 1, 'age' : 1}")
})
public class CompoundIndexUser {
	@Id
	private String id;
	@Indexed
	private String name;
	private int age;
	private EmailAddress emailAddress;

}

@Data
public class EmailAddress {
	private String id;
	private String value;
}
```  

위 `CompoundIndex` 를 사용한 테스트는 아래와 같다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test",
                "spring.data.mongodb.auto-index-creation=true"
        }
)
@ExtendWith(SpringExtension.class)
@AutoConfigureJsonTesters
public class CompoundIndexTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @Document
    @Data
    @CompoundIndexes({
            @CompoundIndex(name = "email_age_index", def = "{'email.id' : 1, 'age' : 1}")
    })
    public static class CompoundIndexUser {
        @Id
        private String id;
        @Indexed
        private String name;
        private int age;
        private EmailAddress emailAddress;

    }

    @Data
    public static class EmailAddress {
        private String id;
        private String value;
    }

    @Test
    public void givenCompoundIndex_whenInsertEntity_thenCreatedCompoundIndex() throws Exception {
        // given
        EmailAddress emailAddress = new EmailAddress();
        emailAddress.setId("myEmailId");
        emailAddress.setValue("myEmail");
        CompoundIndexUser user = new CompoundIndexUser();
        user.setName("myName");
        user.setAge(30);
        user.setEmailAddress(emailAddress);
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.compoundIndexUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_", "name", "email_age_index");
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key._id")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.name")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.['email.id']")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.age")
                .contains(1);
    }
}
```  

<details><summary>db.compoundIndexUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"email.id" : 1,
			"age" : 1
		},
		"name" : "email_age_index"
	},
	{
		"v" : 2,
		"key" : {
			"name" : 1
		},
		"name" : "name"
	}
]
```  

</div>
</details>

단일 인덱스로 설정한 `name` 뿐만아니라 `CompoundIndex` 로 설정한 인덱스까지 모두 생성된 것을 확인 할 수 있다.  


### Programmatically Index
`Spring Data MongoDB` 에서 제공하는 `Auto-Index` 를 사용할 수도 있지만, 
`MongoTemplate` 을 사용하면 직접 원하는 인덱스를 생성할 수도 있다. 
아래는 단일 인덱스와 복합 인덱스 모두 `MongoTemplate` 을 사용해서 생성하는 테스트 코드이다.  

```java
@DataMongoTest(
        properties = {
                "spring.data.mongodb.database=test"
        }
)
@AutoConfigureJsonTesters
@ExtendWith(SpringExtension.class)
public class ProgrammaticallyIndexTest extends MongoDbTest {
    @Autowired
    private MongoTemplate mongoTemplate;
    @Autowired
    private BasicJsonTester jsonTester;

    @BeforeEach
    public void setUp() {
        this.mongoTemplate.dropCollection(ProgrammaticallyIndexUser.class);
        this.mongoTemplate.indexOps(ProgrammaticallyIndexUser.class).dropAllIndexes();
    }

    @Document
    @Data
    public static class ProgrammaticallyIndexUser {
        @Id
        private String id;
        private String name;
        private Integer age;
        private EmailAddress emailAddress;
    }

    @Data
    public static class EmailAddress {
        private String id;
        private String value;
    }

    @Test
    public void givenProgrammaticallyIndexAndNameIndex_whenInsertEntity_thenCreatedNameIndex() throws Exception {
        // given
        ProgrammaticallyIndexUser user = new ProgrammaticallyIndexUser();
        user.setName("myName");
        this.mongoTemplate
                .indexOps(ProgrammaticallyIndexUser.class)
                .ensureIndex(new Index().named("name").on("name", Sort.Direction.ASC));
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.programmaticallyIndexUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_", "name");
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key._id")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.name")
                .contains(1);
    }

    @Test
    public void givenProgrammaticallyCompoundIndex_whenInsertEntity_thenCreatedCompoundIndex() throws Exception {
        // given
        EmailAddress emailAddress = new EmailAddress();
        emailAddress.setId("myEmailId");
        emailAddress.setValue("myEmail");
        ProgrammaticallyIndexUser user = new ProgrammaticallyIndexUser();
        user.setName("myName");
        user.setAge(20);
        user.setEmailAddress(emailAddress);
        IndexDefinition compoundIndex1 = new CompoundIndexDefinition(
                new org.bson.Document().append("age", 1).append("name", 1)
        );
        IndexDefinition compoundIndex2 = new CompoundIndexDefinition(
                new org.bson.Document().append("age", 1).append("email.id", 1)
        );
        this.mongoTemplate
                .indexOps(ProgrammaticallyIndexUser.class)
                .ensureIndex(compoundIndex1);
        this.mongoTemplate
                .indexOps(ProgrammaticallyIndexUser.class)
                .ensureIndex(compoundIndex2);
        user = this.mongoTemplate.insert(user);

        // when
        Container.ExecResult actualExec = MongoDbTest.MONGO_DB.execInContainer(
                "mongo",
                "--quiet",
                "--eval",
                "db.getSiblingDB('test'); db.programmaticallyIndexUser.getIndexes();"
        );

        // then
        System.out.println(actualExec.getStdout());
        JsonContent<Object> actual = this.jsonTester.from(actualExec.getStdout());
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].name")
                .contains("_id_", "age_1_name_1", "age_1_email.id_1");
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key._id")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.age")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.name")
                .contains(1);
        assertThat(actual)
                .extractingJsonPathArrayValue("$[*].key.['email.id']")
                .contains(1);
    }
}
```  

<details><summary>단일 인덱스 db.programmaticallyIndexUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"name" : 1
		},
		"name" : "name"
	}
]
```  

</div>
</details>


<details><summary>복합 인덱스 db.programmaticallyIndexUser.getIndexes() 출력</summary>
<div markdown="1">

```json
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"age" : 1,
			"name" : 1
		},
		"name" : "age_1_name_1"
	},
	{
		"v" : 2,
		"key" : {
			"age" : 1,
			"email.id" : 1
		},
		"name" : "age_1_email.id_1"
	}
]
```  

</div>
</details>



---
## Reference
[Spring Data MongoDB – Indexes, Annotations and Converters](https://www.baeldung.com/spring-data-mongodb-index-annotations-converter)  
[MongoDB Indexes with Spring Data](https://lankydan.dev/2017/06/07/mongodb-indexes-with-spring-data)  