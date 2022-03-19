--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Fetch 전략"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 에서 연관된 엔티티의 조회 쿼리 실행 시점을 설정할 수 있는 Fetch 전략에 대해 알아보자'
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
    - Fetch
    - Eager
    - Lazy
toc: true
use_math: true
---  

## JPA Fetch 전락
`Spring JPA` 는 연관된 엔티티의 조회 쿼리가 실행되는 시점을 선택할 수 있는 아래 2가지 방식을 제공한다. 

- `Eager`(즉시 로딩) : 엔티티를 조회할 때 연관된 엔티티도 함께 조회한다. 
- `Lazy`(지연 로딩) : 연관된 엔티티를 실제 사용할 떄 조회 한다. 

연관관계를 지정할 수 있는 4개의 어노테이션이 가지는 기본 `Fetch` 전략은 아래와 같다. 

- `@ManyToOne`, `@OneToOne` : `Eager`
- `@OneToMany`, `@ManyToMany` : `Lazy`

그리고 `Lazy` 전략을 사용하면 객체 그래프 탐색이 가능하다. 



### @ManyToOne
`@ManayToOne` 의 기본 `Fetch` 전략은 `Eager` 이다. 

#### Eager

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_many2one_eager")
@Table(name = "member_many2one_eager")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @ManyToOne(cascade = CascadeType.PERSIST)
    private Team team;

}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_many2one_eager")
@Table(name = "team_many2one_eager")
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

부모 엔티티인 `Member` 를 조회할때 매핑되는 `Team` 도 함께 조회 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToOneEagerFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.team = Team.builder()
                .name("team-1")
                .build();
        this.member1 = Member.builder()
                .name("a")
                .team(this.team)
                .build();
        this.member2 = Member.builder()
                .name("b")
                .team(this.team)
                .build();
        this.entityManager.persist(this.member1);
        this.entityManager.persist(this.member2);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_eager(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        Team actualTeam = actualMember.getTeam();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from member_many2one_eager",
                "left outer join team_many2one_eager"
        ));
    }

    @Test
    public void select_team_lazy(CapturedOutput capturedOutput) {
        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());
        List<Member> actualMembers = actualTeam.getMembers();

        // then
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_eager"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from member_many2one_eager"
                ))
        ));
        assertThat(actualMembers, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_eager"
                ),
                stringContainsInOrder(
                        "select",
                        "from member_many2one_eager"
                )
        ));
    }
}
```  

부모 `Member` 와 자식 `Team` 은 현재 양방향 `N:1` 매핑이 돼있다. 
그러므로 `Member` 를 조회할 때는 `@ManyToOne` 의 기본 `Fetch` 잔략이 `Eager` 을 사용하므로, 
아래 와 같이 `left outter join` 을 통해 `Team` 도 함께 조회 된다. 

```sql
select
    member0_.id as id1_9_0_,
    member0_.name as name2_9_0_,
    member0_.team_id as team_id3_9_0_,
    team1_.id as id1_17_1_,
    team1_.name as name2_17_1_ 
from
    member_many2one_eager member0_ 
left outer join
    team_many2one_eager team1_ 
        on member0_.team_id=team1_.id 
where
    member0_.id=?
```  

그리고 `Team` 을 조회할 때는 `@OneToMany` 기본 `Fetch` 전략이 `Lazy` 를 사용하기 때문에, 
먼저 `Team` 을 조회하고 나서, 실제 `members` 객체를 사용하는 시점에 `Member` 를 조회하는 쿼리가 실행 된다. 

```sql
# entityManager.find(Team) 시점에 실행
select
    team0_.id as id1_17_0_,
    team0_.name as name2_17_0_ 
from
    team_many2one_eager team0_ 
where
    team0_.id=?
    
# assertThat(actualMembers, not(empty())) 시점에 실행
select
    members0_.team_id as team_id3_9_0_,
    members0_.id as id1_9_0_,
    members0_.id as id1_9_1_,
    members0_.name as name2_9_1_,
    members0_.team_id as team_id3_9_1_
from
    member_many2one_eager members0_
where
    members0_.team_id=?
```  

#### Lazy

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_many2one_lazy")
@Table(name = "member_many2one_lazy")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @ManyToOne(cascade = CascadeType.PERSIST, fetch = FetchType.LAZY)
    private Team team;

}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_many2one_lazy")
@Table(name = "team_many2one_lazy")
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

부모 엔티티인 `Member` 의 `Fetch` 전략을 `Lazy` 로 설정했기 때문에, 
먼저 `Member` 를 조회하고, 실제 `team` 필드가 사용되는 시점에 `Team` 을 조회 한다. 

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToOneLazyFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.team = Team.builder()
                .name("team-1")
                .build();
        this.member1 = Member.builder()
                .name("a")
                .team(this.team)
                .build();
        this.member2 = Member.builder()
                .name("b")
                .team(this.team)
                .build();
        this.entityManager.persist(this.member1);
        this.entityManager.persist(this.member2);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        Team actualTeam = actualMember.getTeam();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2one_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from team_many2one_lazy"
                ))
        ));
        assertThat(actualTeam.getName(), is(this.team.getName()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2one_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from team_many2one_lazy"
                )
        ));
    }

    @Test
    public void select_team_lazy(CapturedOutput capturedOutput) {
        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());
        List<Member> actualMember = actualTeam.getMembers();

        // then
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from member_many2one_lazy"
                ))
        ));
        assertThat(actualMember, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from member_many2one_lazy"
                )
        ));
    }
}
```  

`Member` 를 조회하게 되면 `member` 테이블 조회가 발생하고, 
`team` 속성에 사용되는 시점에 `team` 테이블 조회가 아래와 같이 발생 한다. 

```sql
# this.entityManager.find(Member.class, this.member1.getId()) 시점에 실행
select
    member0_.id as id1_11_0_,
    member0_.name as name2_11_0_,
    member0_.team_id as team_id3_11_0_ 
from
    member_many2one_lazy member0_ 
where
    member0_.id=?
    
    
# assertThat(actualMember, not(empty())) 시점에 실행
select
    team0_.id as id1_19_0_,
    team0_.name as name2_19_0_
from
    team_many2one_lazy team0_
where
    team0_.id=?
```  

`Team` 을 조회할때는 `Lazy` 하게 `Team` 조회 후, `members` 사용 시점에 `Member` 를 조회 한다.


#### Eager(optional=false)

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_many2one_false")
@Table(name = "member_many2one_false")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @ManyToOne(cascade = CascadeType.PERSIST, fetch = FetchType.EAGER, optional = false)
    private Team team;

}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_many2one_false")
@Table(name = "team_many2one_false")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team")
    @Builder.Default
    private List<Member> member = new ArrayList<>();
}
```  


`@ManyToOne` 어노테이션의 속성중 `optional` 기본 값은 `true` 이다.
그러므로 `Eager` 전략일 때 `Member` 를 조회하면 `left outer join` 쿼리가 실행되는데,
이를 `inner join` 으로 변경하는 방법은 `optional` 속성 값을 `false` 로 설정 하면 된다.

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToOneOptionalFalseTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.team = Team.builder()
                .name("team-1")
                .build();
        this.member1 = Member.builder()
                .name("a")
                .team(this.team)
                .build();
        this.member2 = Member.builder()
                .name("b")
                .team(this.team)
                .build();
        this.entityManager.persist(this.member1);
        this.entityManager.persist(this.member2);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_eager_innerjoin(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from member_many2one_false",
                "inner join team_many2one_false"
        ));
    }

    @Test
    public void select_team_lazy(CapturedOutput capturedOutput) {
        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team.getId());
        List<Member> actualMembers = actualTeam.getMember();

        // then
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_false"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from member_many2one_false"
                ))
        ));
        assertThat(actualMembers, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2one_false"
                ),
                stringContainsInOrder(
                        "select",
                        "from member_many2one_false"
                )
        ));
    }
}
```  

`Member` 를 조회하면 `Eager` 전략을 사용중이므로 `Team` 도 함께 조회되지만, 
`left outer join` 이 실행되는게 아니라 `inner join` 이 실행되는 것을 확인 할 수 있다.  

```sql
select
    member0_.id as id1_10_0_,
    member0_.name as name2_10_0_,
    member0_.team_id as team_id3_10_0_,
    team1_.id as id1_18_1_,
    team1_.name as name2_18_1_ 
from
    member_many2one_false member0_ 
inner join
    team_many2one_false team1_ 
        on member0_.team_id=team1_.id 
where
    member0_.id=?
```  

`Team` 을 조회할때는 `Lazy` 하게 `Team` 조회 후, `members` 사용 시점에 `Member` 를 조회 한다.

### @OneToMany
`@OneToMany` 의 기본 `Fetch` 전략은 `Lazy` 이다.  

#### Lazy

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2many_lazy")
@Table(name = "member_one2many_lazy")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is lazy
    @OneToMany
    @JoinColumn(name = "member_id")
    @Builder.Default
    private List<Address> addressList = new ArrayList<>();

    public void addAddress(Address address) {
        this.addressList.add(address);
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2many_lazy")
@Table(name = "address_one2many_lazy")
public class Address {
    @Id
    @GeneratedValue
    private Long id;
    private String addressMain;
    private String addressDetail;

    @OneToOne
    @JoinColumn(insertable = false, updatable = false)
    private Member member;
}
```  

부모 엔티티인 `Member` 를 조회하면 `Member` 만 조회되고, 
자식 엔티티인 `Address` 는 `Lazy` 하게 실제 사용되는 시점에 조회 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToManyLazyFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address1;
    private Address address2;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address1);
        this.address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        this.entityManager.persist(this.address2);
        this.member = Member.builder()
                .name("a")
                .build();
        this.member.addAddress(this.address1);
        this.member.addAddress(this.address2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());
        List<Address> actualAddressList = actualMember.getAddressList();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2many_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from address_one2many_lazy"
                ))
        ));
        assertThat(actualAddressList, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2many_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from address_one2many_lazy"
                )
        ));
    }

    @Test
    public void select_address_eager(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address1.getId());

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from address_one2many_lazy",
                "left outer join member_one2many_lazy"
        ));
    }
}
```  

`Member` 를 조회할 때는 `@OneToMany` 에 `Lazy` 전략이므로 `member` 테이블 조회만 실행되고, 
`addressList` 가 실제로 사용되는 시점에 `Address` 인 `address` 테이블 조회 쿼리가 실행된다.  

```sql
# this.entityManager.find(Member.class, this.member.getId()) 시점에 실행
select
    member0_.id as id1_13_0_,
    member0_.name as name2_13_0_
from
    member_one2many_lazy member0_
where
    member0_.id=?
    
    
# assertThat(actualAddressList, not(empty())) 시점에 실행
select
    addresslis0_.member_id as member_i4_1_0_,
    addresslis0_.id as id1_1_0_,
    addresslis0_.id as id1_1_1_,
    addresslis0_.address_detail as address_2_1_1_,
    addresslis0_.address_main as address_3_1_1_,
    addresslis0_.member_id as member_i4_1_1_
from
    address_one2many_lazy addresslis0_
where
    addresslis0_.member_id=?
```  

그리고 `Address` 를 조회 할때는 `@ManyToOne` 에 `Eager` 동작이이 때문에, 
`Address`, `Member` 모두 함께 `left outer join` 으로 조회 된다.  

```sql
select
    address0_.id as id1_1_0_,
    address0_.address_detail as address_2_1_0_,
    address0_.address_main as address_3_1_0_,
    address0_.member_id as member_i4_1_0_,
    member1_.id as id1_13_1_,
    member1_.name as name2_13_1_ 
from
    address_one2many_lazy address0_ 
left outer join
    member_one2many_lazy member1_ 
        on address0_.member_id=member1_.id 
where
    address0_.id=?
```  

#### Eager

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2many_eager")
@Table(name = "member_one2many_eager")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is lazy
    @OneToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "member_id")
    @Builder.Default
    private List<Address> addressList = new ArrayList<>();

    public void addAddress(Address address) {
        this.addressList.add(address);
    }
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2many_eager")
@Table(name = "address_one2many_eager")
public class Address {
    @Id
    @GeneratedValue
    private Long id;
    private String addressMain;
    private String addressDetail;

    @OneToOne
    @JoinColumn(insertable = false, updatable = false)
    private Member member;
}
```  

부모 엔티티인 `Member` 를 조회하는 시점에 `Address` 엔티티까지 함께 조회 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToManyEagerFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address1;
    private Address address2;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address1);
        this.address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        this.entityManager.persist(this.address2);
        this.member = Member.builder()
                .name("a")
                .build();
        this.member.addAddress(this.address1);
        this.member.addAddress(this.address2);
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_eager(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());
        List<Address> actualAddressList = actualMember.getAddressList();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(actualAddressList, not(empty()));
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from member_one2many_eager",
                "left outer join address_one2many_eager"
        ));
    }

    @Test
    public void select_address_eager(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address1.getId());

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from address_one2many_eager",
                        "left outer join member_one2many_eager"
                ),
                stringContainsInOrder(
                        "select",
                        "from address_one2many_eager"
                )
        ));
    }
}
```  

`Member` 의 `@OneToMany` 가 `Eager` 로 설정 됐기 때문에, 
`Member` 조회 시점에 `left outer join` 쿼리로 `Address` 까지 함께 조회 된다.  

```sql
select
    member0_.id as id1_12_0_,
    member0_.name as name2_12_0_,
    addresslis1_.member_id as member_i4_0_1_,
    addresslis1_.id as id1_0_1_,
    addresslis1_.id as id1_0_2_,
    addresslis1_.address_detail as address_2_0_2_,
    addresslis1_.address_main as address_3_0_2_,
    addresslis1_.member_id as member_i4_0_2_ 
from
    member_one2many_eager member0_ 
left outer join
    address_one2many_eager addresslis1_ 
        on member0_.id=addresslis1_.member_id 
where
    member0_.id=?
```  

`Address` 를 조회 할때는 `Lazy` 때와 동일하게 `left outer join` 으로 `Member` 도 함께 조회 한다.  


### @ManyToMany
`@ManyToMany` 기본 `Fetch` 전략은 `Lazy` 이다. 

#### Lazy

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_many2many_lazy")
@Table(name = "team_many2many_lazy")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is lazy
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
@Entity(name = "member_many2many_lazy")
@Table(name = "member_many2many_lazy")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToMany(mappedBy = "members")
    @Builder.Default
    private List<Team> teams = new ArrayList<>();
}
```  

부모 엔티티인 `Team` 을 조회할때는 `Team` 만 조회 하고,
자식 엔티티 `Member` 가 실제로 사용될때 `Member` 가 조회 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyLazyFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team1;
    private Team team2;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.team1 = Team.builder()
                .name("team-1")
                .build();
        this.team1.addMember(this.member2);
        this.team1.addMember(this.member2);
        this.entityManager.persist(this.team1);
        this.team2 = Team.builder()
                .name("team-2")
                .build();
        this.team2.addMember(this.member1);
        this.team2.addMember(this.member2);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_team_lazy(CapturedOutput capturedOutput) {
        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team1.getId());
        List<Member> actualMemberList = actualTeam.getMembers();

        // then
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy_members",
                        "inner join member_many2many_lazy"
                ))
        ));
        assertThat(actualMemberList, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy_members",
                        "inner join member_many2many_lazy"
                )
        ));
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        List<Team> actualTeamList = actualMember.getTeams();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2many_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy_members",
                        "inner join team_many2many_lazy"
                ))
        ));
        assertThat(actualTeamList, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2many_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from team_many2many_lazy_members",
                        "inner join team_many2many_lazy"
                )
        ));
    }
}
```  

`Team` 의 `@ManyToMany` 에 `Fetch` 전략이 `Lazy` 이므로 `Team` 만 조회 후, 
`members` 필드가 실제로 사용될때 `member` 테이블 조회가 실행되는데 `@ManyToMany` 의 경우 중간에 매핑 테이블이 존재한다. 
그래서 실제 `Member` 조회를 위해서는 `member` 테이블과 중간 매핑 테이블을 `inner join` 하는 방식으로 조회 된다.  

```sql
# this.entityManager.find(Team.class, this.team1.getId()) 시점에 조회
select
    team0_.id as id1_16_0_,
    team0_.name as name2_16_0_ 
from
    team_many2many_lazy team0_ 
where
    team0_.id=?


# assertThat(actualMemberList, not(empty())) 시점에 조회
select
    members0_.teams_id as teams_id1_17_0_,
    members0_.members_id as members_2_17_0_,
    member1_.id as id1_7_1_,
    member1_.name as name2_7_1_
from
    team_many2many_lazy_members members0_
inner join
    member_many2many_lazy member1_
    on members0_.members_id=member1_.id
where
    members0_.teams_id=?
```  

`Member` 를 조회 할때도 `Team` 을 조회할 때랑 크게 다르지 않다. 
`Member` 조회할 때는 `member` 테이블 조회만 이뤄지고, 
`teams` 필드가 실제 사용되는 시점에 `team` 테이블과 중간 매핑 테이블을 `inner join` 해서 조회 한다.  

```sql
# this.entityManager.find(Member.class, this.member1.getId()) 시점에 조회
select
    member0_.id as id1_7_0_,
    member0_.name as name2_7_0_ 
from
    member_many2many_lazy member0_ 
where
    member0_.id=?


# assertThat(actualTeamList, not(empty())) 시점에 조회
select
    teams0_.members_id as members_2_17_0_,
    teams0_.teams_id as teams_id1_17_0_,
    team1_.id as id1_16_1_,
    team1_.name as name2_16_1_
from
    team_many2many_lazy_members teams0_
inner join
    team_many2many_lazy team1_
    on teams0_.teams_id=team1_.id
where
    teams0_.members_id=?
```  

#### Eager

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "team_many2many_eager")
@Table(name = "team_many2many_eager")
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is lazy
    @ManyToMany(fetch = FetchType.EAGER)
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
@Entity(name = "member_many2many_eager")
@Table(name = "member_many2many_eager")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @ManyToMany(mappedBy = "members")
    @Builder.Default
    private List<Team> teams = new ArrayList<>();
}
```  

부모 엔티티 `Team` 를 조회할때 자식 엔티티 `Member` 까지 모두 함께 조회 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ManyToManyEagerFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member1;
    private Member member2;
    private Team team1;
    private Team team2;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.member1 = Member.builder()
                .name("a")
                .build();
        this.entityManager.persist(this.member1);
        this.member2 = Member.builder()
                .name("b")
                .build();
        this.entityManager.persist(this.member2);
        this.team1 = Team.builder()
                .name("team-1")
                .build();
        this.team1.addMember(this.member2);
        this.team1.addMember(this.member2);
        this.entityManager.persist(this.team1);
        this.team2 = Team.builder()
                .name("team-2")
                .build();
        this.team2.addMember(this.member1);
        this.team2.addMember(this.member2);
        this.entityManager.persist(this.team2);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_team_eager(CapturedOutput capturedOutput) {
        // when
        Team actualTeam = this.entityManager.find(Team.class, this.team1.getId());
        List<Member> actualMemberList = actualTeam.getMembers();

        // then
        assertThat(actualTeam, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from team_many2many_eager",
                "left outer join team_many2many_eager_members",
                "left outer join member_many2many_eager"
        ));
        assertThat(actualMemberList, not(empty()));
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member1.getId());
        List<Team> actualTeamList = actualMember.getTeams();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2many_eager"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from team_many2many_eager_members",
                        "inner join team_many2many_eager"
                )),
                not(stringContainsInOrder(
                        "select",
                        "from team_many2many_eager_members",
                        "inner join member_many2many_eager"
                ))
        ));
        assertThat(actualTeamList, not(empty()));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_many2many_eager"
                ),
                stringContainsInOrder(
                        "select",
                        "from team_many2many_eager_members",
                        "inner join team_many2many_eager"
                ),
                stringContainsInOrder(
                        "select",
                        "from team_many2many_eager_members",
                        "inner join member_many2many_eager"
                )
        ));
    }
}
```  

`Team` 을 조회할 때는 `@ManyToMany` `Fetch` 전략이 `Eager` 이므로, 
`team`, 매핑 테이블, `member` 테이블을 `left outer join` 해서 모두 한번에 조회 한다.  

```sql
select
    team0_.id as id1_15_0_,
    team0_.name as name2_15_0_,
    members1_.teams_id as teams_id1_16_1_,
    member2_.id as members_2_16_1_,
    member2_.id as id1_5_2_,
    member2_.name as name2_5_2_ 
from
    team_many2many_eager team0_ 
left outer join
    team_many2many_eager_members members1_ 
        on team0_.id=members1_.teams_id 
left outer join
    member_many2many_eager member2_ 
        on members1_.members_id=member2_.id 
where
    team0_.id=?
```  

`Member` 를 조회 할때는 먼저 `member` 테이블만 조회 이후, 
`teams` 필드가 실제로 사용될때 `team` 테이블과 매핑 테이블을 `inner join` 해서 조회 하는데 
`member` 와 매핑 테이블을 `inner join` 해서 가져오는 쿼리가 한번 더 수행 된다.  

```sql
# this.entityManager.find(Member.class, this.member1.getId()) 시점에 조회
select
    member0_.id as id1_5_0_,
    member0_.name as name2_5_0_
from
    member_many2many_eager member0_
where
    member0_.id=?
    
    
# assertThat(actualTeamList, not(empty())) 시점에 조회
select
    teams0_.members_id as members_2_16_0_,
    teams0_.teams_id as teams_id1_16_0_,
    team1_.id as id1_15_1_,
    team1_.name as name2_15_1_
from
    team_many2many_eager_members teams0_
inner join
    team_many2many_eager team1_
    on teams0_.teams_id=team1_.id
where
    teams0_.members_id=?

select
    members0_.teams_id as teams_id1_16_0_,
    members0_.members_id as members_2_16_0_,
    member1_.id as id1_5_1_,
    member1_.name as name2_5_1_
from
    team_many2many_eager_members members0_
inner join
    member_many2many_eager member1_
    on members0_.members_id=member1_.id
where
    members0_.teams_id=?
```  


### @OneToOne
`@OneToOne` 의 기본 `Fetch` 전략은 `Eager` 이다. 


### Eager

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2one_eager")
@Table(name = "member_one2one_eager")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @OneToOne
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2one_eager")
@Table(name = "address_one2one_eager")
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

부모 엔티티 `Member` 를 조회할 때, 자식 엔티티인 `Address` 까지 함꼐 조회 한다. 

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneEagerFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address);
        this.member = Member.builder()
                .name("a")
                .address(this.address)
                .build();
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_eager(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from member_one2one_eager",
                "left outer join address_one2one_eager"
        ));
    }

    @Test
    public void select_address_eager(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address.getId());

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from address_one2one_eager",
                "left outer join member_one2one_eager"
        ));
    }
}
```  

`Member` 를 조회 할때 `Member` 엔티티의 `@OneToOne` `Fetch` 전략이 `Eager` 이므로, 
`Address` 와 `left outer join` 을 통해 함께 조회 한다.  

```sql
select
    member0_.id as id1_14_0_,
    member0_.address_id as address_3_14_0_,
    member0_.name as name2_14_0_,
    address1_.id as id1_3_1_,
    address1_.address_detail as address_2_3_1_,
    address1_.address_main as address_3_3_1_ 
from
    member_one2one_eager member0_ 
left outer join
    address_one2one_eager address1_ 
        on member0_.address_id=address1_.id 
where
    member0_.id=?
```  

`Address` 를 조회할때도 동일하게 `Member` 와 `left outer join` 을 통해 한번에 조회 된다.  

```sql
select
    address0_.id as id1_3_0_,
    address0_.address_detail as address_2_3_0_,
    address0_.address_main as address_3_3_0_,
    member1_.id as id1_14_1_,
    member1_.address_id as address_3_14_1_,
    member1_.name as name2_14_1_ 
from
    address_one2one_eager address0_ 
left outer join
    member_one2one_eager member1_ 
        on address0_.id=member1_.address_id 
where
    address0_.id=?
```  


#### Lazy

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2one_lazy")
@Table(name = "member_one2one_lazy")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @OneToOne(fetch = FetchType.LAZY)
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2one_lazy")
@Table(name = "address_one2one_lazy")
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

부모 엔티티 `Member` 를 조회할때 는 `Member` 만 조회하고, 
자식 엔티티 `Address` 가 사용될 때 `Address` 를 조회 한다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneLazyFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address);
        this.member = Member.builder()
                .name("a")
                .address(this.address)
                .build();
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());
        Address actualAddress = actualMember.getAddress();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2one_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from address_one2one_lazy"
                ))
        ));
        assertThat(actualAddress.getAddressMain(), is("서울"));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2one_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from address_one2one_lazy"
                )
        ));
    }

    @Test
    public void select_address_eager(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address.getId());

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from address_one2one_lazy",
                "left outer join member_one2one_lazy"
        ));
    }
}
```  


`Member` 를 조회 할때 `Member` 엔티티의 `@OneToOne` `Fetch` 전략이 `Lazy` 이므로,
`member` 테이블 만 조회하고 
`address` 필드가 실제로 사용될 때 `address` 테이블과 `member` 테이블을 `left outer join` 을해서 조회 한다. 
```sql
# this.entityManager.find(Member.class, this.member.getId()) 시점에 조회 
select
    member0_.id as id1_16_0_,
    member0_.address_id as address_3_16_0_,
    member0_.name as name2_16_0_ 
from
    member_one2one_lazy member0_ 
where
    member0_.id=?


# assertThat(actualAddress.getAddressMain(), is("서울")) 시점에 조회 
select
    address0_.id as id1_5_0_,
    address0_.address_detail as address_2_5_0_,
    address0_.address_main as address_3_5_0_,
    member1_.id as id1_16_1_,
    member1_.address_id as address_3_16_1_,
    member1_.name as name2_16_1_
from
    address_one2one_lazy address0_
left outer join
    member_one2one_lazy member1_
    on address0_.id=member1_.address_id
where
    address0_.id=?
```  

> 위 상황을 보면 `Lazy` 로딩이지만 `@OneToOne` 이라는 연관관계이므로 `Address` 엔티티를 사용할 때 `left outer join` 이 발생 했다. 
> 이는 현재 `Member` 와 `Address` 가 양방향 `1:1` 관계이므로 발생하고 있다. 
> `Address` 를 조회할때 `Member` 의 필드가 있기 때문에, `address` 테이블 만으로는 매핑되는 `member` 의 존재여부를 알 수 없기 때문이다. 
> 즉 양방향 `1:1` 관계가 아니라 단방향 `1:1` 로 수정해주면 `left outer join` 은 발생하지 않는다.  


`Address` 를 조회할때는 `address` 와 `member` 를 `left outer join` 해서 `Member` 까지 함께 조회 한다.  


#### Eager(optional=false)

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2one_false")
@Table(name = "member_one2one_false")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @OneToOne(fetch = FetchType.EAGER, optional = false)
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2one_false")
@Table(name = "address_one2one_false")
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


`@OneToOne` 어노테이션의 속성중 `optional` 기본 값은 `true` 이다.
그러므로 `Eager` 전략일 때 `Member` 를 조회하면 `left outer join` 쿼리가 실행되는데,
이를 `inner join` 으로 변경하는 방법은 `optional` 속성 값을 `false` 로 설정 하면 된다.  

```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneOptionalFalseTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address);
        this.member = Member.builder()
                .name("a")
                .address(this.address)
                .build();
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_eager_innerjoin(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from member_one2one_false",
                "inner join address_one2one_false"
        ));
    }

    @Test
    public void select_address_eager_leftouterjoin(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address.getId());

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), stringContainsInOrder(
                "select",
                "from address_one2one_false",
                "left outer join member_one2one_false"
        ));
    }
}
```  

`Member` 를 조회 할때 `Address` 도 함께 조회 되지만, `optional` 속성이 `false` 이므로 `inner join` 으로 쿼리가 수행된다.  

```sql
select
    member0_.id as id1_15_0_,
    member0_.address_id as address_3_15_0_,
    member0_.name as name2_15_0_,
    address1_.id as id1_4_1_,
    address1_.address_detail as address_2_4_1_,
    address1_.address_main as address_3_4_1_ 
from
    member_one2one_false member0_ 
inner join
    address_one2one_false address1_ 
        on member0_.address_id=address1_.id 
where
    member0_.id=?
```  

`Address` 를 조회 할때는 여전히 `left outer join` 으로 `Address` 와 `Member` 를 한번에 조회 한다.  


#### Lazy(양방향)

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_one2one_bi_lazy")
@Table(name = "member_one2one_bi_lazy")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    // default fetch is eager
    @OneToOne(fetch = FetchType.LAZY)
    private Address address;
}

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "address_one2one_bi_lazy")
@Table(name = "address_one2one_bi_lazy")
public class Address {
    @Id
    @GeneratedValue
    private Long id;
    private String addressMain;
    private String addressDetail;

    @OneToOne(mappedBy = "address", fetch = FetchType.LAZY)
    private Member member;
}
```  

부모 엔티티 `Member` 와 자식 엔티티 `Address` 가 양방향 `1:1` 이면서, 
양방향으로 `Lazy` 를 설정하게 될 경우 실제 동작이 `Lazy` 하게 수행되지 않을 수 있다.  


```java
@DataJpaTest(properties = {
        "spring.jpa.properties.hibernate.show_sql=true",
        "spring.jpa.properties.hibernate.format_sql=false"
})
@ExtendWith({SpringExtension.class, OutputCaptureExtension.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class OneToOneBidirectionalLazyFetchTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Member member;
    private Address address;

    @BeforeEach
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();

        this.address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        this.entityManager.persist(this.address);
        this.member = Member.builder()
                .name("a")
                .address(this.address)
                .build();
        this.entityManager.persist(this.member);
        this.entityManager.flush();
        this.entityManager.clear();
    }

    @Test
    public void select_member_lazy(CapturedOutput capturedOutput) {
        // when
        Member actualMember = this.entityManager.find(Member.class, this.member.getId());
        Address actualAddress = actualMember.getAddress();
        String output = capturedOutput.getOut();

        // then
        assertThat(actualMember, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2one_bi_lazy"
                ),
                not(stringContainsInOrder(
                        "select",
                        "from address_one2one_bi_lazy"
                ))
        ));
        assertThat(actualAddress.getAddressMain(), is("서울"));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from member_one2one_bi_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from address_one2one_bi_lazy"
                )
        ));
    }

    @Test
    public void select_address_eager(CapturedOutput capturedOutput) {
        // when
        Address actualAddress = this.entityManager.find(Address.class, this.address.getId());
        Member actualMember = actualAddress.getMember();

        // then
        assertThat(actualAddress, notNullValue());
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from address_one2one_bi_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from member_one2one_bi_lazy"
                )
        ));
        assertThat(actualMember.getName(), is("a"));
        assertThat(capturedOutput.getOut(), allOf(
                stringContainsInOrder(
                        "select",
                        "from address_one2one_bi_lazy"
                ),
                stringContainsInOrder(
                        "select",
                        "from member_one2one_bi_lazy"
                )
        ));
    }
}
```  

`Member` 를 조회 할때는 먼저 `member` 테이블을 조회 한 후, 
`address` 필드가 사용되는 시점에 `address` 테이블을 조회하기 때문에 기대하는 것처럼 동작한다.  

```sql
# this.entityManager.find(Member.class, this.member.getId()) 시점에 조회 
select
    member0_.id as id1_13_0_,
    member0_.address_id as address_3_13_0_,
    member0_.name as name2_13_0_ 
from
    member_one2one_bi_lazy member0_ 
where
    member0_.id=?


# assertThat(actualAddress.getAddressMain(), is("서울")) 시점에 조회
select
    address0_.id as id1_2_0_,
    address0_.address_detail as address_2_2_0_,
    address0_.address_main as address_3_2_0_
from
    address_one2one_bi_lazy address0_
where
    address0_.id=?
```  

`Fetch` 전략이 `Lazy` 로 설정됐기 때문에, 
기대하는 동작은 연관된 엔티티의 필드가 사용 될때 해당 엔티티 조회 쿼리가 수행되는 것이지만, 
`@OneToOne` 에서 양방향으로 `Lazy` 가 설정된 경우 자식 엔티티인 `Address` 를 조회 할때는 그렇게 동작하지 않는다. 

`Address` 를 조회할 때 `address` 테이블을 조회하는 쿼리, `member` 를 조회하는 쿼리가 각 1번씩 총 2번 수행 된다.  

```sql
# this.entityManager.find(Address.class, this.address.getId()) 시점에 모두 조회
select
    address0_.id as id1_2_0_,
    address0_.address_detail as address_2_2_0_,
    address0_.address_main as address_3_2_0_ 
from
    address_one2one_bi_lazy address0_ 
where
    address0_.id=?

select
    member0_.id as id1_13_0_,
    member0_.address_id as address_3_13_0_,
    member0_.name as name2_13_0_
from
    member_one2one_bi_lazy member0_
where
    member0_.address_id=?
```  

이러한 동작의 원인은 자식 엔티티인 `Address` 는 연관관계의 주인이 아니기 때문에 `Lazy` 로 설정하더라도 `Eager` 과 별 차이 없게 동작하게 된다. 
연관관계의 주인인 `Member` 의 경우 자신의 `member` 테이블 만으로 어떤 `Address` 와 매핑되는지 확인 할 수 있고(`FK` 존재), `null` 여부도 판별 할 수 있다. 
하지만 연관관계 주인이 아닌 `Address` 의 경우 자신의 테이블 `address` 만으로는 어떠한 `Member` 와 매핑되는 지(`FK` 존재 하지 않음), `null` 여부도 판별 할 수 없기 때문에 
`Lazy` 하게 동작 될 수 없다.  

위와 같은 이유로 `@OneToOne` 관계일 경우 양방향으로 `Lazy` 를 설정하기 보다는 연관관계의 주인 쪽에만 `Lazy` 를 설정하는게 좋다. 
혹은 관계 자체가 `@ManyToOne`, `@OneToMany` 로 변경 가능한지도 고려해 볼만 하다.  


---  
## Reference