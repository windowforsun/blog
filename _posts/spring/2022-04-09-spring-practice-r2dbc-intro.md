--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Data R2DBC 설정과 기본 사용방법"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 Reactive Stream 데이터소스 구현체인 R2DBC와 Spring Data R2DBC 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Data R2DBc
    - Reactive Stream
    - Spring Webflux
    - R2DBC
    - Transaction
    - H2
    - MySQL
    - Reactor
    - DB
toc: true
use_math: true
---  

## R2DBC
[R2DBC](https://r2dbc.io/) 
는 `Reactive Stream`(`Non-Blocking`) 으로 구현된 `DataSource` 라고 할 수 있고, 
이는 `Blocking` 방식인 `JDBC` 의 대체제라고도 할 수 있다. 
`Spring Webflux` 를 사용한다면 `JDBC` 보다는 `R2DBC` 를 통해 완벽한 `Reactive Stream` 를 구현 할 수 있다.  

`Spring Webflux` 로 인해 `Java` 및 `Spring` 진영에서도 `Reactive Stream` 의 활용과
실제 서비스에서 사용되는 빈도가 높아지고 있다. 
`Reactive Stream` 기반인 `Spring Webflux` 를 100% 활용하기 위해서는 
애플리케이션에서 요청을 받고 응답하는 모든 흐름이 `Reactive` 하게 구현돼야 한다. 
흐름 구간에서 하나의 `blocking` 만 발생하더라도 `Spring Webflux` 의 활용도가 크게 떨어진다.  

웹 애플리케이션의 잘 알려진 구조는 `Application + DB` 인
애플리케이션이 요청 처리를 위해서 `DB` 를 조회해 그 결과를 응답해주는 구조일 것이다.  

`Spring MVC` 를 사용할 때는 `Spring MVC + JDBC` 를 조합을 사용하게 된다. 
몇전 전까지만 하더라도 `Non-Blocking` 데이터소스가 없어 
`Spring Webflux` 에도 `Blocking` 방식인 `JDBC` 조합을 많이 사용했었다. 
`JDBC` 관련 동작은 모두 `Schedulers` 를 사용해서 `Reactive Stream` 상에 `Blocking` 을 발생하지 않도록 회피 하는 방법이 많이 사용됐다. 
`R2DBC` 를 통해 `Non-Blcoing` 방식인 데이터소스 사용방법에 대해 알아본다.   


## Spring Data R2DBC
[Spring Data R2DBC](https://github.com/spring-projects/spring-data-r2dbc)  
는 `Spring Data` 의 `R2DBC` 버전으로 해당 의존성을 사용하면 `Spring Boot` 프로젝트에서 
보편적인 방법으로 `R2DBC` 를 사용할 수 있다.  

`Spring Data R2DBC` 의 특징을 간단하게 나열하면 아래와 같다. 

- `Spring JPA` 와는 다른 구현체이다. 
- `ORM` 이 아니면서 `Spring Data JPA` 에서 사용 가능한 모든 기능을 사용할 수는 없다. 
- `QueryDSL` 이 현재 공식적으로 지원은 되지 않는다. 
  - [외부 라이브러리](https://github.com/infobip/infobip-spring-data-querydsl) 를 사용해서 연동하는 방법은 있는 것 같다.
- `Spring Webflux` 와 조합이 매우 좋다. 

### 의존성
테스트에 사용한 `Spring Boot` 버전은 `Spring Boot 2.5.0` 을 사용했다. 
`Spring Data R2DBC` 사용을 위해서는 아래와 같은 의존성 추가가 필요하다.  

```groovy
implementation "org.springframework.boot:spring-boot-starter-data-r2dbc"

// H2 를 사용하는 경우
runtimeOnly "com.h2database:h2"
runtimeOnly "io.r2dbc:r2dbc-h2"

// MySQL 을 사용하는 경우
runtimeOnly "mysql:mysql-connector-java"
runtimeOnly "dev.miku:r2dbc-mysql"



// 공통
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'io.projectreactor:reactor-test'
testImplementation 'io.projectreactor.tools:blockhound:1.0.6.RELEASE'
compileOnly "org.projectlombok:lombok"
testCompileOnly "org.projectlombok:lombok"
annotationProcessor "org.projectlombok:lombok"
testAnnotationProcessor "org.projectlombok:lombok"

// TestContainers
testImplementation "org.testcontainers:mysql:1.16.3"
testImplementation "org.testcontainers:r2dbc:1.16.3"
testImplementation "org.testcontainers:junit-jupiter:1.16.3"
```  

### 설정
`Spring Data R2DBC` 를 설정하는 방법으로는 `application.yaml` 사용하는 방법과 
직접 `Java Config` 를 사용해서 설정하는 방법이 있는데 `application.yaml` 을 사용해서 설정하는 방법에 대해 알아본다. 

먼저 `application.yaml` 을 사용하는 방법은 아래와 같다. 
`H2` 는 아래 설정으로 가능하다. 

```yaml
spring:
    r2dbc:
        url: r2dbc:h2:mem:///test
        username: sa

# show query log
logging:
    level:
        io.r2dbc.h2: DEBUG
        org.springframework.r2dbc: DEBUG
```  

`MySQL` 을 사용한다면 아래와 같이 설정할 수 있다.  

```yaml
spring:
    r2dbc:
        url: r2dbc:mysql://<host>:<port>/<dbname>?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Seoul
        username: root
        password: test

# show query log
logging:
  level:
    dev.miku.r2dbc.mysql.client.ReactorNettyClient: DEBUG
    org.springframework.r2dbc: DEBUG
```  

`R2DBC` 커넥션 풀 관련 커스텀한 설정이 필요하다면 아래 설정으로 가능하다.  

```yaml
spring:
  r2dbc:
    pool:
      enabled: true # r2dbc-pool 의존성이 있다면 자동 활성화
      max-size: 150 # default 10
      initial-size: 50 # default 10

```  

### 엔티티 
`Spring Data R2DBC` 에서 엔티티를 생성하는 방법은 `Spring Data JPA` 와 거의 유사하다. 
하지만 약간에 차이가 있고, 연관관계 매핑과 같은 부분은 아직 공식적으로 지원하지 않는 것으로 보인다.  

테스트를 위해 아래 `Member`, `Team` 엔티티를 생성해주었다.  

```java
@Getter
@Setter
@Builder
@ToString
@Table("member")
@NoArgsConstructor
@AllArgsConstructor
public class Member {
    @Id
    private Long id;
    private String name;
    private int age;
    private Long teamId;
}

@Getter
@Setter
@Builder
@ToString
@Table("team")
@NoArgsConstructor
@AllArgsConstructor
public class Team {
    @Id
    private Long id;
    private String name;
}
```  

`@Table` 어노테이션을 사용해서 해당 엔티티가 매핑될 테이블 이름을 명시해 주었다. 
그리고 각 엔티티에는 테이블에서 `PK` 역할을 하는 `@Id` 어노테이션 설정이 필요하다. 
또한 `@Column` 어노테이션을 사용해서 컬럼명 매핑을 직접 지정할 수도 있다.  

그리고 `Spring Data R2DBC` 는 `Spring Data JPA` 에서 사용할 수 있었던 `ddl-auto` 프로퍼티 옵션사용이 불가하다. 
이는 즉 엔티티와 매핑된 테이블이 데이터베이스에 존재하지 않을 때 자동으로 생성해주지 못한다는 의미이다. 
하지만 `Spring Boot` 에서 제공하는 데이터베이스 초기화 기능을 사용하면 어느정도 해결을 할 수 있다.  

`src/main/resources` 디렉토리 하위에 `schema.sql` 파일을 하나 만들고 아래와 같은 테이블 생성에 필요한 쿼리를 작성해 준다.  

```sql
DROP TABLE IF EXISTS member;
CREATE TABLE member (
    id INT(20) AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(20) NOT NULL,
    age INT(3) NOT NULL,
    team_id INT(20)
);

DROP TABLE IF EXISTS team;
CREATE TABLE team (
    id INT(20) AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(20) NOT NULL
);

INSERT INTO team(id, name) VALUES(1, 'team-1');
INSERT INTO team(id, name) VALUES(2, 'team-2');
INSERT INTO team(id, name) VALUES(3, 'team-3');


INSERT INTO member(name, age, team_id) VALUES('a', 20, 1);
INSERT INTO member(name, age, team_id) VALUES('b', 21, 2);
INSERT INTO member(name, age, team_id) VALUES('c', 22, 3);
INSERT INTO member(name, age, team_id) VALUES('d', 23, 1);
INSERT INTO member(name, age, team_id) VALUES('e', 24, 2);
INSERT INTO member(name, age, team_id) VALUES('a', 24, 3);
```  

`Spring Boot` 가 기동되면서 연결된 데이터베이스를 타겟으로 `schema.sql` 파일 내용을 실행한다.  

### R2dbcEntityTemplate
`R2dbcEntityTemplate` 은 `Spring Data R2DBC` 의 진입점으로 쿼리와 엔티티간의 인터페이스 역할을 수행한다. 
`R2dbcEntityTemplate` 의 실제 쿼리는 `DatabaseClient` 를 통해 수행되게 되고 `DatabaseClient` 는 `ConnectionFactory` 를 가지게 된다. 
그 관계를 표현하면 아래와 같다.  

```java
DatabaseClient dbClient = DatabaseClient.create(connectionFactory);
R2dbcEntityTemplate template1 = new R2dbcEntityTemplate(dbClient, dialect);

// 혹은 connectionFactory 를 바로 사용해서도 R2dbcEntityTemplate 생성가능
R2dbcEntityTemplate template2 = new R2dbcEntityTenplate(connectionFactory); 
```  

`R2dbcEntityTemplate` 을 사용하면 `QueryBuilder` 처럼 쿼리를 사용하거나, 
`DatabaseClient` 에 직접 쿼리를 작성해서 사용할 수도 있다. 

```java
@DataR2dbcTest(
        properties = {
                "spring.r2dbc.url=r2dbc:h2:mem:///test",
                "spring.r2dbc.username=sa"
        }
)
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class R2dbcEntityTemplateTest {
    @Autowired
    private R2dbcEntityTemplate r2dbcEntityTemplate;
    @Autowired
    private MappingR2dbcConverter mappingR2dbcConverter;

    @Test
    public void select_member_all() {
        Flux<Member> flux = this.r2dbcEntityTemplate.select(Member.class).all();

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(6));
                    assertThat(members, everyItem(instanceOf(Member.class)));
                })
                .verifyComplete();
    }

    @Test
    public void select_member_by_name() {
        Flux<Member> flux = this.r2dbcEntityTemplate.select(
                Query.query(Criteria.where("name").is("a")),
                Member.class
        );

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("name", is("a"))));
                })
                .verifyComplete();
    }

    @Test
    public void select_member_by_age_between() {
        Flux<Member> flux = this.r2dbcEntityTemplate.select(
                Query.query(Criteria.where("age").between(21, 23)),
                Member.class
        );

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(3));
                    assertThat(members, everyItem(hasProperty("age", oneOf(21, 22, 23))));
                })
                .verifyComplete();
    }

    @Test
    public void select_member_by_age_max() {
        Flux<Member> flux = this.r2dbcEntityTemplate
                .getDatabaseClient()
                .sql("SELECT * FROM member WHERE age = (SELECT MAX(age) FROM member)")
                .map((row, rowMetadata) -> this.mappingR2dbcConverter.read(Member.class, row, rowMetadata))
                .all();

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("age", is(24))));
                })
                .verifyComplete();
    }

    @Test
    public void select_member_by_name_ordeBy_age_desc() {
        Flux<Member> flux = this.r2dbcEntityTemplate
                .getDatabaseClient()
                .sql("SELECT * FROM member WHERE name = :name ORDER BY age DESC")
                .bind("name", "a")
                .map((row, rowMetadata) -> this.mappingR2dbcConverter.read(Member.class, row, rowMetadata))
                .all();

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("name", is("a"))));

                    List<Member> memberArrayList = new ArrayList<>(members);
                    assertThat(memberArrayList.get(0).getAge(), is(24));
                    assertThat(memberArrayList.get(1).getAge(), is(20));
                })
                .verifyComplete();
    }

    @Test
    public void delete_member_by_name() {
        Mono<Integer> deleteMono = this.r2dbcEntityTemplate.delete(
                Query.query(Criteria.where("name").is("a")),
                Member.class
        );

        StepVerifier
                .create(deleteMono)
                .expectNext(2)
                .verifyComplete();
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Table("team_member")
    public static class TeamMember {
        @Column("memberId")
        private Long memberId;
        @Column("memberName")
        private String memberName;
        @Column("memberAGe")
        private Integer memberAge;
        @Column("teamId")
        private Long teamId;
        @Column("teamName")
        private String teamName;
    }

    @Test
    public void select_member_join_team_all() {
        Flux<TeamMember> flux = this.r2dbcEntityTemplate
                .getDatabaseClient()
                .sql("SELECT m.id as memberId, m.name as memberName, m.age as memberAge, t.id as teamId, t.name as teamName FROM member m inner join team t on m.team_id = t.id")
                .map((row, rowMetadata) -> this.mappingR2dbcConverter.read(TeamMember.class, row, rowMetadata))
                .all();

        StepVerifier
                .create(flux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(teamMember -> true)
                .consumeRecordedWith(teamMembers -> {
                    assertThat(teamMembers, hasSize(6));
                    assertThat(teamMembers, everyItem(instanceOf(TeamMember.class)));
                    assertThat(teamMembers, everyItem(hasProperty("teamId", oneOf(1L, 2L, 3L))));
                    assertThat(teamMembers, everyItem(hasProperty("memberName", oneOf("a", "b", "c", "d", "e"))));
                })
                .verifyComplete();
    }
}
```  

`select_member_join_team_all` 테스트에서는 현재 `member`, `team` 테이블을 조인한 결과를 별도를 생성한 `POJO` 인 `TeamMember` 에 매핑해서 결과를 가져오고 있다.  
이러한 조인결과 매핑을 위해서는 `POJO` 에 `@Column` 어노테이션을 사용해서 필드명 지정이 필요하다.  

### Repository
`Spring Data R2DBC` 도 `Spring Data JPA` 처럼 `Repository` 를 통핸 `QueryMethod` 를 사용할 수 있다.  

```java
@Repository
public interface MemberRepository extends ReactiveCrudRepository<Member, Long> {
    Flux<Member> findByName(String name);

    Flux<Member> findByTeamId(Long id);

    Flux<Member> findByAge(int age);

    Flux<Member> findByAgeBetween(int min, int max);

    @Query("SELECT * FROM member WHERE age = (SELECT MAX(age) FROM member)")
    Flux<Member> findAgeMax();

    @Query("SELECT * FROM member WHERE name = :name ORDER BY age DESC")
    Flux<Member> findByNameOderByAgeDesc(String name);

    Mono<Void> deleteByName(String name);

    Mono<Void> deleteByTeamId(Long id);

    @Query("DELETE FROM member WHERE name = :name LIMIT 1")
    Mono<Void> deleteOneByName(String name);
}
```  

```java
@Repository
public interface TeamRepository extends ReactiveCrudRepository<Team, Long> {
}
```  

`ReactiveCrudRepository` 인터페이스를 상속받아 기존 `QueryMethod` 와 비슷하게 원하는 메소드를 생성해서 사용할 수 있다. 
또한 복잡한 쿼리의 경우 `@Query` 어노테이션을 사용해서 직접 쿼리로 작성해서 사용할 수도 있다.  

### Repository Test
앞서 작성한 `Repository` 의 `QueryMethod` 에 대한 테스트 클래스는 `H2` 를 사용하는 경우, `MySQL` 을 사용하는 경우로 2가지로 
진행해 보았다.  

먼저 `H2` 를 사용하는 경우는 아래와 같다.  

```java
@DataR2dbcTest(
        properties = {
                "spring.r2dbc.url=r2dbc:h2:mem:///test",
                "spring.r2dbc.username=sa"
        }
)
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class R2dbcH2Test {
    @Autowired
    private MemberRepository memberRepository;

    @TestConfiguration
    public static class TestConfig {
        @Bean
        public ReactiveTransactionManager transactionManager(ConnectionFactory connectionFactory) {
            return new R2dbcTransactionManager(connectionFactory);
        }
    }

    @Test
    public void save() {
        Member member = Member.builder()
                .name("a")
                .build();

        StepVerifier
                .create(this.memberRepository.save(member))
                .consumeNextWith(member1 -> {
                    assertThat(member1, notNullValue());
                    assertThat(member1.getId(), greaterThan(0L));
                    assertThat(member1.getName(), is("a"));

                })
                .verifyComplete();

    }

    @Test
    public void findAll() {
        StepVerifier
                .create(this.memberRepository.findAll())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(6));
                    assertThat(members, everyItem(hasProperty("name", not(emptyOrNullString()))));
                    assertThat(members, everyItem(hasProperty("age", greaterThan(0))));
                })
                .verifyComplete();
    }

    @Test
    public void findByName() {
        StepVerifier
                .create(this.memberRepository.findByName("d"))
                .recordWith(ArrayList::new)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                    assertThat(members, hasItem(hasProperty("name", is("d"))));
                })
                .verifyComplete();
    }

    @Test
    public void findByTeamId() {
        StepVerifier
                .create(this.memberRepository.findByTeamId(2L))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("teamId", is(2L))));
                })
                .verifyComplete();
    }

    @Test
    public void findByAge() {
        StepVerifier
                .create(this.memberRepository.findByAge(20))
                .recordWith(ArrayList::new)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                    assertThat(members, hasItem(hasProperty("age", is(20))));
                })
                .verifyComplete();
    }

    @Test
    public void findByAgeBetween() {
        StepVerifier
                .create(this.memberRepository.findByAgeBetween(21, 23))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(3));
                    assertThat(members, hasItem(hasProperty("age", is(21))));
                    assertThat(members, hasItem(hasProperty("age", is(22))));
                    assertThat(members, hasItem(hasProperty("age", is(23))));
                })
                .verifyComplete();
    }

    @Test
    public void findAgeMax() {
        StepVerifier
                .create(this.memberRepository.findAgeMax())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("age", is(24))));
                })
                .verifyComplete();
    }


    @Test
    public void findByNameOrderByAgeDesc() {
        StepVerifier
                .create(this.memberRepository.findByNameOderByAgeDesc("a"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("name", is("a"))));

                    List<Member> memberArrayList = new ArrayList<>(members);
                    assertThat(memberArrayList.get(0).getAge(), is(24));
                    assertThat(memberArrayList.get(1).getAge(), is(20));
                })
                .verifyComplete();
    }

    @Test
    public void deleteByTeamId() {
        StepVerifier
                .create(this.memberRepository.deleteByTeamId(1L))
                .verifyComplete();

        StepVerifier
                .create(this.memberRepository.findAll())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(greaterThan(1)));
                    assertThat(members, everyItem(hasProperty("teamId", not(1L))));
                })
                .verifyComplete();
    }

    @Test
    public void deleteOneByName() {
        StepVerifier
                .create(this.memberRepository.deleteOneByName("a"))
                .verifyComplete();

        StepVerifier
                .create(this.memberRepository.findByName("a"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                })
                .verifyComplete();
    }
}
```  

다음은 `MySQL` 을 사용했을 때의 테스트이다. 
`MySQL` 은 환경구성을 위해 `TestContainers` 를 사용했다. 

```java
@DataR2dbcTest
@ExtendWith(SpringExtension.class)
@Testcontainers
public class R2dbcMySqlTest {
    @Autowired
    private MemberRepository memberRepository;
    @Container
    private static final MySQLContainer MY_SQL = new MySQLContainer(DockerImageName.parse("mysql:8"));

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () -> String.format("r2dbc:tc:mysql:///%s?TC_IMAGE_TAG=8", MY_SQL.getDatabaseName()));
        registry.add("spring.r2dbc.username", () -> MY_SQL.getUsername());
        registry.add("spring.r2dbc.password", () -> MY_SQL.getPassword());
    }

    @Test
    public void save() {
        Member member = Member.builder()
                .name("a")
                .build();

        StepVerifier
                .create(this.memberRepository.save(member))
                .consumeNextWith(member1 -> {
                    assertThat(member1, notNullValue());
                    assertThat(member1.getId(), greaterThan(0L));
                    assertThat(member1.getName(), is("a"));

                })
                .verifyComplete();

    }

    @Test
    public void findAll() {
        StepVerifier
                .create(this.memberRepository.findAll())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(6));
                    assertThat(members, everyItem(hasProperty("name", not(emptyOrNullString()))));
                    assertThat(members, everyItem(hasProperty("age", greaterThan(0))));
                })
                .verifyComplete();
    }

    @Test
    public void findByName() {
        StepVerifier
                .create(this.memberRepository.findByName("d"))
                .recordWith(ArrayList::new)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                    assertThat(members, hasItem(hasProperty("name", is("d"))));
                })
                .verifyComplete();
    }

    @Test
    public void findByAge() {
        StepVerifier
                .create(this.memberRepository.findByAge(20))
                .recordWith(ArrayList::new)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                    assertThat(members, hasItem(hasProperty("age", is(20))));
                })
                .verifyComplete();
    }

    @Test
    public void findByAgeBetween() {
        StepVerifier
                .create(this.memberRepository.findByAgeBetween(21, 23))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(3));
                    assertThat(members, hasItem(hasProperty("age", is(21))));
                    assertThat(members, hasItem(hasProperty("age", is(22))));
                    assertThat(members, hasItem(hasProperty("age", is(23))));
                })
                .verifyComplete();
    }

    @Test
    public void findAgeMax() {
        StepVerifier
                .create(this.memberRepository.findAgeMax())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("age", is(24))));
                })
                .verifyComplete();
    }


    @Test
    public void findByNameOrderByAgeDesc() throws Exception {
        StepVerifier
                .create(this.memberRepository.findByNameOderByAgeDesc("a"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(2));
                    assertThat(members, everyItem(hasProperty("name", is("a"))));

                    List<Member> memberArrayList = new ArrayList<>(members);
                    assertThat(memberArrayList.get(0).getAge(), is(24));
                    assertThat(memberArrayList.get(1).getAge(), is(20));
                })
                .verifyComplete();
    }

    @Test
    public void deleteByTeamId() {
        StepVerifier
                .create(this.memberRepository.deleteByTeamId(1L))
                .verifyComplete();

        StepVerifier
                .create(this.memberRepository.findAll())
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(greaterThan(1)));
                    assertThat(members, everyItem(hasProperty("teamId", not(1L))));
                })
                .verifyComplete();
    }

    @Test
    public void deleteOneByName() {
        StepVerifier
                .create(this.memberRepository.deleteOneByName("a"))
                .verifyComplete();

        StepVerifier
                .create(this.memberRepository.findByName("a"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(member -> true)
                .consumeRecordedWith(members -> {
                    assertThat(members, hasSize(1));
                })
                .verifyComplete();
    }
}
```  


### Transaction
`Spring Data R2DBC` 에서도 트랜잭션은 `@Transactional` 어노테이션을 사용한 선언적 트랜잭션을 메소드 단위로 사용할 수 있다.  


```java
@SpringBootTest(
        properties = {
                "spring.r2dbc.url=r2dbc:h2:mem:///test",
                "spring.r2dbc.username=sa"
        }
)
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
@EnableTransactionManagement
@Slf4j
public class TransactionTest {
    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private TeamRepository teamRepository;



    @Transactional(transactionManager = "transactionManager")
    public Mono<Boolean> deleteTeamById(Long id) {
        return this.memberRepository.deleteByTeamId(id)
                .then(this.teamRepository.deleteById(id))
                .thenReturn(true)
                .onErrorResume(throwable -> {
                    log.error("error", throwable);
                    return Mono.just(false);
                });

    }

    @Test
    public void deleteTeamById() {
        // teamId 1번 존재
        StepVerifier
                .create(this.memberRepository.findByTeamId(1L))
                .expectNextCount(2)
                .verifyComplete();

        StepVerifier
                .create(this.teamRepository.findById(1L))
                .expectNextCount(1)
                .verifyComplete();

        // teamId 1번 삭제
        StepVerifier
                .create(this.deleteTeamById(1L))
                .expectNext(true)
                .verifyComplete();

        // teamId 1번 존재하지 않음
        StepVerifier
                .create(this.memberRepository.findByTeamId(1L))
                .verifyComplete();

        StepVerifier
                .create(this.teamRepository.findById(1L))
                .verifyComplete();
    }

}
```  



---  
## Reference
[Spring Data R2DBC Refrence](https://docs.spring.io/spring-data/r2dbc/docs/current/reference/html/#preface)  
[A Quick Look at R2DBC with Spring Data](https://www.baeldung.com/spring-data-r2dbc)  
[Handle R2dbc in Spring Projects](https://www.sipios.com/blog-tech/handle-the-new-r2dbc-specification-in-java)  
[Spring Data R2DBC - Transactions](https://medium.com/geekculture/spring-data-r2dbc-transactions-cd5e064d59a8)  
Spring Data R2DBC Transaction(https://www.vinsguru.com/spring-data-r2dbc-transaction/)  