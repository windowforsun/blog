--- 
layout: single
classes: wide
title: "[Spring 실습] JPA 연관관계 매핑"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 를 사용한 객체 중심 연관관계 매핑에 대해 알아보자'
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
    - ORM
    - JPA Modeling
toc: true
use_math: true
---  

## Data Modeling
애플리케이션에서 `RDB` 에 쿼리를 수행하는 방법은 주로 `ORM`(`JPA`), `SQL Mapper`(`MyBatis`) 로 나눌 수 있다. 
이 둘은 애플리케이션에 쿼리를 수행할 수 있다는 목적에 있어서는 비슷하지만, 
사용 방법과 특징은 확연히 다른 성격을 가지고 있다.  

이번 포스트에서는 `JPA` 를 사용해서 객체 중심으로 모델링을 수행하는 객체 연관관계에 대해서 알아보고, 
몇가지 케이스를 다뤄보려고 한다. 
그리고 테이블 중심으로 모델링하는 테이블 연관관계와의 차이점에 대해서 간략하게 비교하며 설명을 진행한다.  

## 객체 연관관계와 테이블 연관관계
테이블 연관관계의 특징과 기본적인 객체 연관관계를 맺을 때 특징에 대해서 알아본다. 

### 테이블 연관관계
`ORM`(`JPA`)를 사용하고 있지 않다면, 일반적인 데이터 모델링이라고 할 수 있다. 
테이블 연관관계는 외래키(`Foreign Key`) 를 사용해서 관계를 맺는다.   

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-1.png)  

- 하나의 `MEMBER` 은 하나의 `TEAM` 에 포함 될 수도 있고 안될 수 있다. 
- `TEAM` 에 소속된 `MEMBER` 는 하나도 없을 수도 있고 1명 이상일 수도 있다. 
- 테이블 연관관계는 `TEAM_ID` 라는 외래키(`FK`)를 통해 연관관계를 맺는다. 
- `MEMBER` 와 `TEAM` 은 양방향 관계이다.
  - `MEMBER` 의 `TEAM_ID` 를 사용해서 `TEAM` 을 조회 할 수 있다. 
  - `TEAM` 의 `ID` 를 사용해서 `MEMBER` 를 조회 할 수 있다. 
	
```sql
SELECT
	*
FROM
	MEMBER M
JOIN
	TEAM T
ON 
	M.TEAM_ID = T.ID;


SELECT
	*
FROM
	TEAM T
JOIN
	MEMBER M
ON
	T.ID = M.TEAM_ID;
```  

### 객체 연관관계
`JPA` 와 같은 `ORM` 을 사용할 때는 객체 중심으로 연관관계 모델링을 수행해줘야 한다. 
이는 객체 중심이기 때문에 `참조` 를 사용해서 관계를 맺는다.  

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-2.png)  

- `Member` 엔티티는 `Member.team` 이라는 외래키 역할을 하는 필드를 통해 `Team` 엔티티와 연관관계를 맺는다. 
- `Member` 와 `Team` 은 단방향 관계이다. 
  - `Member` 에서는 `team` 필드를 통해 `Team` 을 조회할 수 있지만, `Team` 에서는 `Member` 를 조회할 수 없다.  

```java
Member member = memberRepository.findOne(id);
Team team = member.getTeam();
```  

### 객체, 테이블 연관관계 차이

구분|객체 연관관계|테이블 연관관계
---|---|---
연관관계 방식|참조|외래키
관계 방향|단방향|양방향
조회|참조를 가진 쪽에서만 조회 가능|양쪽에서 모두 조회 가능


## 객체 연관관계 방향
앞서 기본적인 객체 연관관계에 대해 설명할때 `단방향` 관계라고 했었다. 
하지만 객체 연관관계에서도 테이블 연관관계와 같이 `양방향` 관계를 맺을 수 있다. 
또한 객체 양방향 연관관계에서는 연관관계를 주도하는 주인 지정이 필요하다. 

### 단방향
단방향은 앞에서 살펴본 예제와 동일하다. 

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-3.png)  

`Member` 의 `Team` 객체 참조를 `MEMBER` 테이블의 `TEAM_ID(FK)` 로 매핑한다. 
단방향이기 때문에 `Member` 에서는 `Team` 을 조회할 수 있지만, 
`Team` 에서는 `Member` 를 조회할 수 없는 관계이다.  

### 양방향
객체 연관관계에서 `양방향` 관계를 맺기 위해서는 `Team` 엔티티에서 `Member` 엔티티쪽으로 새로운 참조를 추가해 주면된다. 

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-4.png)  

- `Team` 에 `members` 라는 필드를 통해 `Member` 쪽으로 참조를 추가했다. 
- 객체 참조는 2개 이지만, 테이블에서 `FK` 는 하나로만 구성돼 있다. 
- `Member`, `Team` 객체 연관관계 중 테이블에서 `FK` 를 가지고 있는 `MEMBER` 테이블 즉 `Member` 엔티티가 연관관계의 주인이다. 

### 양방향 연관관계 주인
- 객체 연관관계는 기본적으로 `단방향`이기 때문에, `앙뱡향` 관계를 위해서는 주인 지정이 필수 적이다. 
- `연관관계 주인`이 `FK` 를 가지게 되고 읽기, 쓰기, 수정, 삭제를 수행할 수 있다. 
- 주인이 아닌 쪽은 읽기만 수행 할 수 있다.
- 테이블 입장에서 `연관관계 주인`은 `FK`를 어느 테이블에서 관리할 지로 볼 수 있다. 
- 일반적으로 `N:1`, `1:N` 관계에서 `N` 테이블에서 `FK` 를 가지게 되고, 매핑되는 엔티티가 `연관관계 주인`이 된다. 


## 객체 연관관계 관련 어노테이션
객체 연관관계 매핑을 수행할 때 사용되는 어노테이션에 대해 알아본다. 

어노테이션|용도|
---|---
@ManyToOne|`N:1` 관계 매핑에 사용된다.
@OneToMany|`1:N` 관계 매핑에 사용된다. 
@OneToOne|`1:1` 관계 매핑에 사용된다.
@ManyToMany|`N:M` 관계 매핑에 사용된다.
@JoinColumn|관계 매핑에 사용하는 외래키의 테이블 컬럼명을 지정한다. 
@MapsId|엔티티에 포함된 임베디드 엔티티의 `PK` 를 엔티티의 `PK` 로 설정한다. ??????


## 테스트 환경 구성
- `build.gradle`

```groovy
buildscript {
    ext {
        lombokVersion = "1.18.18"
    }
}

plugins {
    id 'org.springframework.boot' version '2.5.0'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.windowforsun.jpamodeling'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'org.apache.commons:commons-lang3:3.12.0'
    compileOnly "org.projectlombok:lombok:${lombokVersion}"
    testCompileOnly "org.projectlombok:lombok:${lombokVersion}"
    annotationProcessor "org.projectlombok:lombok:${lombokVersion}"
    testAnnotationProcessor "org.projectlombok:lombok:${lombokVersion}"
    compile 'com.h2database:h2'
    implementation 'org.springframework.boot:spring-boot-devtools'

}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

test {
    useJUnitPlatform()
}
```  

- `application.yaml`

```yaml
spring:
  h2:
    console:
      enabled: true
  datasource:
    hikari:
      driver-class-name: org.h2.Driver
      jdbc-url: jdbc:h2:./data/testdb
      username: sa
      password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernates:
        show_sql: true # 쿼리 출력
        format_sql: true # 쿼리 이쁘게 출력
    show-sql: true # 뭐든 쿼리 출력(ddl 포함)
```  

## 객체 연관관계 다중성 매핑
객체 연관관계는 테이블 연관관계와 동일하게 다중성에 대한 매핑이 가능하다.  
`다대일(N:1)`, `일대다(1:N)`, `일대일(1:1)`, `다대다(N:M)` 가 있고 
각 다중성 매핑에서 연관관계의 `단방향`, `양방향` 매핑을 수행하는 방법에 대해서도 함께 알아본다.  

### 다대일(N:1) 매핑
- `@ManyToOne` 어노테이션을 사용해서 `FK` 와 매핑한다. 
- `@JoinColumn` 을 사용하면 `FK` 컬럼명을 지정할 수 있다. 
- `@ManyToOne` 의 `optional` 속성을 통해 `Not Null` 제약 조건 추가를 할 수 있다.
  - `optional=true` : `null` 허용(default)
  - `optionsl=false` : `Not Null` 제약 조건 설정
- 일반적으로 `N` 인 `Member` 가 `FK` 를 갖는다. 
	

#### 다대일 단방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-3.png)

- `FK` 를 갖는 `Member` 에서만 `Team` 을 조회할 수 있다. 
- `Team` 에서는 `Member` 조회가 불가하다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_1_uni")
@Table(name = "member_1_uni")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    // fk to team id
    @ManyToOne
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_1_uni")
@Table(name = "team_1_uni")
public class Team {
	@Id
	@GeneratedValue
	private Long id;
	private String name;
}
```  

```sql
create table member_1_uni
(
	id      bigint not null,
	name    varchar(255),
	team_id bigint,
	primary key (id)
);
create table team_1_uni
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);

alter table member_1_uni
	add constraint FK9ubcwqwxrusiq0rb80ucqev6o foreign key (team_id) references team_1_uni;
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToOneUnidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Team team1;
    private Team team2;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member);
        this.team1 = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team1);
        this.team2 = Team.builder()
                .name("team2")
                .build();
        this.entityManager.persist(this.team2);
    }
    
    @Test
    public void select_member_team() {
        // given
        this.member.setTeam(this.team1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team1.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualTeam.getId(), is(this.team1.getId()));
        assertThat(actualMember.getId(), is(this.member.getId()));
        assertThat(actualMember.getTeam().getId(), is(this.team1.getId()));
    }

    @Test
    public void update_member_updated() {
        // given
        this.member.setTeam(this.team1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member = this.entityManager.find(Member.class, this.member.getId());
        this.member.setTeam(this.team2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember.getTeam().getId(), is(this.team2.getId()));
    }
}
```  

#### 다대일 양방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-4.png)  

- `FK` 를 갖는 `Member` 가 `양방향` 연관관계의 주인이 된다. 
- 연관관계의 주인이 아닌 `Team` 의 `members` 에는 `@OneToManay(mappedBy=..)` 어노테이션 추가가 필요하다. 
- 주인인 `Member` 는 등록, 수정, 삭제가 모두 가능하고 주인이 아닌 `Team` 에서는 조회만 가능하다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_1_bi")
@Table(name = "member_1_bi")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인(fk to team id)
    @ManyToOne
    private Team team;

    public void setTeam(Team team) {
        if(this.team != null) {
            this.team.getMembers().remove(this);
        }

        this.team = team;
        team.getMembers().add(this);
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_1_bi")
@Table(name = "team_1_bi")
public class Team {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

	@OneToMany(mappedBy = "team")
	@Builder.Default
	private List<Member> members = new ArrayList<>();
}
```  

```sql
create table member_1_bi
(
	id      bigint not null,
	name    varchar(255),
	team_id bigint,
	primary key (id)
);
create table team_1_bi
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);

alter table member_1_bi
	add constraint FKc6plo1fi3dmfnotyfyw1f0laa foreign key (team_id) references team_1_bi;
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToOneBidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Team team1;
    private Team team2;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member);
        this.team1 = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team1);
        this.team2 = Team.builder()
                .name("team2")
                .build();
        this.entityManager.persist(this.team2);
    }
    
    @Test
    public void select_member_team() {
        // given
        this.member.setTeam(this.team1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team1.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualTeam.getId(), is(this.team1.getId()));
        assertThat(actualMember.getId(), is(this.member.getId()));
        assertThat(actualMember.getTeam().getId(), is(this.team1.getId()));
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member.getId()));
    }

    @Test
    public void update_member_updated() {
        // given
        this.member.setTeam(this.team1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member = this.entityManager.find(Member.class, this.member.getId());
        this.member.setTeam(this.team2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam1 = this.entityManager.find(Team.class, this.team1.getId());
        Team actualTeam2 = this.entityManager.find(Team.class, this.team2.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualTeam1.getMembers(), is(empty()));
        assertThat(actualTeam2.getMembers(), hasSize(1));
        assertThat(actualMember.getTeam().getId(), is(this.team2.getId()));
    }

    @Test
    public void update_team_notUpdated() {
        // given
        this.member.setTeam(this.team1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team1 = this.entityManager.find(Team.class, this.team1.getId());
        this.team1.setMembers(new ArrayList<>());
        this.team2 = this.entityManager.find(Team.class, this.team2.getId());
        this.team2.setMembers(Arrays.asList(this.entityManager.find(Member.class, this.member.getId())));
        this.entityManager.persist(this.team1);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam1 = this.entityManager.find(Team.class, this.team1.getId());
        Team actualTeam2 = this.entityManager.find(Team.class, this.team2.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualTeam2.getMembers(), is(empty()));
        assertThat(actualTeam1.getMembers(), hasSize(1));
        assertThat(actualMember.getTeam().getId(), is(this.team1.getId()));
    }
}
```  

### 일대다(1:N) 매핑
- 일대다 관계에서는 엔티티를 하나 이상 참조 할 수 있기 때문에 `Collection` 과 같은 자료형을 사용해야 한다. 
- `@OneToMany` 어노테이션은 `1` 쪽에 사용하고, 실제 `FK` 는 `N` 쪽에서 관리한다. 
- 연관관계의 주인 `1`이고 `1`에서 등록, 수정, 삭제를 수행한다.




#### 일대다 단방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-5.png)

- 일대다 단방향 매핑시에는 `1` 쪽에 `N` 쪽의 `FK` 컬럼을 매핑할 `@JoinColumn` 어노테이션 설정이 필요하다.
  - 설정되지 않은 경우 `JPA` 는 추가적인 연결 테이블을 중간에 두고 연관관계를 관리한다.
- 일대다 단방향 관계에서는 `FK` 를 갖는 테이블(`N`)과 엔티티의 관계를 갖는 테이블(`1`) 이 다르다.
  - 이러한 구조로 인해 관계를 설정할때 `UPDATE` 쿼리가 추가로 실행된다.
  - `Member` 저장과 `Team` 과 `Member` 관계를 한번에 설정할 수 없기 때문에 `INSERT` 이후 관계 설정을 위해 `FK` `UPDATE` 가 수행된다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_2_uni")
@Table(name = "team_2_uni")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @OneToMany
    // 조인컬럼 명시가 없으면 연결테이블을 두고 연관관계를 관리
    @JoinColumn(name = "team_id")
    @Builder.Default
    private List<Member> members = new ArrayList<>();

    public void addMember(Member member) {
        if(!this.members.contains(member)) {
            this.members.add(member);
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_2_uni")
@Table(name = "member_2_uni")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

}
```  

```sql
 create table member_2_uni
 (
	 id      bigint not null,
	 name    varchar(255),
	 team_id bigint,
	 primary key (id)
 );
create table team_2_uni
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);

alter table member_2_uni
	add constraint FKaj3smok4xsr26ifl9u3si2kne foreign key (team_id) references team_2_uni;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToManyUnidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.team = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team);
    }
    
    @Test
    public void select_member_team() {
        // given
        this.team.addMember(this.member1);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualMember.getId(), is(this.member1.getId()));
        assertThat(actualTeam.getId(), is(this.team.getId()));
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member1.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team.addMember(this.member1);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team = this.entityManager.find(Team.class, this.team.getId());
        this.team.setMembers(Arrays.asList(this.member2));
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member2.getId()));
    }
}
```  

#### 일대다 양방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-6.png)

- 일대다 양방향 매핑은 `1` 쪽을 `연관관계 주인` 으로 설정하는 매핑방식이다. 
- 일대다 양방향 매핑 주인이 `1` 쪽이므로 등록, 수정, 삭제도 `1` 쪽에서 처리한다. 
- 위와 같은 구조로 인해 일대다 양방향 매핑보다는 `연관관계 주인`이 `N` 쪽에 있는 `다대일 양방향` 매핑을 추천한다. 
- `N` 쪽인 `Member` 에도 `@JoinColumn` 어노테이션 설정이 필요하고, 주인이 아니기 때문에 `insertable`, `updatable` 은 모두 `false` 로 설정해 준다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_2_bi")
@Table(name = "team_2_bi")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @OneToMany
    // 조인컬럼 명시가 없으면 연결테이블을 두고 연관관계를 관리
    @JoinColumn(name = "team_id")
    @Builder.Default
    private List<Member> members = new ArrayList<>();

    public void addMember(Member member) {
        if(!this.members.contains(member)) {
            this.members.add(member);
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_2_bi")
@Table(name = "member_2_bi")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

	// team_id fk to team id
	@ManyToOne
	@JoinColumn(insertable = false, updatable = false)
	private Team team;

}
```  

```sql
 create table team_2_bi
 (
	 id   bigint not null,
	 name varchar(255),
	 primary key (id)
 );
create table member_2_bi
(
	id      bigint not null,
	name    varchar(255),
	team_id bigint,
	primary key (id)
);

alter table member_2_bi
	add constraint FK7qoq65m7904bqbt0ky89mana6 foreign key (team_id) references team_2_bi;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToManyBidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.team = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team);
    }
    
    @Test
    public void select_member_team() {
        // given
        this.team.addMember(this.member1);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualMember.getId(), is(this.member1.getId()));
        assertThat(actualTeam.getId(), is(this.team.getId()));
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualMember.getTeam().getId(), is(this.team.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team.addMember(this.member1);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team = this.entityManager.find(Team.class, this.team.getId());
        this.team.setMembers(Arrays.asList(this.member2));
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());

        // then
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member2.getId()));
        assertThat(actualMember1.getTeam(), nullValue());
        assertThat(actualMember2.getTeam().getId(), is(this.team.getId()));
    }

    @Test
    public void update_member_notUpdated() {
        // given
        this.team.addMember(this.member1);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member1 = this.entityManager.find(Member.class, this.member1.getId());
        this.member1.setTeam(null);
        this.member2 = this.entityManager.find(Member.class, this.member2.getId());
        this.member2.setTeam(this.entityManager.find(Team.class, this.team.getId()));
        this.member2.setTeam(this.team);
        this.entityManager.persist(this.member1);
        this.entityManager.persist(this.member2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());

        // then
        assertThat(actualTeam.getMembers(), hasSize(1));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualMember1.getTeam().getId(), is(this.team.getId()));
        assertThat(actualMember2.getTeam(), nullValue());
    }
}
```  

### 일대일(1:1) 매핑
일대일 관계에서는 양쪽 중 `FK` 를 가질 곳을 선택해야 하고, `FK` 를 갖는 엔티티가 관계의 주인이 된다.


#### 일대일 단방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-7.png)

- `FK` 를 갖은 `Member` 가 연관관계의 주인이 된다.
- 일대일 단방향 매핑은 다대일 단방향 매핑과 크게 다르지 않다.

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_3_uni")
@Table(name = "member_3_uni")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @OneToOne
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_3_uni")
@Table(name = "address_3_uni")
public class Address {
	@Id
	@GeneratedValue
	private Long id;
	private String addressMain;
	private String addressDetail;
}
```  

```sql
 create table member_3_uni
 (
	 id         bigint not null,
	 name       varchar(255),
	 address_id bigint,
	 primary key (id)
 );
create table address_3_uni
(
	id             bigint not null,
	address_detail varchar(255),
	address_main   varchar(255),
	primary key (id)
);

alter table member_3_uni
	add constraint FKgn5u7bgymchy8vwhcn0tofkcp foreign key (address_id) references address_3_uni;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneUnidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Address address1;
    private Address address2;
    private Member member;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.address1 = Address.builder()
                .addressMain("서을특별시")
                .addressDetail("1111호")
                .build();
        this.entityManager.persist(this.address1);
        this.address2 = Address.builder()
                .addressMain("경기도")
                .addressDetail("2222호")
                .build();
        this.entityManager.persist(this.address2);
        this.member = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member);
    }
    
    @Test
    public void select_member_address() {
        // given
        this.member.setAddress(this.address1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address1.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualAddress.getId(), is(this.address1.getId()));
        assertThat(actualMember.getId(), is(this.member.getId()));
        assertThat(actualMember.getAddress().getId(), is(this.address1.getId()));
    }

    @Test
    public void update_member_updated() {
        // given
        this.member.setAddress(this.address1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member = this.entityManager.find(Member.class, this.member.getId());
        this.member.setAddress(this.address2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember.getAddress(), notNullValue());
        assertThat(actualMember.getAddress().getId(), is(this.address2.getId()));
    }
}
```  

#### 일대일 양방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-8.png)

- 일대일 양방향 관계는 연관관계 주인이 아닌쪽인 `Address` 에 `@OnetoOne(mappedBy=..)` 설정이 필요하다. 
- 주인인 `Member` 에서는 조회, 등록, 수정, 삭제가 가능하고 주인이 아닌 `Address` 에서는 조회만 가능하다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_3_bi")
@Table(name = "member_3_bi")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @OneToOne
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_3_bi")
@Table(name = "address_3_bi")
public class Address {
	@Id
	@GeneratedValue
	private Long id;
	private String addressMain;
	private String addressDetail;

	@OneToOne(mappedBy = "address")
	private Member member;
}
```  

```sql
 create table member_3_bi
 (
	 id         bigint not null,
	 name       varchar(255),
	 address_id bigint,
	 primary key (id)
 );
create table address_3_bi
(
	id             bigint not null,
	address_detail varchar(255),
	address_main   varchar(255),
	primary key (id)
);

alter table member_3_bi
	add constraint FKhi6uhaqbisn1try1b13emd5i6 foreign key (address_id) references address_3_bi;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneBidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Address address1;
    private Address address2;
    private Member member;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.address1 = Address.builder()
                .addressMain("서을특별시")
                .addressDetail("1111호")
                .build();
        this.entityManager.persist(this.address1);
        this.address2 = Address.builder()
                .addressMain("경기도")
                .addressDetail("2222호")
                .build();
        this.entityManager.persist(this.address2);
        this.member = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member);
    }

    @Test
    public void select_member_address() {
        // given
        this.member.setAddress(this.address1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address1.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualAddress.getId(), is(this.address1.getId()));
        assertThat(actualAddress.getMember().getId(), is(this.member.getId()));
        assertThat(actualMember.getId(), is(this.member.getId()));
        assertThat(actualMember.getAddress().getId(), is(this.address1.getId()));
    }

    @Test
    public void update_member_updated() {
        // given
        this.member.setAddress(this.address1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member = this.entityManager.find(Member.class, this.member.getId());
        this.member.setAddress(this.address2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Address actualAddress1 = this.entityManager.find(Address.class, this.address1.getId());
        Address actualAddress2 = this.entityManager.find(Address.class, this.address2.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember.getAddress().getId(), is(this.address2.getId()));
        assertThat(actualAddress1.getMember(), nullValue());
        assertThat(actualAddress2.getMember().getId(), is(this.member.getId()));
    }

    @Test
    public void update_address_notUpdated() {
        // given
        this.member.setAddress(this.address1);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
        this.address1 = this.entityManager.find(Address.class, this.address1.getId());
        this.address1.setMember(null);
        this.address2 = this.entityManager.find(Address.class, this.address2.getId());
        this.address2.setMember(this.member);
        this.entityManager.persist(this.address1);
        this.entityManager.persist(this.address2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Address actualAddress1 = this.entityManager.find(Address.class, this.address1.getId());
        Address actualAddress2 = this.entityManager.find(Address.class, this.address2.getId());
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember.getAddress().getId(), is(this.address1.getId()));
        assertThat(actualAddress1.getMember().getId(), is(this.member.getId()));
        assertThat(actualAddress2.getMember(), nullValue());
    }
}
```  


#### 일대일 식별 관계 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-9.png)  

- 일대일 식별관계는 `Address`(자식테이블)에서 `Member`(부모 테이블)의 `PK` 를 `PK` 이자 `FK` 로 갖는다. 
- 자식 테이블인 `Address` 에 `@Mapsid` 를 사용해서 `PK` 와 `FK` 를 매핑한다. 
- 자식 테이블인 `Address` 가 `FK` 를 갖기 때문에 `연관관계 주인`이 된다. 
- 관계 설정은 자식 테이블인 `Address` 에서 수행 돼야 한다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_3_bi_iden")
@Table(name = "address_3_bi_iden")
public class Address {
    @Id
    private Long id;
    private String addressMain;
    private String addressDetail;

    // 연관관계 주인
    @MapsId
    @OneToOne
    @JoinColumn(name = "id")
    private Member member;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_3_bi_iden")
@Table(name = "member_3_bi_iden")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

	@OneToOne(mappedBy = "member")
	private Address address;

}
```  

```sql
 create table address_3_bi_iden
 (
	 id             bigint not null,
	 address_detail varchar(255),
	 address_main   varchar(255),
	 primary key (id)
 );
create table member_3_bi_iden
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);

alter table address_3_bi_iden
	add constraint FK5ff6pww09ui228faauc6no3yp foreign key (id) references member_3_bi_iden;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneBidirectionalIdentifyTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Address address;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.address = Address.builder()
                .addressMain("서을특별시")
                .addressDetail("1111호")
                .member(this.member1)
                .build();
        this.entityManager.persist(this.address);
    }

    @Test
    public void select_member_address() {
        // given
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        Address actualAddress = this.entityManager.find(Address.class, address.getId());

        // then
        assertThat(actualMember.getId(), is(this.member1.getId()));
        assertThat(actualMember.getAddress().getId(), is(this.address.getId()));
        assertThat(actualAddress.getMember().getId(), is(this.member1.getId()));
        assertThat(actualAddress.getId(), is(this.address.getId()));
        assertThat(actualMember.getId(), is(actualAddress.getId()));
    }
}
```  


### 다대다(N:M) 매핑
- 다대다 매핑은 정규화된 테이블 2개로 관계를 표현할 수 없기 때문에 추가적인 연결 테이블을 사용한다. 
- 일반적으로 일대다-다대다 관계로 연결 테이블을 사용한다. 
- `@ManyToMany` 어노테이션을 사용해서 연결 테이블과 그리고 참조 테이블의 관계가 구성된다. 
- 다대다 매핑에서 연결 테이블에 컬럼 추가는 불가하기 때문에, 컬럼 추가가 필요한 경우 연결 테이블을 별도의 엔티티로 만들고 일대다-다대다 관계로 매핑해줘야 한다.
- 다대다 엔티티 식별자 설정에는 다양한 전략을 사용할 수 있다.

#### 다대다 단방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-10.png)

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_4_uni")
@Table(name = "team_4_uni")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @ManyToMany
    @Builder.Default
    private List<Member> members = new ArrayList<>();

    public void addMember(Member member) {
        if(!this.members.contains(member)) {
            this.members.add(member);
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_4_uni")
@Table(name = "member_4_uni")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

}
```  

```sql
 create table team_4_uni
 (
	 id   bigint not null,
	 name varchar(255),
	 primary key (id)
 );
create table member_4_uni
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table team_4_uni_members
(
	team_4_uni_id bigint not null,
	members_id    bigint not null
);

alter table team_4_uni_members
	add constraint FK6qtwo2idy3xmt3lvrjpm9a1bo foreign key (members_id) references member_4_uni;
alter table team_4_uni_members
	add constraint FK7cb2hnycq81qyvwsy2synouhh foreign key (team_4_uni_id) references team_4_uni;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyUnidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Member member3;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.member3 = Member.builder()
                .name("c")
                .build();
        this.entityManager.persist(this.member3);
        this.team = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team);
    }

    @Test
    public void select_team_member() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualMember1.getId(), is(this.member1.getId()));
        assertThat(actualMember2.getId(), is(this.member2.getId()));
        assertThat(actualTeam.getId(), is(this.team.getId()));
        assertThat(actualTeam.getMembers(), hasSize(2));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualTeam.getMembers().get(1).getId(), is(this.member2.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team = this.entityManager.find(Team.class, this.team.getId());
        this.team.addMember(this.member3);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualTeam.getMembers(), hasSize(3));
        assertThat(actualTeam.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualTeam.getMembers().get(1).getId(), is(this.member2.getId()));
        assertThat(actualTeam.getMembers().get(2).getId(), is(this.member3.getId()));
    }
}
```  

#### 다대다 양방향 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-11.png)

- 다대다 양방향관계 에서는 `연관관계 주인` 인 `Team` 에서 등록, 수정, 삭제가 수행되고 주인이 아닌 `Member` 는 조회만 가능하다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_4_bi")
@Table(name = "team_4_bi")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // 연관관계 주인
    @ManyToMany
    @Builder.Default
    private List<Member> members = new ArrayList<>();

    public void addMember(Member member) {
        if(!this.members.contains(member)) {
            this.members.add(member);
        }

        if(!member.getTeams().contains(this)) {
            member.getTeams().add(this);
        }
    }

    public void removeMember(Member member) {
        if(member != null) {
            this.members.removeIf(m -> m.getId().equals(member.getId()));
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_4_bi")
@Table(name = "member_4_bi")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

	@ManyToMany(mappedBy = "members")
	@Builder.Default
	private List<Team> teams = new ArrayList<>();

	public void removeTeam(Team team) {
		if(team != null) {
			this.teams.removeIf(t -> t.getId().equals(team.getId()));
		}
	}
}
```  

```sql
 create table team_4_bi
 (
	 id   bigint not null,
	 name varchar(255),
	 primary key (id)
 );
create table member_4_bi
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table team_4_bi_members
(
	teams_id   bigint not null,
	members_id bigint not null
);

alter table team_4_bi_members
	add constraint FKkpgjhnabggvkl5uk6krn9irho foreign key (members_id) references member_4_bi;
alter table team_4_bi_members
	add constraint FKp6peywbysyxbja5ohfrr5bhsh foreign key (teams_id) references team_4_bi;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyBidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team1;
    private Team team2;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.team1 = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team1);
        this.team2 = Team.builder()
                .name("team2")
                .build();
        this.entityManager.persist(this.team2);
    }

    @Test
    public void select_team_member() {
        // given
        this.team1.addMember(this.member1);
        this.team1.addMember(this.member2);
        this.entityManager.persist(this.team1);
        this.team2.addMember(this.member2);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam1 = this.entityManager.find(Team.class, this.team1.getId());
        Team actualTeam2 = this.entityManager.find(Team.class, this.team2.getId());

        // then
        assertThat(actualMember1.getId(), is(this.member1.getId()));
        assertThat(actualMember2.getId(), is(this.member2.getId()));
        assertThat(actualTeam1.getId(), is(this.team1.getId()));
        assertThat(actualTeam2.getId(), is(this.team2.getId()));
        assertThat(actualTeam1.getMembers(), hasSize(2));
        assertThat(actualTeam1.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualTeam1.getMembers().get(1).getId(), is(this.member2.getId()));
        assertThat(actualTeam2.getMembers(), hasSize(1));
        assertThat(actualTeam2.getMembers().get(0).getId(), is(this.member2.getId()));
        assertThat(actualMember1.getTeams(), hasSize(1));
        assertThat(actualMember1.getTeams().get(0).getId(), is(this.team1.getId()));
        assertThat(actualMember2.getTeams(), hasSize(2));
        assertThat(actualMember2.getTeams().get(0).getId(), is(this.team1.getId()));
        assertThat(actualMember2.getTeams().get(1).getId(), is(this.team2.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team1.addMember(this.member1);
        this.team1.addMember(this.member2);
        this.entityManager.persist(this.team1);
        this.team2.addMember(this.member2);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team1 = this.entityManager.find(Team.class, this.team1.getId());
        this.team1.removeMember(this.member2);
        this.entityManager.persist(this.team1);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam1 = this.entityManager.find(Team.class, this.team1.getId());

        // then
        assertThat(actualMember2.getTeams(), hasSize(1));
        assertThat(actualMember2.getTeams().get(0).getId(), is(this.team2.getId()));
        assertThat(actualTeam1.getMembers(), hasSize(1));
        assertThat(actualTeam1.getMembers().get(0).getId(), is(this.member1.getId()));
    }

    @Test
    public void update_member_notUpdated() {
        // given
        this.team1.addMember(this.member1);
        this.team1.addMember(this.member2);
        this.entityManager.persist(this.team1);
        this.team2.addMember(this.member2);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();
        this.member2 = this.entityManager.find(Member.class, this.member2.getId());
        this.member2.removeTeam(this.team1);
        this.entityManager.persist(this.member2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam1 = this.entityManager.find(Team.class, this.team1.getId());

        // then
        assertThat(actualMember2.getTeams(), hasSize(2));
        assertThat(actualMember2.getTeams().get(0).getId(), is(this.team1.getId()));
        assertThat(actualMember2.getTeams().get(1).getId(), is(this.team2.getId()));
        assertThat(actualTeam1.getMembers(), hasSize(2));
        assertThat(actualTeam1.getMembers().get(0).getId(), is(this.member1.getId()));
        assertThat(actualTeam1.getMembers().get(1).getId(), is(this.member2.getId()));
    }
}
```  

#### 다대다 연결 엔티티 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-12.png)  

- 연결 테이블인 `TEAM_MEMBER` 에 컬럼을 추가하기(`timestamp`) 위해 `TeamMember` 엔티티를 생성해 매핑 시켰다. 
- `Team` -일대다- `TeamMember` -다대일- `Member` 와 같은 관계를 갖는다. 
- `TeamMember` 는 고유한 `PK` 를 가지고 있다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_4_rel")
@Table(name = "team_4_rel")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.PERSIST)
    @Builder.Default
    private List<TeamMember> teamMembers = new ArrayList<>();

    public void addMember(Member member) {
        boolean isExists = this.teamMembers.stream()
                .map(TeamMember::getMember)
                .map(Member::getId)
                .anyMatch(id -> id.equals(member.getId()));

        if(!isExists) {
            TeamMember teamMember = new TeamMember();
            teamMember.setTeam(this);
            teamMember.setMember(member);
            this.teamMembers.add(teamMember);
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_4_rel")
@Table(name = "member_4_rel")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_member_4_rel")
@Table(name = "team_member_4_rel")
public class TeamMember {
	@Id
	@GeneratedValue
	private Long id;

	// 연관관계 주인
	@ManyToOne
	private Team team;
	// 연관관계 주인
	@ManyToOne
	private Member member;

	@Temporal(TemporalType.TIMESTAMP)
	private Date timestamp;
}
```  

```sql
create table team_4_rel
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table member_4_rel
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table team_member_4_rel
(
	id        bigint not null,
	timestamp timestamp,
	member_id bigint,
	team_id   bigint,
	primary key (id)
);

alter table team_member_4_rel
	add constraint FKmxd5jupt5pabsvb08wye1fkve foreign key (member_id) references member_4_rel;
alter table team_member_4_rel
	add constraint FKlq5k6nygmsl0oa522nph9keud foreign key (team_id) references team_4_rel;

```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyRelationMappingTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Member member3;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.member3 = Member.builder()
                .name("c")
                .build();
        this.entityManager.persist(this.member3);
        this.team = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team);
    }

    @Test
    public void select_team_member() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualMember1, notNullValue());
        assertThat(actualMember2, notNullValue());
        assertThat(actualTeam, notNullValue());
        assertThat(actualMember1.getId(), is(this.member1.getId()));
        assertThat(actualMember2.getId(), is(this.member2.getId()));
        assertThat(actualTeam.getId(), is(this.team.getId()));
        assertThat(actualTeam.getTeamMembers(), hasSize(2));
        assertThat(actualTeam.getTeamMembers().get(0).getMember().getId(), is(this.member1.getId()));
        assertThat(actualTeam.getTeamMembers().get(1).getMember().getId(), is(this.member2.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team = this.entityManager.find(Team.class, this.team.getId());
        this.team.addMember(this.entityManager.find(Member.class, this.member3.getId()));
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualTeam.getTeamMembers(), hasSize(3));
        assertThat(actualTeam.getTeamMembers().get(0).getMember().getId(), is(this.member1.getId()));
        assertThat(actualTeam.getTeamMembers().get(1).getMember().getId(), is(this.member2.getId()));
        assertThat(actualTeam.getTeamMembers().get(2).getMember().getId(), is(this.member3.getId()));
    }
}
```  

#### 다대다 엔티티 식별관계 매핑

![그림 1]({{site.baseurl}}/img/spring/practice-jpa-association-mapping-13.png)

- `TeamMember` 는 고유한 `PK` 를 갖는게 아니라, `Team` 과 `Member` 의 각 `PK` 를 자신의 복합 `PK` 로 갖는다. 

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_4_rel_iden")
@Table(name = "team_4_rel_iden")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.PERSIST)
    @Builder.Default
    private List<TeamMember> teamMembers = new ArrayList<>();

    public void addMember(Member member) {
        TeamMemberId teamMemberId = new TeamMemberId(this.getId(), member.getId());
        boolean isExists = this.teamMembers.stream()
                .map(TeamMember::getId)
                .anyMatch(id -> id.equals(teamMemberId));

        if(!isExists) {
            TeamMember teamMember = new TeamMember();
            teamMember.setTeam(this);
            teamMember.setMember(member);
            this.teamMembers.add(teamMember);
        }
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_4_rel_iden")
@Table(name = "member_4_rel_iden")
public class Member {
	@Id
	@GeneratedValue
	private Long id;
	private String name;

}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_member_4_rel_iden")
@Table(name = "team_member_4_rel_iden")
@IdClass(TeamMemberId.class)
public class TeamMember {
	// 연관관계 주인
	@Id
	@ManyToOne
	private Team team;

	// 연관관계 주인
	@Id
	@ManyToOne
	private Member member;

	@Temporal(TemporalType.TIMESTAMP)
	private Date timestamp;

	public TeamMemberId getId() {
		return new TeamMemberId(this.team.getId(), this.member.getId());
	}
}

@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
public class TeamMemberId implements Serializable {
	private Long team;
	private Long member;
}
```  

```sql

 create table team_4_rel_iden
 (
	 id   bigint not null,
	 name varchar(255),
	 primary key (id)
 );
create table member_4_rel_iden
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table team_member_4_rel_iden
(
	timestamp timestamp,
	member_id bigint not null,
	team_id   bigint not null,
	primary key (member_id, team_id)
);

alter table team_member_4_rel_iden
	add constraint FKnxwvehycu9ays0qnwdabioyoh foreign key (member_id) references member_4_rel_iden;
alter table team_member_4_rel_iden
	add constraint FKd2amsbqfbb96mbbdiiug3i2ep foreign key (team_id) references team_4_rel_iden;
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyRelationIdentifyTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Member member3;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.member3 = Member.builder()
                .name("c")
                .build();
        this.entityManager.persist(this.member3);
        this.team = Team.builder()
                .name("team1")
                .build();
        this.entityManager.persist(this.team);
    }

    @Test
    public void select_team_member() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, this.member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, this.member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualMember1, notNullValue());
        assertThat(actualMember2, notNullValue());
        assertThat(actualTeam, notNullValue());
        assertThat(actualMember1.getId(), is(this.member1.getId()));
        assertThat(actualMember2.getId(), is(this.member2.getId()));
        assertThat(actualTeam.getId(), is(this.team.getId()));
        assertThat(actualTeam.getTeamMembers(), hasSize(2));
        assertThat(actualTeam.getTeamMembers().get(0).getMember().getId(), is(this.member1.getId()));
        assertThat(actualTeam.getTeamMembers().get(1).getMember().getId(), is(this.member2.getId()));
    }

    @Test
    public void update_team_updated() {
        // given
        this.team.addMember(this.member1);
        this.team.addMember(this.member2);
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();
        this.team = this.entityManager.find(Team.class, this.team.getId());
        this.team.addMember(this.entityManager.find(Member.class, this.member3.getId()));
        this.entityManager.persist(this.team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());

        // then
        assertThat(actualTeam.getTeamMembers(), hasSize(3));
        assertThat(actualTeam.getTeamMembers().get(0).getMember().getId(), is(this.member1.getId()));
        assertThat(actualTeam.getTeamMembers().get(1).getMember().getId(), is(this.member2.getId()));
        assertThat(actualTeam.getTeamMembers().get(2).getMember().getId(), is(this.member3.getId()));
    }
}
```  





---  
## Reference