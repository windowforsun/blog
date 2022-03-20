--- 
layout: single
classes: wide
title: "[Spring 실습] JPA QueryDSL Repository 사용하기(with join)"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 과 QueryDSL 을 사용할 떄 사용할 수 있는 Repository 와 조인에 대해서 알아보자'
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
    - Repository
toc: true
use_math: true
---  

## Spring JPA QueryDSL Repository
`Spring JPA` 를 사용하면 `Repository` 에 `QueryMethod` 를 사용해서 쿼리를 간편하게 수행할 수 있다. 
[JPA QueryDSL 설정 및 기본 사용]({{site.baseurl}}{% link _posts/spring/2022-03-19-spring-practice-jpa-querydsl-intro.md.md %})
에서 설명했던 것 처럼 `Spring JPA Repository` 의 `QueryMethod` 의 경우 복잡한 쿼리에 대한 제약사항이 있기 때문에, 
문자열 기반인 `JPQL` 을 불가피하게 사용할 수 밖에 없다.  

하지만 `QueryDSL` 을 사용한다면 `JPQL` 대신 `QueryDSL` 의 장점을 살리며 쿼리를 간편하게 작성, 관리가 가능하다.  

`Spring JPA` 에는 연관관계라는 엔티티간의 매핑 규칙이 존재한다. 
`QueryDSL` 최신 버전부터는 연관관계 매핑을 사용하지 않더라도 엔티티간 조인이 가능하므로, 
연관관계 매핑을 사용하지 않는 경우, 연관관계 매핑을 사용하는 경우로 나눠서 사용방법에 대해 알아보고자 한다.  

### 환경 구성

- `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '2.5.0'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.windowforsun.querydslmapping'
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

- `QuerydslConfig`

```java
@Configuration
public class QuerydslConfig {
    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(this.entityManager);
    }
}
```  

### 연관관계 매핑 X

테스트에 사용할 엔티티는 `Member`, `Team` 가 있다. 
`Member`, `Team` 은 `N:1` 관계로 `team_id` 를 통해 연결된다. 
`Team` 에 해당하는 `Member` 를 조회하는 등의 동작은 모두 명시적인 조인을 수행해줘야 가능하다.  

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_nomapping")
@Table(name = "member_nomapping")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private Long teamId;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_nomapping")
@Table(name = "team_nomapping")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
}
```  

다음으로는 `Spring JPA Repository` 클래스이다. 
`Member`, `Team` 과 매핑되며 `QueryMethod` 를 사용해서 간편하게 간단한 쿼리를 수행할 수 있다.  

```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
    Member findFirstByName(String name);

    List<Member> findAllByName(String name);
}

@Repository
public interface TeamRepository extends JpaRepository<Team, Long> {
    Team findByName(String name);
}
```  

`QueryDSL` 을 사용해서 쿼리를 수행하는 `RepositorySupport` 클래스이다. 
`Spring JPA` 에서 수행할 수있는 쿼리 뿐만 아니라, 조인, 서브쿼리 등 복잡한 쿼리를 구성할 수 있다.  

```java
import static com.windowforsun.querydslmapping.nomapping.QMember.*;
import static com.windowforsun.querydslmapping.nomapping.QTeam.*;

@Repository
public class MemberRepositorySupport extends QuerydslRepositorySupport {
    private final JPAQueryFactory jpaQueryFactory;

    public MemberRepositorySupport(JPAQueryFactory jpaQueryFactory) {
        super(Member.class);
        this.jpaQueryFactory = jpaQueryFactory;
    }

    public List<Member> findAllByName(String name) {
        return this.jpaQueryFactory
                .selectFrom(member)
                .where(member.name.eq(name))
                .fetch();
    }

    public Member findFirstByName(String name) {
        return this.jpaQueryFactory
                .selectFrom(member)
                .where(member.name.eq(name))
                .limit(1)
                .fetchOne();
    }

    public List<Member> findMemberByTeamName(String teamName) {
        return this.jpaQueryFactory
                .select(
                        Projections.fields(Member.class, member.id, member.name, member.teamId)
                )
                .from(member)
                .join(team).on(member.teamId.eq(team.id))
                .where(team.name.eq(teamName))
                .fetch();
    }
}
```  

```java
import static com.windowforsun.querydslmapping.nomapping.QMember.*;
import static com.windowforsun.querydslmapping.nomapping.QTeam.*;

@Repository
public class TeamRepositorySupport extends QuerydslRepositorySupport {
    private final JPAQueryFactory jpaQueryFactory;

    public TeamRepositorySupport(JPAQueryFactory jpaQueryFactory) {
        super(Team.class);
        this.jpaQueryFactory = jpaQueryFactory;
    }

    public Team findByName(String name) {
        return this.jpaQueryFactory
            .selectFrom(team)
            .where(team.name.eq(name))
            .fetchOne();
    }

    public List<Team> findTeamByMemberName(String memberName) {
        return this.jpaQueryFactory
            .select(
                Projections.fields(Team.class, team.id, team.name)
            )
            .from(member)
            .join(team).on(member.teamId.eq(team.id))
            .where(member.name.eq(memberName))
            .fetch();
    }
}
```  

`RepositorySupport` 를 구현하기 위해서는 `QClass` 가 필요하기 때문에 
해당 클래스에서 사용이 필요한 `QClass` 를 `import static` 을 사용해서 임포트 시켜주면 간편하게 사용할 수 있다.  

그리고 `findMemberByTeamName`, `findTeamByMemberName` 가 `Member`, `Team` 을 조인해서 원하는 데이터를 조회하는 메소드이다. 
연관관계 매핑을 사용하지 않는 경우 `on()` 을 사용해서 조인 조건을 명시해줘야 한다.  

먼저 `Spring JPA Repository` 에 대한 테스트 클래스는 아래와 같다. 

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class EntityNoMappingRepositoryTest {
    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private TeamRepository teamRepository;

    @BeforeEach
    public void setUp() {
        this.memberRepository.deleteAll();
        this.teamRepository.deleteAll();

        Team team1 = this.teamRepository.save(Team.builder()
                .name("team-1")
                .build());
        Team team2 = this.teamRepository.save(Team.builder()
                .name("team-2")
                .build());
        this.memberRepository.save(Member.builder()
                .name("a")
                .teamId(team1.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("b")
                .teamId(team1.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("a")
                .teamId(team2.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("c")
                .teamId(team2.getId())
                .build());
    }

    @Test
    public void member_findAllByName() {
        // when
        List<Member> actual = this.memberRepository.findAllByName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, everyItem(hasProperty("name", is("a"))));
    }

    @Test
    public void member_findFirstByName() {
        // when
        Member actual = this.memberRepository.findFirstByName("a");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("a"));
    }

    @Test
    public void team_findByName() {
        // when
        Team actual = this.teamRepository.findByName("team-1");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("team-1"));
    }
}
```  

다음으로 `QueryDSL RepositorySuppory` 를 사용한 테스트 클래스는 아래와 같다. 

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class EntityNoMappingRepositorySupportTest {
    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private TeamRepository teamRepository;
    @Autowired
    private MemberRepositorySupport memberRepositorySupport;
    @Autowired
    private TeamRepositorySupport teamRepositorySupport;

    @BeforeEach
    public void setUp() {
        this.memberRepository.deleteAll();
        this.teamRepository.deleteAll();

        Team team1 = this.teamRepository.save(Team.builder()
                .name("team-1")
                .build());
        Team team2 = this.teamRepository.save(Team.builder()
                .name("team-2")
                .build());
        this.memberRepository.save(Member.builder()
                .name("a")
                .teamId(team1.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("b")
                .teamId(team1.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("a")
                .teamId(team2.getId())
                .build());
        this.memberRepository.save(Member.builder()
                .name("c")
                .teamId(team2.getId())
                .build());
    }

    @Test
    public void member_findAllByName() {
        // when
        List<Member> actual = this.memberRepositorySupport.findAllByName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, everyItem(hasProperty("name", is("a"))));
    }

    @Test
    public void member_findFirstByName() {
        // when
        Member actual = this.memberRepositorySupport.findFirstByName("a");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("a"));
    }

    @Test
    public void member_join_findMemberByTeamName() {
        // when
        List<Member> actual = this.memberRepositorySupport.findMemberByTeamName("team-1");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("a"))));
        assertThat(actual, hasItem(hasProperty("name", is("b"))));
    }

    @Test
    public void team_findByName() {
        // when
        Team actual = this.teamRepositorySupport.findByName("team-1");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("team-1"));
    }

    @Test
    public void team_join_findTeamByMemberName() {
        // when
        List<Team> actual = this.teamRepositorySupport.findTeamByMemberName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("team-1"))));
        assertThat(actual, hasItem(hasProperty("name", is("team-2"))));
    }

}
```  


### 연관관계 매핑 O

이번에는 `Member`, `Team` 이 연관관계 매핑을 사용해서 `N:1` 관계가 구성된 경우이다. 
현재 양방향으로 연관관계 매핑이 돼있기 때문에, 관계가 맺어진 엔티티도 별도의 조인 없이 조회 해볼 수 있다.  

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_mapping")
@Table(name = "member_mapping")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_mapping")
@Table(name = "team_mapping")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // QueryDSL RepositorySupport 에서 Team 조회시 연관관계(Member)를 조회하기 위해 기 위해 FetchType.Eager 로 설정
    @OneToMany(mappedBy = "team", fetch = FetchType.EAGER)
    @Builder.Default
    private List<Member> members = new ArrayList<>();
}
```  

다음은 위 엔티티에 대한 `Spring JPA Repository` 클래스 내용이다. 

```java
@Repository
public interface MemberMappingRepository extends JpaRepository<Member, Long> {
    Member findFirstByName(String name);

    List<Member> findAllByName(String name);
}

@Repository
public interface TeamMappingRepository extends JpaRepository<Team, Long> {
    Team findByName(String name);
}
```  

`QueryDSL RepositorySupport` 에 대한 클래스로, 
조인을 수행하는 부분에 연관관계를 사용하지 않는 예시와 차이가 있다.  

```java
import static com.windowforsun.querydslmapping.mapping.QMember.*;
import static com.windowforsun.querydslmapping.mapping.QTeam.*;

@Repository
public class MemberMappingRepositorySupport extends QuerydslRepositorySupport {
    private final JPAQueryFactory jpaQueryFactory;

    public MemberMappingRepositorySupport(JPAQueryFactory jpaQueryFactory) {
        super(Member.class);
        this.jpaQueryFactory = jpaQueryFactory;
    }

    public List<Member> findAllByName(String name) {
        return this.jpaQueryFactory
                .selectFrom(member)
                .where(member.name.eq(name))
                .fetch();
    }

    public Member findFirstByName(String name) {
        return this.jpaQueryFactory
                .selectFrom(member)
                .where(member.name.eq(name))
                .limit(1)
                .fetchOne();
    }

    public List<Member> findMemberByTeamName(String teamName) {
        return this.jpaQueryFactory
                .selectFrom(member)
                .join(member.team, team)
                .where(team.name.eq(teamName))
                .fetch();
    }
}
```  

```java
import static com.windowforsun.querydslmapping.mapping.QMember.*;
import static com.windowforsun.querydslmapping.mapping.QTeam.*;

@Repository
public class TeamMappingRepositorySupport extends QuerydslRepositorySupport {
    private final JPAQueryFactory jpaQueryFactory;

    public TeamMappingRepositorySupport(JPAQueryFactory jpaQueryFactory) {
        super(Team.class);
        this.jpaQueryFactory = jpaQueryFactory;
    }

    public Team findByName(String name) {
        return this.jpaQueryFactory
                .selectFrom(team)
                .where(team.name.eq(name))
                .fetchOne();
    }

    public List<Team> findTeamByMemberName(String memberName) {
        return this.jpaQueryFactory
                .selectFrom(team)
                .join(team.members, member)
                .where(member.name.eq(memberName))
                .fetch();
    }
}
```  

앞선 예제와 동일하게 `import static` 을 사용해서 사용에 필요한 `QClass` 를 임포트해주고 있다. 
그리고 `QueryDSL` 로 별도의 조인을 수행하고 있는 `findMemberByTeamName`, `findTeamByMemberName` 을 보면, 
연관관계를 사용하지 않았을 때와 차이가 있다. 
차이점은 `.on()` 을 사용하지 않더라도 연관관계 매핑 설정을 바탕으로 조인 조건이 이뤄 진다.  


연관관계가 구성된 경우 `Spring JPA Repository` 를 사용해서 조회할 때 연관된 엔티티도 함꼐 조회해 볼 수 있다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class EntityMappingRepositoryTest {
    @Autowired
    private MemberMappingRepository memberMappingRepository;
    @Autowired
    private TeamMappingRepository teamMappingRepository;

    @BeforeEach
    public void setUp() {
        this.memberMappingRepository.deleteAll();
        this.teamMappingRepository.deleteAll();

        Team team1 = this.teamMappingRepository.save(Team.builder()
                .name("team-1")
                .build());
        Team team2 = this.teamMappingRepository.save(Team.builder()
                .name("team-2")
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("a")
                .team(team1)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("b")
                .team(team1)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("a")
                .team(team2)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("c")
                .team(team2)
                .build());
    }

    @Test
    public void member_findAllByName() {
        // when
        List<Member> actual = this.memberMappingRepository.findAllByName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, everyItem(hasProperty("name", is("a"))));
        assertThat(actual, everyItem(hasProperty("team", notNullValue())));
    }

    @Test
    public void member_findFirstByName() {
        // when
        Member actual = this.memberMappingRepository.findFirstByName("a");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("a"));
        assertThat(actual.getTeam(), notNullValue());
    }

    @Test
    public void team_findByName() {
        // when
        Team actual = this.teamMappingRepository.findByName("team-1");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("team-1"));
        assertThat(actual.getMembers(), hasSize(2));
    }
}
```  

`QueryDSL RepositorySupport` 또한 연관관계 매핑을 통해 연관된 엔티티조회가 가능하다. 

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class EntityMappingRepositorySupportTest {
    @Autowired
    private MemberMappingRepository memberMappingRepository;
    @Autowired
    private TeamMappingRepository teamMappingRepository;
    @Autowired
    private MemberMappingRepositorySupport memberMappingRepositorySupport;
    @Autowired
    private TeamMappingRepositorySupport teamMappingRepositorySupport;

    @BeforeEach
    public void setUp() {
        this.memberMappingRepository.deleteAll();
        this.teamMappingRepository.deleteAll();

        Team team1 = this.teamMappingRepository.save(Team.builder()
                .name("team-1")
                .build());
        Team team2 = this.teamMappingRepository.save(Team.builder()
                .name("team-2")
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("a")
                .team(team1)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("b")
                .team(team1)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("a")
                .team(team2)
                .build());
        this.memberMappingRepository.save(Member.builder()
                .name("c")
                .team(team2)
                .build());
    }

    @Test
    public void member_findAllByName() {
        // when
        List<Member> actual = this.memberMappingRepositorySupport.findAllByName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, everyItem(hasProperty("name", is("a"))));
        assertThat(actual, everyItem(hasProperty("team", notNullValue())));
    }

    @Test
    public void member_findFirstByName() {
        // when
        Member actual = this.memberMappingRepositorySupport.findFirstByName("a");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("a"));
        assertThat(actual.getTeam(), notNullValue());
    }

    @Test
    public void member_join_findMemberByTeamName() {
        // when
        List<Member> actual = this.memberMappingRepositorySupport.findMemberByTeamName("team-1");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("a"))));
        assertThat(actual, hasItem(hasProperty("name", is("b"))));
        assertThat(actual, everyItem(hasProperty("team", notNullValue())));
    }

    @Test
    public void team_findByName() {
        // when
        Team actual = this.teamMappingRepositorySupport.findByName("team-1");

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getName(), is("team-1"));
        assertThat(actual.getMembers(), hasSize(2));
    }

    @Test
    public void team_join_findTeamByMemberName() {
        // when
        List<Team> actual = this.teamMappingRepositorySupport.findTeamByMemberName("a");

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("team-1"))));
        assertThat(actual, hasItem(hasProperty("name", is("team-2"))));
        assertThat(actual, everyItem(hasProperty("members", hasSize(greaterThan(0)))));
    }
}
```  



---  
## Reference
[QueryDSL JPA](http://querydsl.com/static/querydsl/5.0.0/reference/html_single/#jpa_integration)  