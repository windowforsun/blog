--- 
layout: single
classes: wide
title: "[Spring 실습] JPA 값 타입"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring JPA 엔티티에서 사용할 수 있는 값 타입의 종류와 특징에 대해 알아보자'
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
toc: true
use_math: true
---  

## JPA 타입 종류
`Spring JPA` 엔티티에서 사용할 수 있는 데이터 타입을 크게 나누면 `엔티티 타입` 과 `값 타입` 이 있다. 
`엔티티 타입`은 말 그대로 클래스에 `@Entity` 어노테이션이 붙은 객체를 의미한다. 
그리고 `값 타입` 은 `int`, `Integer`, `String` 등과 같이 자바 기본 타입 혹은 객체를 의미한다.  

-  엔티티 타입
  - `@Entity` 로 정의하는 객체
  - 데이터에 변경이 있더라도 식별자를 통해 지속적인 추적이 가능
  - 엔티티마다 고유한 생명주기를 갖는다. 
  - 엔티티간 참조가 가능하다. 
- 값 타입
  - 자바 기본 타입 혹은 객체(커스텀 객체 포함)
  - 식별자가 없기 때문에 데이터 변경이 있을 때 추적 불가
  - 값 타입은 값 타입을 포함하고 있는 엔티티의 생명주기에 포함된다. 
    - 스스로 생명주기를 가지지 않고 엔티티가 삭제되면 같이 삭제 된다. 
  - 엔티티간 값 타입을 공유하는 것은 위험하다.
  - 값 타입은 불편 객체로 만드는 것이 안전하다.
    - `setter` 를 만들지 않는 방법으로 생성후 수정이 불가하도록
  

## 값 타입
값 타입으로 사용할 수 있는 종류는 아래와 같다. 
그리고 값 타입은 값 타입을 포함하고 있는 엔티티의 생명주기에 포함된다. 

- 기본값 타입
  - 자바 기본 타입(`int`, `double`, ..)
  - 래퍼 클래스(`Integer`, `Long`, ..)
- 임베디드 타입
  - 사용자가 커스텀하게 정의한 객체 타입
- 컬렉션 타입
  - `Java Collection` 에 `기본값 타입` 혹은 `임베디드 타입` 을 사용

### 테스트 환경 구성
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
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        #show_sql: true # 쿼리 출력
        format_sql: true # 쿼리 이쁘게 출력
```  

### 기본값 타입

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_default")
@Table(name = "member_default")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;
}
```  

`Member` 엔티티는 `id` 라는 식별자를 가지고 고유한 생명주기로 관리되지만, 
값 타입에 해당하는 `name`, `age` 필드는 별도의 식별자와 생명주기가 없이 `Member` 엔티티에 의존한다. 

```sql
create table member_default
(
  id   bigint  not null,
  age  integer not null,
  name varchar(255),
  primary key (id)
);
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
public class DefaultTypeTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();
    }

    @Test
    public void insert_select() {
        // given
        Member member = Member.builder()
                .name("a")
                .age(20)
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
    }
    /*
     insert
     into
         member_default
         (age, name, id)
     values
         (?, ?, ?);
     
     select
         member0_.id as id1_2_0_,
         member0_.age as age2_2_0_,
         member0_.name as name3_2_0_
     from
         member_default member0_
     where
         member0_.id=?;;
     */
}
```  

### 임베디드 타입

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_embedded")
@Table(name = "member_embedded")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;

    @Embedded
    private Address address;
}

@Getter
// 불편 객체를 위해 setter 생성하지 않음
//@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
// equals override 필요
@EqualsAndHashCode
@Embeddable
public class Address {
    private String addressMain;
    private String addressDetail;
}

```  

`임베디드 타입` 은 사용자가 정의한 객체를 엔티티의 필드로 사용하는 것을 의미한다. 
`Member` 엔티티가 `Address` 라는 `임베디드 타입` 을 가지고 있는데, 
이는 `Member` 엔티티에 `addressMain`, `addressDetail` 필드를 그냥 추가하는 것과 테이블 입장에서는 다르지 않다. 
그리고 `임베디드 타입` 의 모든 변수가 `Null` 일 경우 `임베디드 객체` 는 `Null` 이 된다. 

`임베디드 타입` 을 사용하기 위해서는 `임베디드 타입` 클래스 레벨에 `@Embeddable` 어노테이션을 설정해주고, 
엔티티에서 `임베디트 타입` 이 사용되는 필드에는 `@Embedded` 어노테이션을 설정해 준다. 

```sql
create table member_embedded (
   id bigint not null,
    address_detail varchar(255),
    address_main varchar(255),
    age integer not null,
    name varchar(255),
    primary key (id)
);
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
public class EmbeddedTypeTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();
    }

    @Test
    public void insert_select() {
        // given
        Address address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .address(address)
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getAddress().getAddressMain(), is(address.getAddressMain()));
        assertThat(actual.getAddress().getAddressDetail(), is(address.getAddressDetail()));
    }
    /*
    insert
    into
        member_embedded
        (address_detail, address_main, age, name, id)
    values
        (?, ?, ?, ?, ?);

    select
        member0_.id as id1_3_0_,
        member0_.address_detail as address_2_3_0_,
        member0_.address_main as address_3_3_0_,
        member0_.age as age4_3_0_,
        member0_.name as name5_3_0_
    from
        member_embedded member0_
    where
        member0_.id=?;
     */

    @Test
    public void update_embeddedEntity() {
        // given
        Address address = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .address(address)
                .build();
        this.entityManager.persist(member);
        member.setAddress(Address.builder()
                .addressMain("경기")
                .addressDetail("201호")
                .build());
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getAddress().getAddressMain(), is("경기"));
    }
    /*
    insert
    into
        member_embedded
        (address_detail, address_main, age, name, id)
    values
        (?, ?, ?, ?, ?);

    update
        member_embedded
    set
        address_detail=?,
        address_main=?,
        age=?,
        name=?
    where
        id=?;

    select
        member0_.id as id1_3_0_,
        member0_.address_detail as address_2_3_0_,
        member0_.address_main as address_3_3_0_,
        member0_.age as age4_3_0_,
        member0_.name as name5_3_0_
    from
        member_embedded member0_
    where
        member0_.id=?;
     */
}
```  

#### @AttributeOverride
`@AttributeOverride` 는 하나의 엔티티에 동일한 `임베디드 타입` 이 두번 이상 사용 됐을 때 사용할 수 있다. 
만약 해당 어노테이션을 사용하지 않고 한 엔티티에 동일한 `임베디드 타입` 이 중복해서 사용되면 아래와 같은 예외가 발생한다. 

```
Caused by: javax.persistence.PersistenceException: [PersistenceUnit: default] Unable to build Hibernate SessionFactory; nested exception is org.hibernate.MappingException: Repeated column in mapping for entity:
....

Caused by: org.hibernate.MappingException: Repeated column in mapping for entity:
...
```  

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_embedded_override")
@Table(name = "member_embedded_override")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride(name = "addressMain", column = @Column(name = "home_address_main")),
            @AttributeOverride(name = "addressDetail", column = @Column(name = "home_address_detail"))
    })
    private Address homeAddress;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride(name = "addressMain", column = @Column(name = "company_address_main")),
            @AttributeOverride(name = "addressDetail", column = @Column(name = "company_address_detail"))
    })
    private Address companyAddress;
}
```  

`Address` 라는 `임베디드 타입` 이 `Member` 엔티티에 2번 사용되기 때문에 `@AttributeOverride` 어노테이션을 사용해서 해결해 주었다. 
이는 엔티티가 테이블로 매핑될 때 중복되는 필드이름이 생기기 때문에 위 예시처럼 모두 어노테이션을 사용해도 되고 
둘 중 하나의 `Address` 타입 필드에는 사용해주지 않아도 예외는 발생하지 않는다. 

```sql
create table member_embedded_override (
   id bigint not null,
    age integer not null,
    company_address_detail varchar(255),
    company_address_main varchar(255),
    address_detail varchar(255),
    address_main varchar(255),
    name varchar(255),
    primary key (id)
);
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
public class EmbeddedTypeAttributeOverrideTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();
    }

    @Test
    public void insert_select() {
        // given
        Address homeAddress = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Address companyAddress = Address.builder()
                .addressMain("경기도")
                .addressDetail("2101호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .homeAddress(homeAddress)
                .companyAddress(companyAddress)
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getHomeAddress().getAddressMain(), is(homeAddress.getAddressMain()));
        assertThat(actual.getHomeAddress().getAddressDetail(), is(homeAddress.getAddressDetail()));
        assertThat(actual.getCompanyAddress().getAddressMain(), is(companyAddress.getAddressMain()));
        assertThat(actual.getCompanyAddress().getAddressDetail(), is(companyAddress.getAddressDetail()));
    }
    /*
    insert
    into
        member_embedded_override
        (age, company_address_detail, company_address_main, address_detail, address_main, name, id)
    values
        (?, ?, ?, ?, ?, ?, ?);
        

    select
        member0_.id as id1_5_0_,
        member0_.age as age2_5_0_,
        member0_.company_address_detail as company_3_5_0_,
        member0_.company_address_main as company_4_5_0_,
        member0_.address_detail as address_5_5_0_,
        member0_.address_main as address_6_5_0_,
        member0_.name as name7_5_0_
    from
        member_embedded_override member0_
    where
        member0_.id=?;
     */
}
```  

### 컬렉션 타입
`컬렉션 타입` 은 하나 이상의 값 타입을 저장 할 때 사용한다. 
`List`, `Set`, .. 과 같은 `Java Collection` 타입은 모두 사용 가능하고 사용하는 변수에 `@ElementCollection` 어노테이션 선언이 필요하다. 
`컬렉션 타입` 은 기본적으로 별도의 테이블로 관리된다. 
기본 매핑 테이블 명은 `엔티티명_컬렉션타입명` 이 되는데, 
커스텀한 설정이 필요할 경우 `@CollectionTable` 어노테이션을 사용해서 지정이 가능하다. 
그리고 `컬렉션 테이블` 의 `PK` 는 엔티티의 `PK` 를 그대로 사용한다.  

데이터 추가, 삭제, 변경시 특징
- `컬렉션 타입` 값이 변경되면 모든 `컬렉션 타입` 값을 삭제 한 뒤 새롭게 추가한다. 
- `컬렉션 타입` 값 설정은 컬렉션의 개수만큼 `INSERT` 쿼리가 수행된다. 
- `컬렉션 타입` 조작에 따른 성능 이슈가 예상될때는 `@OneToManay` 구조를 사용 할 수 있다. 
- `컬렉션 타입` 의 조회 기본 전략은 `LAZY` 로 `컬렉션 타입` 에 조회가 실제로 일어날 때 조회 쿼리가 수행된다. 


#### 값 컬렉션 타입

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_value_coll")
@Table(name = "member_value_coll")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;

    @ElementCollection
    @CollectionTable(name = "hobbies", joinColumns = @JoinColumn(name = "id"))
    private List<String> hobbies;
}
```  

`Member` 엔티티는 `hobbies` 라는 문자열 `값 컬렉션 타입` 을 사용하고 있다. 

```sql
create table member_value_coll (
   id bigint not null,
    age integer not null,
    name varchar(255),
    primary key (id)
);

create table hobbies (
	id bigint not null,
	hobbies varchar(255)
);
alter table hobbies
	add constraint FKm6i93ol5vdwuc0ovwnufkdms1
		foreign key (id)
			references member_value_coll;
			
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
public class ValueCollectionTypeTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();
    }

    @Test
    public void insert_select() {
        // given
        Member member = Member.builder()
                .name("a")
                .age(20)
                .hobbies(Arrays.asList("음악", "영화"))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getHobbies(), contains("음악", "영화"));
    }
    /*
    insert
    into
        member_value_coll
        (age, name, id)
    values
        (?, ?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    select
        member0_.id as id1_6_0_,
        member0_.age as age2_6_0_,
        member0_.name as name3_6_0_
    from
        member_value_coll member0_
    where
        member0_.id=?;

    select
        hobbies0_.id as id1_1_0_,
        hobbies0_.hobbies as hobbies2_1_0_
    from
        hobbies hobbies0_
    where
        hobbies0_.id=?;
     */

    @Test
    public void remove_valueCollection() {
        // given
        Member member = Member.builder()
                .name("a")
                .age(20)
                .hobbies(Arrays.asList("음악", "영화", "축구"))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getHobbies().remove("음악");
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getHobbies(), contains("영화", "축구"));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        hobbies
    where
        id=?;

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);
     */

    @Test
    public void add_valueCollection() {
        // given
        Member member = Member.builder()
                .name("a")
                .age(20)
                .hobbies(Arrays.asList("음악", "영화"))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getHobbies().add("축구");
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getHobbies(), contains("음악", "영화", "축구"));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        hobbies
    where
        id=?;

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);
     */

    @Test
    public void update_valueCollection() {
        // given
        Member member = Member.builder()
                .name("a")
                .age(20)
                .hobbies(Arrays.asList("음악", "영화"))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getHobbies().set(0, "독서");
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getHobbies(), contains("독서", "영화"));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        hobbies
    where
        id=?;

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);

    insert
    into
        hobbies
        (id, hobbies)
    values
        (?, ?);
     */
}
```  

#### 임베디드 컬렉션 타입

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "member_embedded_coll")
@Table(name = "member_embedded_coll")
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;

    @ElementCollection
    @CollectionTable(name = "addresses", joinColumns = @JoinColumn(name = "id"))
    private List<Address> addresses;
}

@Getter
// 불편 객체를 위해 setter 생성하지 않음
//@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
// equals override 필요
@EqualsAndHashCode
@Embeddable
public class Address {  
    private String addressMain;
    private String addressDetail;
}
```  

`Member` 엔티티는 `address` 라는 문자열 `임베디드 컬렉션 타입` 을 사용하고 있다.


```sql
create table member_embedded_coll (
   id bigint not null,
    age integer not null,
    name varchar(255),
    primary key (id)
);

create table addresses (
   id bigint not null,
   address_detail varchar(255),
   address_main varchar(255)
);
alter table addresses
	add constraint FK4lbt35e1qkf5ssfj7jej3lqwf
		foreign key (id)
			references member_embedded_coll;
```  

```java
@DataJpaTest
@ExtendWith(SpringExtension.class)
public class EmbeddedCollectionTypeTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;

    @BeforeEach
    public void setUp() {
        this.entityManager = testEntityManager.getEntityManager();
    }

    @Test
    public void insert_select() {
        // given
        Address address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Address address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .addresses(Arrays.asList(address1, address2))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getAddresses().get(0).getAddressMain(), is(address1.getAddressMain()));
        assertThat(actual.getAddresses().get(0).getAddressDetail(), is(address1.getAddressDetail()));
        assertThat(actual.getAddresses().get(1).getAddressMain(), is(address2.getAddressMain()));
        assertThat(actual.getAddresses().get(1).getAddressDetail(), is(address2.getAddressDetail()));
    }
    /*
    insert
    into
        member_embedded_coll
        (age, name, id)
    values
        (?, ?, ?);

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);

    select
        member0_.id as id1_4_0_,
        member0_.age as age2_4_0_,
        member0_.name as name3_4_0_
    from
        member_embedded_coll member0_
    where
        member0_.id=?;

    select
        addresses0_.id as id1_0_0_,
        addresses0_.address_detail as address_2_0_0_,
        addresses0_.address_main as address_3_0_0_
    from
        addresses addresses0_
    where
        addresses0_.id=?;
     */

   @Test
    public void remove_embeddedCollection() {
        // given
        Address address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Address address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .addresses(Arrays.asList(address1, address2))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getAddresses().remove(address1);
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getAddresses().get(0).getAddressMain(), is(address2.getAddressMain()));
        assertThat(actual.getAddresses().get(0).getAddressDetail(), is(address2.getAddressDetail()));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        addresses
    where
        id=?;

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);
     */

    @Test
    public void add_embeddedCollection() {
        // given
        Address address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Address address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .addresses(Arrays.asList(address1, address2))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getAddresses().add(Address.builder()
                .addressMain("강원")
                .addressDetail("401호")
                .build());
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getAddresses().get(2).getAddressMain(), is("강원"));
        assertThat(actual.getAddresses().get(2).getAddressDetail(), is("401호"));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        addresses
    where
        id=?;

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);
     */

    @Test
    public void update_embeddedCollection() {
        // given
        Address address1 = Address.builder()
                .addressMain("서울")
                .addressDetail("201호")
                .build();
        Address address2 = Address.builder()
                .addressMain("경기")
                .addressDetail("301호")
                .build();
        Member member = Member.builder()
                .name("a")
                .age(20)
                .addresses(Arrays.asList(address1, address2))
                .build();
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();
        member = this.entityManager.find(Member.class, member.getId());
        member.getAddresses().set(0, Address.builder()
                .addressMain("강원")
                .addressDetail("401호")
                .build());
        this.entityManager.persist(member);
        this.entityManager.flush();
        this.entityManager.clear();

        // when
        Member actual = this.entityManager.find(Member.class, member.getId());

        // then
        assertThat(actual.getName(), is(member.getName()));
        assertThat(actual.getAge(), is(member.getAge()));
        assertThat(actual.getAddresses().get(0).getAddressMain(), is("강원"));
        assertThat(actual.getAddresses().get(0).getAddressDetail(), is("401호"));
        assertThat(actual.getAddresses().get(1).getAddressMain(), is(address2.getAddressMain()));
        assertThat(actual.getAddresses().get(1).getAddressDetail(), is(address2.getAddressDetail()));
    }
    /*
    초기 INSERT, SELECT 문 제외

    delete
    from
        addresses
    where
        id=?;

    insert
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);

    insert 
    into
        addresses
        (id, address_detail, address_main)
    values
        (?, ?, ?);
     */
}
```  


---  
## Reference