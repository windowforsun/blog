--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Cascade 영속성 전이"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 엔티티 연관관계에서 영속성 전이의 종류와 특징에 대해서 알아보자'
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
    - Cascade
    - Orphan
toc: true
use_math: true
---  

## Cascade(영속성 전이)
`Spring JPA` 는 연관관계를 통해 엔티티(테이블)간의 참조(관계)를 관리한다. 
그리고 각 엔티티는 생명주기를 가지게 되는데, 
`영속성 전이` 는 연관관계로 맺어진 엔티티들에게도 생명주기(상태변화)를 전파하는 것을 의미한다. 

> - [JPA 연관관계]({{site.baseurl}}{% link _posts/spring/2022-03-01-spring-practice-jpa-association-mapping.md %})
> - [생명주기(상태변화)]({{site.baseurl}}{% link _posts/spring/2020-10-28-spring-concept-jpa-persistence-context.md %})

`Spring JPA` 는 이런 영속성 전이를 연관관계 매핑 설정에 아래와 같은 `Cascade` 속성을 설정하는 방법으로 제공한다. 

- `CascadeType.PERSIST` : `EntityManager.persist()` 동작 전파로, 연관된 엔티티를 함께 영속상태로 설정한다. (INSERT)
- `CascadeType.REMOVE` : `EntityManager.remove()` 동작 전파로, 연관된 엔티티를 함께 삭제한다. (DELETE)
- `CascadeType.DETACH` : `EntityManager.detach()` 동작 전파로, 연관된 엔티티를 함께 영속상태에서 제거한다. 
- `CascadeType.REFRESH` : `EntityManager.refresh()` 동작 전파로, 연관된 엔티티를 함께 다시 `DB` 로 부터 읽어온다. (SELECT)
- `CascadeType.MERGE` : `EntityManager.merge()` 동작 전파로, 연관된 엔티티를 함께 다시 영속상태로 설정한다. (UPDATE)
- `CascadeType.ALL` : 모든 상태 변화에 대해 연관된 엔티티를 함께 적용한다. 

영속성 전이는 아래와 같이 `@ManyToOne` 과 같은 어노테이션에 `cascade` 속성 값으로 설정할 수 있다. 

```java
@ManayToOne(cascade = CascadeType.PERSIST)
private Team team;
```  

만약 여러 타입을 설정하고 싶을 경우 아래와 같이 배열로 설정 가능하다.  

```java
@ManyToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private Team team;
```  

그리고 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 `고아`(`Orphan`)객체라고 하는데, 
이를 자동으로 삭제해주는 기능도 제공해 준다.  

### 영속성 전이 주의사항
- `Cascade` 설정은 편리한 기능이지만, 잘못 된 설정으로 인해 의도치 않은 동작과 성능에 영향을 줄 수 있다. 
- `CascadeType.ALL`, `CascadeType.REMOVE` 를 설정할 때는 발생가능한 영속성 문제에 대해서 사전 확인이 필요하다. 
- 양방향 연관관계의 경우 `Cascade` 설정은 한쪽에서 관리하는게 좋다. 
  - 양방향 연관관계에서 `CacadeType.REMOVE` 를 양쪽에 설정할 경우, 의도치 않은 엔티티 삭제가 발생할 수 있다. 
- `Cacade` 설정은 편리함 목적보다는 연관관계 엔티티 생명주기 관리 용도로 사용해야 한다. 

### CascadeType.PERSIST

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_persist")
@Table(name = "member_persist")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.PERSIST)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_persist")
@Table(name = "team_persist")
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

`Member`(부모 엔티티) 를 `persist`(`INSERT`)하면 `Team`(자식 엔티티)도 함께 저장된다. 
반대의 경우는 `Team` 만 저장된다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadePersistTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void persistMember_persistWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // when
        assertThat(actualMember.getId(), is(member.getId()));
        assertThat(actualMember.getTeam().getId(), is(team.getId()));
        assertThat(actualTeam.getId(), is(team.getId()));
        assertThat(actualTeam.getMembers().get(0).getId(), is(member.getId()));
    }

    @Test
    public void persistTeam_persistOnlyTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // when
        assertThat(member.getId(), nullValue());
        assertThat(actualTeam.getId(), is(team.getId()));
        assertThat(actualTeam.getMembers(), empty());
    }
}
```  

### CascadeType.REMOVE

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_remove")
@Table(name = "member_remove")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.REMOVE)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_remove")
@Table(name = "team_remove")
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

`Member`(부모 엔티티) 를 `remove`(`DELETE`)하면 `Team`(자식 엔티티)도 함께 삭제된다. 
반대의 경우 `Member` 가 `Team` 의 `PK` 를 `FK` 로 가지고 있기 때문에 예외가 발생한다. 
그리고 여러 `Member` 가 동일한 `Team` 을 참조하고 있을 경우에 하나의 `Member` 를 삭제 하게 되면, 
다른 `Member` 가 삭제하려는 `Team` 의 `PK` 를 `FK` 로 가지고 있어 예외가 발생한다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeRemoveTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void removeMember_removeWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        this.entityManager.remove(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember, nullValue());
        assertThat(actualTeam, nullValue());
    }

    @Test
    public void removeAddress_throwsPersistenceException() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());

        // when
        this.entityManager.remove(team);
        assertThrows(PersistenceException.class, () -> {
            this.entityManager.flush();
            this.entityManager.clear();
        });
    }

    @Test
    public void multipleMember_removeMember1_throwsPersistenceException() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member1 = Member.builder()
                .name("a")
                .team(team)
                .build();
        Member member2 = Member.builder()
                .name("b")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member1);
        this.entityManager.persist(member2);
        this.entityManager.flush();
        this.entityManager.clear();
        member1 = this.entityManager.find(Member.class, member1.getId());

        // when
        this.entityManager.remove(member1);
        assertThrows(PersistenceException.class, () -> {
            this.entityManager.flush();
            this.entityManager.clear();
        });
    }
}
```  

#### 역방향(mappedBy) 에서 CascadeType.REMOVE 설정

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_remove_mappedby")
@Table(name = "member_remove_mappedby")
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
@Entity(name = "team_remove_mappedby")
@Table(name = "team_remove_mappedby")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.REMOVE)
    @Builder.Default
    private List<Member> members = new ArrayList<>();
}
```  

`Team`(자식 엔티티) 을 `remove` 하면 `Member`(부모 엔티티) 까지 모두 삭제된다. 
반대의 경우에는 삭제하려는 `Member` 만 삭제되고 `Team` 은 삭제되지 않는다. 
그리고 여러 `Member` 가 동일한 `Team` 을 참조하고 있는 경우, 
`Team` 을 삭제하면 참조하고 있는ㄷ 모든 `Member` 가 삭제된다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeRemoveMappedByTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void removeTeam_removeWithMember() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        this.entityManager.remove(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember, nullValue());
        assertThat(actualTeam, nullValue());
    }
    
    @Test
    public void removeMember_removeOnlyMember() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        this.entityManager.remove(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember, nullValue());
        assertThat(actualTeam, notNullValue());
    }

    @Test
    public void multipleMember_removeTeam_removeAllMembers() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member1 = Member.builder()
                .name("a")
                .team(team)
                .build();
        Member member2 = Member.builder()
                .name("b")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member1);
        this.entityManager.persist(member2);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        this.entityManager.remove(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember1, nullValue());
        assertThat(actualMember2, nullValue());
        assertThat(actualTeam, nullValue());
    }
}
```  

#### 양방향에서 CascadeType.REMOVE 설정

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_remove_bidirectional")
@Table(name = "member_remove_bidirectional")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.REMOVE)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_remove_bidirectional")
@Table(name = "team_remove_bidirectional")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.REMOVE)
    @Builder.Default
    private List<Member> members = new ArrayList<>();
}
```  

`Member`(부모 엔티티)를 `remove` 하면 `Team`(자식 엔티티)까지 삭제되고, 
반대의 경우에도 동일하다. 
그리고 여러 `Member` 가 동일한 `Team` 을 참조하는 경우, 
`Team` 을 삭제하던 `Member` 중 하나를 삭제하던 모든 엔티티가 삭제 된다.  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeRemoveBidirectionalTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void removeMember_removeWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        this.entityManager.remove(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember, nullValue());
        assertThat(actualTeam, nullValue());
    }

    @Test
    public void removeTeam_removeWithMember() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        this.entityManager.remove(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember, nullValue());
        assertThat(actualTeam, nullValue());
    }


    @Test
    public void multipleMember_removeMember1_removeAllMembers() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member1 = Member.builder()
                .name("a")
                .team(team)
                .build();
        Member member2 = Member.builder()
                .name("b")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member1);
        this.entityManager.persist(member2);
        this.entityManager.flush();
        this.entityManager.clear();
        member1 = this.entityManager.find(Member.class, member1.getId());
        this.entityManager.remove(member1);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember1, nullValue());
        assertThat(actualMember2, nullValue());
        assertThat(actualTeam, nullValue());
    }

    @Test
    public void multipleMember_removeTeam_removeAllMembers() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member1 = Member.builder()
                .name("a")
                .team(team)
                .build();
        Member member2 = Member.builder()
                .name("b")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member1);
        this.entityManager.persist(member2);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        this.entityManager.remove(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember1 = this.entityManager.find(Member.class, member1.getId());
        Member actualMember2 = this.entityManager.find(Member.class, member2.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember1, nullValue());
        assertThat(actualMember2, nullValue());
        assertThat(actualTeam, nullValue());
    }
}
```  


### CascadeType.DETACH

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_detach")
@Table(name = "member_detach")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.DETACH)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_detach")
@Table(name = "team_detach")
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

`Member`(부모 엔티티)를 `detach` 하게 되면 `Team`(자식 엔티티)까지 함께 `detach` 된다. 
반대의 경우는 `Team` 만 `detach` 된다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeDetachTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }
    
    @Test
    public void detachMember_detachWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        member = this.entityManager.find(Member.class, member.getId());

        // when
        this.entityManager.detach(member);

        // then
        assertThat(this.entityManager.contains(member), is(false));
        assertThat(this.entityManager.contains(team), is(false));
    }

    @Test
    public void detachTeam_detachOnlyTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        member = this.entityManager.find(Member.class, member.getId());

        // when
        this.entityManager.detach(team);

        // then
        assertThat(this.entityManager.contains(member), is(true));
        assertThat(this.entityManager.contains(team), is(false));
    }
}
```  

### CascadeType.REFRESH

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_refresh")
@Table(name = "member_refresh")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.REFRESH)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_refresh")
@Table(name = "team_refresh")
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

`Member`(부모 엔티티)를 `refresh`(`SELECT`) 하면 `Team`(자식 엔티티)까지 함께 조회 된다. 
반대의 경우에는 `Team` 만 `refresh` 된다. 

> `refresh` 동작은 `DB` 에서 엔티티를 다시 조회하는 동작으로 연관관계로 인해 연속적인 발생은 성능 저히가 발생할 수 있어 사용에 주의가 필요하다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeRefreshTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void refreshMember_refreshWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        member = this.entityManager.find(Member.class, member.getId());
        team.setName("team-2");
        member.setName("b");

        // when
        this.entityManager.refresh(member);

        // then
        assertThat(member.getName(), is("a"));
        assertThat(member.getTeam().getName(), is("team-1"));
        assertThat(team.getName(), is("team-1"));
        assertThat(team.getMembers().get(0).getName(), is("a"));
    }

    @Test
    public void refreshTeam_refreshOnlyTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team = this.entityManager.find(Team.class, team.getId());
        member = this.entityManager.find(Member.class, member.getId());
        team.setName("team-2");
        member.setName("b");

        // when
        this.entityManager.refresh(team);

        // then
        assertThat(member.getName(), is("b"));
        assertThat(member.getTeam().getName(), is("team-1"));
        assertThat(team.getName(), is("team-1"));
        assertThat(team.getMembers().get(0).getName(), is("b"));
    }
}
```  


### CascadeType.MERGE

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_merge")
@Table(name = "member_merge")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(cascade = CascadeType.MERGE)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_merge")
@Table(name = "team_merge")
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

`Member`(부모 엔티티) 를 `merge`(`UPDATE`) 하면 `Team`(자식 엔티티) 까지 함께 갱신 된다. 
반대의 경우 `Team` 만 `merge` 된다. 

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeMergeTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void mergeMember_mergeWithTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team.setName("team-2");
        member.setName("b");
        this.entityManager.merge(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember.getName(), is("b"));
        assertThat(actualMember.getTeam().getName(), is("team-2"));
        assertThat(actualTeam.getName(), is("team-2"));
        assertThat(actualTeam.getMembers().get(0).getName(), is("b"));
    }

    @Test
    public void mergeTeam_mergeOnlyTeam() {
        // given
        Team team = Team.builder()
                .name("team-1")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team)
                .build();
        this.entityManager.persist(team);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        team.setName("team-2");
        member.setName("b");
        this.entityManager.merge(team);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam = this.entityManager.find(Team.class, team.getId());

        // then
        assertThat(actualMember.getName(), is("a"));
        assertThat(actualMember.getTeam().getName(), is("team-2"));
        assertThat(actualTeam.getName(), is("team-2"));
        assertThat(actualTeam.getMembers().get(0).getName(), is("a"));
    }
}
```  

### 고아 객체(Orphan)

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_orphan")
@Table(name = "member_orphan")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToOne(orphanRemoval = true)
    private Team team;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_orphan")
@Table(name = "team_orphan")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToOne(mappedBy = "team")
    private Member member;
}
```  

`Member`(부모 엔티티)에 참조되는 `Team`(자식 엔티티)를 다른 엔티티로 변경했을 때, 
이제 참조 되지 않아 고아 객체가 된 엔티티를 자동으로 삭제 해준다. 
`orphanRemoval` 설정은 `@OneToOne`, `@OneToMany` 어노테이션에서만 사용 할 수 있다. 
`CascadeType.REMOVE` 와는 다른 역할의 동작이므로 차이점에 대해 잘 알고 사용해야 한다.  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class CascadeOrphanTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void change_team_removedOrphanTeam() {
        // given
        Team team1 = Team.builder()
                .name("team-1")
                .build();
        Team team2 = Team.builder()
                .name("team-2")
                .build();
        Member member = Member.builder()
                .name("a")
                .team(team1)
                .build();
        this.entityManager.persist(team1);
        this.entityManager.persist(team2);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.setTeam(team2);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actualMember = this.entityManager.find(Member.class, member.getId());
        Team actualTeam1 = this.entityManager.find(Team.class, team1.getId());
        Team actualTeam2 = this.entityManager.find(Team.class, team2.getId());

        // then
        assertThat(actualMember.getTeam().getId(), is(team2.getId()));
        assertThat(actualTeam1, nullValue());
        assertThat(actualTeam2.getMember().getId(), is(member.getId()));
    }
}
```  


---  
## Reference