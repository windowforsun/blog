--- 
layout: single
classes: wide
title: "[Spring 실습] JPA QueryDSL 설정 및 기본 사용"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 에서 타입에 안전하게 쿼리를 생성 관리할 수 있는 QueryDSL 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring JPA
    - Hibernate
    - QueryDSL
toc: true
use_math: true
---  

## QueryDSL
`QueryDSL` 은 `Hibernate` 에서 쿼리를 실행하는 `HQL`, `Criteria` 방식중 
`HQL(Hibernate Query Language)` 쿼리를 `Type-Safe` 하게 쿼리를 생성하고 관리할 수 있는 프레임워크이다. 
마치 `QueryBuilder` 처럼 코드 기반으로 원하는 쿼리를 생성할 수 있다고 생각하면 간단하다. 

[QueryDSL](http://querydsl.com/static/querydsl/5.0.0/reference/html_single/#preface) 
공식 레퍼런스에서는 아래 몇가지 특징으로 소개하고 있다. 

- `IDE` 의 코드 자동 완성 기능 사용 가능
- 문법적으로 잘못된 쿼리를 허용하지 않음
- 도메인 타입과 프로퍼티를 안전하게 참조 가능
- 도메인 타임의 리펙토링의 용의성

### QueryDSL 의 필요성
`QueryDSL` 를 사용하지 않고 `Spring` 에서 쿼리를 수행할 할 수 있는 방법 중 몇가지만 나열하면 아래와 같은 것들이 있을 것이다. 

- `MyBatis`
- `Spring JPA Repository(QueryMethod)`
- `Spring JPA JPQL`

먼저 `MyBatis` 를 사용하는 경우에 쿼리를 작성하면 `XML` 파일에 아래와 같이 작성될 것이다.  

```xml
SELECT
    *
FROM
    user
WHERE
    id = #{id}
```  

여기서 조건절 `id` 가 어떠한 타입을 가지고 있는지가 명확하지 않는 다는 점이 있다. 
만약 잘못된 타입이 설정되더라도, 빌드타입에서는 탐지가 되지 않고 실제 쿼리가 수행되는 런타임에서야 탐지가 가능하다.  

하지만 `QueryDSL` 을 사용하게 되면, 실제 테이블과 매핑되는 엔티티 클래스를 바탕으로 `QClass` 라는 것을 생성하기 때문에, 
이러한 타입에 대한 문제를 빌드 타임에 탐지가 가능하다.  

```java
queryFactory
    .selectFrom(qUser)
    .where(qUser.id.eq(myId))
    .fetchOne();
```  

다음으로 `Spring JPA` 를 사용하는 경우를 보자. 
일반적으로 `Repository` 클래스의 `QueryMethod` 를 사용하면 아래와 같이 사용할 수 있다.  

```java
@Repository
public interface UserRepository extends JpaRepository<User, String> {
    User findByName(String name);
}
```  

위와 같은 `QueryMethod` 의 경우 간단한 쿼리만 사용한다면 생산성을 높일 수 있지만, 
복잡한 쿼리가 필요한 경우에는 아래 처럼 직접 쿼리를 작성하는 `JPQL` 을 사용하게 된다.  

```java

@Repository
public interface UserRepository extends JpaRepository<User, String> {
    @Query(value = "SELECT id, name FROM user WHERE address_id = (SELECT id FROM address WHERE addressMain = :address ORDER BY update_time DESC)", nativeQuery = true)
    User findbyAddress(String address);
}
```  

복잡한 쿼리를 사용할 수는 있지만, 
위와 같이 문자열 상으로 작성된 쿼리이기 때문에 가독성이나 휴먼에러 발생 여지가 높다고 할 수 있다.  

`QueryDSL` 을 사용하게 된다면 위와 같은 문제를 앞서 언급한 `Type-Safe` 한 방식에서 `Builder` 패턴을 통해 복잡한 쿼리를 더 효율적으로 작성 할 수 있다.  

```java
queryFactory
    .selectFrom(qUser)
    .where(qUser.addressId.eq(
            JPAExpressions.selectFrom(qAddress)
                .where(qAddress.addressMain.eq(myAddress))
                .orderBy(qAddress.updateTime.desc())
    ))
    .fetch()
```  

### QueryDSL 환경 구성 및 설정

- 환경
  - `Spring Boot 2.5`
  - `Gradle 6.8`
  - `Java 11`
	

- `build.gradle`

```groovy
plugins {
	id 'org.springframework.boot' version '2.5.0'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.windowforsun.querydslexam'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	compileOnly "org.projectlombok:lombok"
	testCompileOnly "org.projectlombok:lombok"
	annotationProcessor "org.projectlombok:lombok"
	testAnnotationProcessor "org.projectlombok:lombok"
	implementation "com.querydsl:querydsl-jpa"
	runtimeOnly 'com.h2database:h2'

	annotationProcessor(
		// java.lang.NoClassDefFoundError(javax.annotation.Entity) 발생 대응
		"jakarta.persistence:jakarta.persistence-api",
		// java.lang.NoClassDefFoundError(javax.annotation.Generated) 발생 대응
		"jakarta.annotation:jakarta.annotation-api",
		// Querydsl JPAAnnotationProcessor 사용 지정
		"com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
	)
}

test {
	useJUnitPlatform()
}
```  

- `application.yaml`

```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true # 쿼리 출력
        format_sql: true # 쿼리 이쁘게 출력
```  

- `QueryDSLConfig`

```java
@Configuration
public class QueryDslConfig {
    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(this.entityManager);
    }
}
```  

- `Entity`

```java
@Getter
@Setter
@Entity
@Table(name = "user")
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class User {
    @Id
    @Column(name = "user_id")
    private String userId;
    private String name;
    private int age;
    private String groupId;
}
```  

- `Repository`

```java
@Repository
public interface UserRepository extends JpaRepository<User, String> {
}
```  

- `QClass` 생성
  - `QClass` 생성은 앞선 구성들이 모두 완료된 상태에서 빌드를 수행해주면 된다. 
  - 빌드를 하게 되면 아래처럼 `build/genterated/sources/annotationProcessor` 하위에 `@Entity` 가 붙은 클래스와 매핑되는 `QClass` 가 생성된다.
	
    ![그림 1]({{site.baseurl}}/img/spring/practice-jpa-querydsl-intro-1.png)


### QueryDSL 기본 사용법

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class QueryDslTest {
    @Autowired
    private JPAQueryFactory jpaQueryFactory;
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private GroupRepository groupRepository;
    private final QUser qUser = QUser.user;

    @BeforeEach
    public void setUp() {
        this.userRepository.deleteAll();
        this.userRepository.save(User.builder()
                .userId("id-1")
                .name("aa")
                .age(20)
                .groupId("study")
                .build());
        this.userRepository.save(User.builder()
                .userId("id-2")
                .name("ab")
                .age(30)
                .groupId("study")
                .build());
        this.userRepository.save(User.builder()
                .userId("id-3")
                .name("ab")
                .age(25)
                .groupId("trip")
                .build());
    }

    @Test
    public void selectUser_all() {
        // when
        List<User> actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .fetch();

        // then
        assertThat(actual, hasSize(3));
        assertThat(actual.get(0).getUserId(), is("id-1"));
    }

    @Test
    public void selectUserSomeField_all() {
        // when
        List<Tuple> actual = this.jpaQueryFactory
                .select(this.qUser.name, this.qUser.age)
                .from(this.qUser)
                .fetch();

        // then
        assertThat(actual, hasSize(3));
        assertThat(actual.get(0).get(this.qUser.name), is("aa"));
        assertThat(actual.get(0).get(this.qUser.age), is(20));
        assertThat(actual.get(1).get(this.qUser.name), is("ab"));
        assertThat(actual.get(1).get(this.qUser.age), is(30));
    }

    @Test
    public void selectUser_first() {
        // when
        User actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .fetchFirst();

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getUserId(), is("id-1"));
    }

    @Test
    public void selectUser_all_where_eq() {
        // when
        List<User> actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .where(this.qUser.name.eq("ab"))
                .fetch();

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, everyItem(hasProperty("name", is("ab"))));
    }

    @Test
    public void selectUser_where_null() {
        // when
        List<User> actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .where(null, this.qUser.name.eq("aa"))
                .fetch();

        // then
        assertThat(actual, hasSize(1));
        assertThat(actual.get(0).getUserId(), is("id-1"));
    }

    @Test
    public void selectUser_one_where_eq_and_eq() {
        // when
        User actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .where(this.qUser.name.eq("ab"), this.qUser.age.eq(30))
                .fetchOne();

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getUserId(), is("id-2"));
        assertThat(actual.getName(), is("ab"));
    }

    @Test
    public void selectUser_all_where_eq_or_eq() {
        // when
        List<User> actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .where(this.qUser.name.eq("ab").or(this.qUser.age.eq(30)))
                .fetch();

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("ab"))));
        assertThat(actual, hasItem(hasProperty("age", is(30))));
    }

    @Test
    public void selectUser_all_order_name_desc_age_asc() {
        // when
        List<User> actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .orderBy(this.qUser.name.desc(), this.qUser.age.asc())
                .fetch();

        // then
        assertThat(actual, hasSize(3));
        assertThat(actual.get(0), allOf(hasProperty("name", is("ab")), hasProperty("age", is(25))));
        assertThat(actual.get(1), allOf(hasProperty("name", is("ab")), hasProperty("age", is(30))));
        assertThat(actual.get(2), allOf(hasProperty("name", is("aa")), hasProperty("age", is(20))));
    }

    @Test
    public void selectUserName_all_group() {
        // when
        List<String> actual = this.jpaQueryFactory
                .select(this.qUser.name)
                .from(this.qUser)
                .groupBy(this.qUser.name)
                .fetch();

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual.get(0), is("aa"));
        assertThat(actual.get(1), is("ab"));
    }

    @Test
    @Transactional
    public void deleteUser_all() {
        // when
        long actual = this.jpaQueryFactory
                .delete(this.qUser)
                .execute();

        // then
        assertThat(actual, is(3L));
    }

    @Test
    @Transactional
    public void deleteUser_where() {
        // when
        long actual = this.jpaQueryFactory
                .delete(this.qUser)
                .where(this.qUser.name.eq("ab"))
                .execute();

        // then
        assertThat(actual, is(2L));
    }

    @Test
    @Transactional
    public void updateUser_where() {
        // when
        long actual = this.jpaQueryFactory
                .update(this.qUser)
                .where(this.qUser.name.eq("ab"))
                .set(this.qUser.age, 40)
                .execute();

        // then
        assertThat(actual, is(2L));
    }

    @Test
    public void selectUser_subquery_maxUserAge() {
        // when
        User actual = this.jpaQueryFactory
                .selectFrom(this.qUser)
                .where(this.qUser.age.eq(
                        JPAExpressions.select(this.qUser.age.max()).from(this.qUser)
                ))
                .fetchOne();

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getUserId(), is("id-2"));
        assertThat(actual.getAge(), is(30));
    }
}
```  


---  
## Reference
[QueryDSL JPA](http://querydsl.com/static/querydsl/5.0.0/reference/html_single/#jpa_integration)  