--- 
layout: single
classes: wide
title: "[Spring 개념] JPA Architecture 와 Persistence Context"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Java ORM 표준 JPA 의 구조와 영속성 컨텍스트에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - JPA
    - Persistence Context
    - Dirty Checking
    - EntityManager
toc: true
use_math: true
---  

## JPA Architecture
`JPA`(`Java Persistence API`) 는 `Java` 객체를 관계형 데이터베이스에 매핑하기 위한 `Java` 표준을 의미한다. 
`Java` 객체를 데이터베이스 테이블에 매핑하는 것을 `ORM`(`Object-Replicational Mapping`) 이라고 한다. 
`JPA` 는 `Java` 에서 `ORM` 동작을 수행할 수 있는 하나의 접근법이다. 
`JPA` 를 사용해서 `Java` 객체와 관계형 데이터베이스의 매핑, 저장, 업데이트, 검색 등의 동작을 수행할 수 있다. 
`JPA` 스펙에 맞춰 구현된 사용가능한 라이브러리는 `Hibernate`, `EclipseLink`, `Apache OpenJPA` 등이 있다.  
그리고 `JPA` 는 비지니스 레이어에서 생성되는 엔티티를 관계 엔티티로 저장하고 관리하는 역할을 수행한다. 

`JPA` 구조와 `javax.persistence` 패키지 구성을 클래스 레벨로 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept_jpa_1.png)  

### Persistence 
`Persistence` 클래스는 `EntityManagerFactory` 인스턴스를 생성하는 정적 메소드를 가지고 있는 클래스이다. 
`Persistence` 의 `createEntityManagerFactory()` 메소드를 사용하면 `EntityManagerFactory` 의 인스턴스를 리턴 받을 수 있다. 

### EntityManagerFactory
`EntityManagerFactory` 인터페이스는 `EntityManager` 의 `Factory` 역할을 수행한다. 
하나 이상의 필요에 따라 `EntityManager` 인스턴스를 생성할 수 있다. 
`EntityManagerFactory` 는 애플리케이션이 시작하면 `PersistenceUnit` 의 설정값에 맞춰 생성된다. 
`EntityManagerFactory` 는 연결되는 하나의 데이터베이스와 매칭되는 구조로 생성되기 떄문에, 
`Connection Pool` 을 포함한 여러 작업이 인스턴스 생성과정에 포함된다. 
그러므로 생성 비용이 매우 크기때문에 한 번 생성후 애플리케이션에서 재사용하게 된다. 
`DataSource` 마다 하나의 `EntityManagerFactory` 가 생성된다고 생각할 수 있다. 
그리고 여러 스레드에서 사용이 가능해야 하기때문에 `Thread-safe` 하다.  

### EntityManager
`EntityManager` 는 `Entity` 에 대한 영속성관련 동작을 실제로 수행하면서 데이터베이스 `Connection` 과 관계가 있는 인터페이스이다. 
그리고 영속성 관련 인스턴스(`Persistence Context`) 를 생성하고, 삭제하는 역할을 수행한다. 
데이터베이스와 매핑되는 `Entity` 에 대해 `Persistence Context` 와 `Connection` 를 상황에 따라 적절하 상호동작하게 된다. 
`Entity` 의 `PrimaryKey` 를 사용해서 생성, 수정, 삭제, 조회 등의 동작을 수행하며 이를 관리한다. 
`EntityManager` 는 `Persistence Context` 를 사용해서 `Entity` 를 관리하다가 데이터베이스의 커넥션이 필요한 시점이 되면, 
`Query` 인터페이스를 사용해서 데이터베이스와 동기화를 수행한다. 
그리고 `EntityTransaction` 을 사용해서 트랜잭션 관련 동작을 수행한다.  
`EntityManagerFactory` 와 비교했을 때 `EntityManager` 는 생성비용이 거의 들지 많지만, 
`Thread-unsafe` 하기 때문에 스레드간 공유되지 않도록 주의가 필요하다. 
`EntityManager` 에서 수행하는 동작들은 `Transaction` 안에서 수행되야 하고, 그렇지 않다면 예외가 발생한다.   

### Entity
`Entity` 는 데이터베이스의 테이블과 매핑되는 클래스이면서, 
`Persistence Context` 를 사용해서 `EntityManager` 의 관리 대상이기도 하다. 

### EntityTransaction
`EntityTransaction` 은 `EntityManager` 와 1:1 관계를 가지면서, 
`EntityManager` 에서 트랜잭션 관련 동작을 수행할 수 있도록 한다.  

### Query
`Query` 는 `EntityManager` 에서 `Entity` 에 대한 쿼리를 수행하는 역할을 수행한다. 


## 클래스 관계
앞서 설명한 `javax.persistence` 패키지를 구성하는 주요 클래스와 인터페이스의 관계도를 그리면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept_jpa_2.png)  

- `EntityManagerFactory` 와 `EntityManager` 는 1:N 관계를 갖는다. 
`EntityManagerFactory` 에서 필요에 따라 `EntityManager` 를 생성할 수 있다. 
- `EntityManager` 와 `EntityTransaction` 은 1:1 관계를 갖는다. 
`EntityManager` 에서는 하나의 `EntityTransaction` 을 생성해서 사용한다. 
- `EntityManager` 와 `Query` 는 1:N 관계를 갖는다. 
`EntityManager` 에서는 여러 `Query` 인스턴스를 사용해서 쿼리를 동작을 수행한다. 
- `EntityManager` 와 `Entity` 는 1:N 관계를 갖는다. 
`EntityManager` 에서는 여러 `Entity` 에 대해 관리를 수행할 수 있다. 

## Persistence Context
`Persistence Context`(영속성 컨텍스트) 는 `Entity` 를 영구 저장하는 논리적인 공간이면서 개념이다. 
`EntityManager` 와 `Persistence Context` 가 1:1 관계는 갖는 경우도 있지만, 
트랜잭션, 스레드를 기준으로 여러 `EntityManager` 가 `Persistence Context` 를 공유해서 사용하는 경우도 있다. 

### 생명주기
`Persistence Context` 의 생명주기에는 `New(Transient)`, `Managed`, `Detached`, `Removed` 가 있는데 
전체적인 흐름은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept_jpa_3.png)  

- `New(Transient)`(비영속) : `Entity` 에 해당하는 객체가 인스턴스로 생성만 된 상태로 아직 `Persistence Context` 와는 관련이 없는 상태이다. 

    ```java    
    NoAutoIncrement entity = new NoAutoIncrement();
    entity.setId(11);
    entity.setNum(111);
    entity.setStr("str111");
    ```  

- `Managed`(영속) : `Entity` 가 `Persistence Context` 에 저장된 상태로 `EntityManager` 에 의해 관리 된다. 
`persist()` 메소드를 호출하거나 `find()`, `JPQL` 로 `Entity` 를 조회한 상태를 의미한다. 
`Persistence Context` 에 저장된 상태이지 아직 데이터베이스에는 저장및 관련 쿼리는 수행되지 않았다. 
이후 `flush()` 혹은 `commit()` 이 호출되면 `Persistence Context` 에 있는 `Entity` 정보가 데이터베이스에 반영된다. 

    ```java
    // persist()
    entityManager.persist(entity);
  
    // find()
    NoAutoIncrement entity = entityManager.find(NoAutoIncrement.class, 11);
  
    // JPQL
    String jpql = "SELECT nai FROM NoAutoIncrement nai WHERE nai.id = :id";
    NoAutoIncrement entity = this.entityManager
            .createQuery(jpql, NoAutoIncrement.class)
            .setParameter("id", 11)
            .getSingleResult();
    ```  
  
- `Detached`(준영속) : `Persistence Context` 에 저장돼 있다가 분리된 상태로, `EntityManager` 의 관리 대상에서 벗어난 `Entity` 를 의미한다. 
`detach()` 메소드 호출로 `Detached` 상태를 만들 수 있다. 

    ```java
    entityManager.detach(entity);
    ```  

- `Removed`(삭제) : `Persistence Context` 와 데이터베이스에서도 제거되는 상태이다. 
`remove()` 메소드 호출로 `Entity` 를 삭제할 수 있다. 

    ```java
    entityManager.remove(entity);
    ```  

### 1차 캐시
`Persistence Context` 에는 1차 캐시(`first level cache`) 라는 것이 존재한다. 
이는 `Map` 구조로 `Entity` 의 `Id` 를 키로 가지고 `Entity` 를 값으로 갖는다.  

@Id|Entity
---|---
11|`Entity` 인스턴스
...|...  

`persist()`, `find()`, `JPQL` 등으로 `Entity` 가 `Persistence Context` 에 한번 저장되어 관리대상이 되면, 
유지되는 범위에 한해서 저장된 `Entity` 는 이후 조회, 업데이트 동작에 있어서 실제 데이터베이스와 쿼리를 수행하지 않고 `Persistence Context` 에 해당하는 인스턴스만 수정한다.  

간단한 예로 `find()` 메소드로 찾고자하는 `Entity` 가 `Persistence Context`에 존재하지 않으면, 
`SELECT` 쿼리를 통해 데이터베이스에서 가져와 `Persistence Context` 에 저장해서 `Managed` 상태로 만든다. 
그리고 이후 동일한 `Entity` 를 `find()` 메소드로 찾으면 데이터베이스를 통하지 않고 바로 `Persistence Context` 에서 해당하는 `Entity` 를 반환 한다. 

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class FirstLevelCacheTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();
    }

    @Test
    public void persist로_Entity를_영속상태로만들고_find하면_Select쿼리는수행되지않는다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);

        // when
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual.getId(), is(entity.getId()));
        assertThat(actual.getNum(), is(entity.getNum()));
        assertThat(actual.getStr(), is(entity.getStr()));
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: select"
        )));
    }

    @Test
    public void 데이터베이스의_Entity를_2번find하면_Select쿼리는_1번만수행된다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        String query = "insert into no_auto_increment(num, str, id) values(?, ?, ?)";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getNum());
        psmt.setString(2, entity.getStr());
        psmt.setInt(3, entity.getId());
        psmt.execute();

        // when
        this.entityManager.find(NoAutoIncrement.class, entity.getId());
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual.getId(), is(entity.getId()));
        assertThat(actual.getNum(), is(entity.getNum()));
        assertThat(actual.getStr(), is(entity.getStr()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Hibernate: select"),
                not(stringContainsInOrder("Hibernate: select", "Hibernate: select"))
        ));
    }

    @Test
    public void 데이터베이스의_Entity를_JPQL조회하고_find하면_Select쿼리는_1번만수행된다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        String query = "insert into no_auto_increment(num, str, id) values(?, ?, ?)";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getNum());
        psmt.setString(2, entity.getStr());
        psmt.setInt(3, entity.getId());
        psmt.execute();

        // when
        this.entityManager.createQuery("select nai from NoAutoIncrement nai where nai.id = :id")
                .setParameter("id", entity.getId())
                .getSingleResult();
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual.getId(), is(entity.getId()));
        assertThat(actual.getNum(), is(entity.getNum()));
        assertThat(actual.getStr(), is(entity.getStr()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Hibernate: select"),
                not(stringContainsInOrder("Hibernate: select", "Hibernate: select"))
        ));
    }

    @Test
    public void persist_Entity를_수정하고_find하면_수정된Entity를리턴하고_쿼리는수행되지않는다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        entity.setNum(222);
        entity.setStr("str222");

        // when
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual.getId(), is(entity.getId()));
        assertThat(actual.getNum(), is(entity.getNum()));
        assertThat(actual.getStr(), is(entity.getStr()));
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: select"
        )));
    }
}
```  

### 동일성
`Persistence Context` 에서 같은 키로 반환하는 `Entity` 의 인스턴스는 동일성을 보장한다. 
이 또한 1차 캐시를 사용하기 때문에 가능하다. 
반복적인 읽기 동작에 대해서 애플리케이션 수준에서 `REPEATABLE READ` 격리 수준을 제공한다.  

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.format_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class IdentityTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();

        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        String query = "insert into no_auto_increment(num, str, id) values(?, ?, ?)";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getNum());
        psmt.setString(2, entity.getStr());
        psmt.setInt(3, entity.getId());
        psmt.execute();
    }

    @Test
    public void find두번하면_두인스턴스는같다() {
        // given
        int entityId = 11;
        NoAutoIncrement expected = this.entityManager.find(NoAutoIncrement.class, entityId);

        // when
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entityId);

        // then
        assertThat(actual, is(expected));
    }

    @Test
    public void JPQL조회두번하면_두인스턴스는같다() {
        // given
        int entityId = 11;
        String jpql = "select nai from NoAutoIncrement nai where nai.id = :id";
        NoAutoIncrement expected = this.entityManager.createQuery(jpql, NoAutoIncrement.class)
                .setParameter("id", entityId)
                .getSingleResult();

        // when
        NoAutoIncrement actual = this.entityManager.createQuery(jpql, NoAutoIncrement.class)
                .setParameter("id", entityId)
                .getSingleResult();

        // then
        assertThat(actual, is(expected));
    }

    @Test
    public void find한_Entity와_JPQL조회한_Entity의인스턴스는같다() {
        // given
        int entityId = 11;
        String jpql = "select nai from NoAutoIncrement nai where nai.id = :id";
        NoAutoIncrement expected = this.entityManager.find(NoAutoIncrement.class, entityId);

        // when
        NoAutoIncrement actual = this.entityManager.createQuery(jpql, NoAutoIncrement.class)
                .setParameter("id", entityId)
                .getSingleResult();

        // then
        assertThat(actual, is(expected));
    }
}
```  


### 쓰기 지연
`persist()` 메소드를 사용해서 `Entity` 를 `Persistence Context` 에 추가하면 바로 데이터베이스에 `INSERT` 쿼리가 수행되지 않는다. 
`EntityManager` 는 이를 쿼리 저장소라는 곳에 추가된 `Entity` 의 `INSERT` 쿼리을 저장해 놓게 된다. 
그리고 `commit()`, `flush()` 가 호출될때 저장해 둔 `INSERT` 쿼리를 데이터베이스에 보내 실제로 저장한다.  

<!-- 이러한 쓰기 지연을 사용하게 되면 추가되는 `Entity` 가 있을 때마다 `INSERT` 쿼리를 수행하는 것이 아니라, -->
<!-- 모아 뒀다가 한번에 `INSERT` 쿼리를 데이터베이스로 보내 수행하기 때문에 성능을 높일 수 있다. -->

`EntityManager` 에서 `commit()` 메소드를 수행하게 되면, 
내부에서 먼저 `flush()` 를 호출해서 `Persistence Context` 의 내용을 데이터베이스의 동기화하는 작업을 수행한다. 
그리고 작업이 완료되면 최종적으로 트랜잭션 `commit` 을 수행해서 데이터베이스에 영구적으로 반영한다.  

`hibernate.jdbc.batch_size` 의 키 값으로 한번에 처리할 `INSERT` 쿼리의 크기를 지정할 수 있다.   

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class TransactionWriteBehindTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();
    }

    @Test
    public void persist_Database에는존재하지않는다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);

        // when
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        int actual = rs.getInt(1);

        // then
        assertThat(actual, is(0));
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: insert"
        )));
    }

    @Test
    public void persist_Commit_Database에존재한다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();

        // when
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        int actual = rs.getInt(1);

        // then
        assertThat(actual, is(1));
        assertThat(this.capture.getOut(), containsString(
                "Hibernate: insert"
        ));
    }
}
```  


### Dirty Checking
변경 감지(`Dirty Checking`)은 영속 상태인 `Entity` 의 변경을 감지하고 이를 데이터베이스에 반영하는 것을 의미한다. 
`Persistence Context` 는 최초로 1차 캐시에 저장될 때의 `Entity` 상태를 복사한 스냅샨을 관리한다. 
그리고 `flush()` 가 수행될 때 1차 캐시에 존재하는 `Entiy` 와 스냅샷을 비교해서 변경된 `Entity` 라면 전체 필드에 대한 `UPDATE` 쿼리를 수행한다.  

변경된 `Entity` 에 대해서 모든 필드에 대한 `UPDATE` 쿼리를 수행함으로써, 
애플리케이션 로딩 시점에 미리 생성한 `UPDATE` 쿼리를 재사용 할 수 있고 데이터베이스에 동일한 쿼리를 보낼때 데이터베이스에서도 쿼리 캐싱을 사용할 수 있기 때문에 성능적으로 이점이 있다. 
만약 전체 필드에 대한 `UPDATE` 쿼리가 아닌 변경된 필드에만 `UPDATE` 쿼리를 수행하고 싶다면 `@DynamicUpdate` 어노테이션을 사용해서 가능하다.  

`Dirty Checking` 은 영속 상태인 `Entity` 에 대해서만 변경을 감지하기 때문에 `detach()` 된 `Entity` 의 변경은 감지하지 못한다.  

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class DirtyCheckingTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();
    }

    @Test
    public void 영속Entity를_수정하고_Commit하면_데이터베이스에반영된다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        entity.setNum(222);
        entity.setStr("str222");
        this.entityManager.getTransaction().commit();

        // when
        String query = "select * from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();

        // then
        assertThat(rs.getInt("id"), is(entity.getId()));
        assertThat(rs.getInt("num"), is(entity.getNum()));
        assertThat(rs.getString("str"), is(entity.getStr()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert",
                "Hibernate: update no_auto_increment set num=?, str=? where id=?"
        ));
    }

    @Test
    public void 준영속Entity를_수정하고_Commit하면_데이터베이스에반영되지않는다() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();
        this.entityManager.getTransaction().begin();
        this.entityManager.detach(entity);
        entity.setNum(222);
        entity.setStr("str222");
        this.entityManager.getTransaction().commit();

        // when
        String query = "select * from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();

        // then
        assertThat(rs.getInt("id"), is(entity.getId()));
        assertThat(rs.getInt("num"), not(entity.getNum()));
        assertThat(rs.getString("str"), not(entity.getStr()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Hibernate: insert"),
                not(containsString("Hibernate: update"))
        ));
    }

    @Test
    public void DynamicUpdate선언된Entity에서_특정필드만수정하고_Commit하면_특정필드Update쿼리수행한다() throws Exception {
        // given
        DynamicUpdateNoAutoIncrement entity = new DynamicUpdateNoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        entity.setNum(222);
        this.entityManager.getTransaction().commit();

        // when
        String query = "select * from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();

        // then
        assertThat(rs.getInt("id"), is(entity.getId()));
        assertThat(rs.getInt("num"), is(entity.getNum()));
        assertThat(rs.getString("str"), is(entity.getStr()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert",
                "Hibernate: update no_auto_increment set num=? where id=?"
        ));
    }
}

```  


### remove
`EntityManager` 를 사용해서 `remove()` 메소드를 호출하면 `Persistence Context` 에서 `Entity` 를 삭제하고, 
쿼리 저장소에 `DELETE` 쿼리를 저장하게 된다. 
그리고 `flush()` 시점에 데이터베이스로 `DELETE` 쿼리가 전송된다. 

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.format_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class RemoveTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();

        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        String query = "insert into no_auto_increment(num, str, id) values(?, ?, ?)";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getNum());
        psmt.setString(2, entity.getStr());
        psmt.setInt(3, entity.getId());
        psmt.execute();
    }

    @Test
    public void 데이터베이스Entity_remove하면_PersistenceContext에서_삭제된다() {
        // given
        int entityId = 11;
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, entityId);
        this.entityManager.remove(entity);

        // when
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entityId);

        // then
        assertThat(actual, nullValue());
    }

    @Test
    public void 데이터베이스Entity_remove하면_데이터베이스에서_삭제되지않는다() throws Exception {
        // given
        int entityId = 11;
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, entityId);
        this.entityManager.remove(entity);

        // when
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entityId);
        ResultSet rs = psmt.executeQuery();
        rs.next();
        int actual = rs.getInt(1);

        // then
        assertThat(actual, is(1));
    }

    @Test
    public void 데이터베이스Entity_remove하고_commit하면_데이터베이스에서_삭제된다() throws Exception {
        // given
        int entityId = 11;
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, entityId);
        this.entityManager.remove(entity);
        this.entityManager.getTransaction().commit();

        // when
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entityId);
        ResultSet rs = psmt.executeQuery();
        rs.next();
        int actual = rs.getInt(1);

        // then
        assertThat(actual, is(0));
    }
}
```  


<!-- ## Persistence Context 특징 ????????????????? -->
### flush
앞서 설명한 것처럼 `flush()` 는 `Persistence Context` 의 내용과 데이터베이스의 동기화를 맞추는 역할을 수행한다. 
`flush()` 가 호출 됨으로써 `INSERT`, `UPDATE`, `DELETE` 쿼리가 데이터베이스로 전송된다. 
`flush()` 의 동작과정은 아래와 같다. 
- `Persistence Context` 의 `Entity` 에 대해서 `Dirty Checking` 동작을 수행한다. 
- 수정된 `Entity` 에 대해서 `UPDATE` 쿼리를 쿼리 저장소에 등록한다. 
- 쿼리 저장소에 저장된 쿼리를 데이터베이스에 보낸다. 
이 시점에 트랜잭션 `commit` 이 수행되지는 않는다. 

`flush()` 가 호출되면 쿼리 저장소의 쿼리만 데이터베이스에 전송되고, 
1차 캐시는 그대로 남아있다. 

`flush()` 를 호출하는 방법으로는 아래와 같은 것들이 있다. 
- `EntityManager` 의 `flush()` 메소드 호출
- 트랜잭션 `commit()` 호출할때 자동으로 `flush()` 호출
- `JPQL` 쿼리 수행 전 `flush()` 호출

`EntityManager` 의 `setFlushMode()` 메소드를 사용해서, 
자동으로 `flush()` 실행 시점을 옵션으로 정할 수 있는데 그 내용은 아래와 같다. 
- `FlushModeType.AUTO` : 기본 값으로 `commit` 이나 쿼리 실행할때 호출된다.  
- `FlushModeType.COMMIT` : `commit` 을 수행할 때만 호출된다. 

이렇게 `flush()` 는 단지 `Persistence Context` 의 내용과 데이터베이스를 동기화 하는 작업을 수행하는 동작이다. 
이는 트랜잭션이 필수적인데, 트랜잭션는 하나의 작업 단위이기 때문에 매번 데이터베이스와 동기화를 맞추는 것이 아닌 
트랜잭션 단위로 데이터베이스와 동기화를 맞추는 매커니즘을 사용한다.  

```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class FlushTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    private Connection connection;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.entityManager = this.testEntityManager.getEntityManager();
        this.connection = this.dataSource.getConnection();

        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(111);
        entity.setNum(1111);
        entity.setStr("str1111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();
        this.entityManager.getTransaction().begin();
    }

    @Test
    public void 새로운Entity추가하고_flush호출하면_Insert쿼리실행() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);

        // when
        this.entityManager.flush();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert", "Hibernate: insert"
        ));
    }

    @Test
    public void 새로운Entity추가하고_flush호출하면_데이터베이스에는존재하지않음() throws Exception {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);

        // when
        this.entityManager.flush();

        // then
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        assertThat(rs.getInt(1), is(0));
    }

    @Test
    public void 기존Entity수정하고_flush호출하면_Update쿼리실행() {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        entity.setNum(2222);

        // when
        this.entityManager.flush();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert", "Hibernate: update"
        ));
    }

    @Test
    public void 기존Entity수정하고_flush호출하면_데이터베이스에는반영되지않음() throws Exception {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        entity.setNum(2222);

        // when
        this.entityManager.flush();

        // then
        String query = "select * from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        assertThat(rs.getInt("num"), not(entity.getNum()));
    }

    @Test
    public void Entity삭제하고_flush호출하면_Delete쿼리수행() {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        this.entityManager.remove(entity);

        // when
        this.entityManager.flush();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert", "Hibernate: delete"
        ));
    }

    @Test
    public void Entity삭제하고_flush호출하면_데이터베이스에는삭제되지않음() throws Exception {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        this.entityManager.remove(entity);

        // when
        this.entityManager.flush();

        // then
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        assertThat(rs.getInt(1), is(1));
    }

    @Test
    public void Entity삭제하고_JPQL수행하면_flush호출되면서_Delete쿼리수행() {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        this.entityManager.remove(entity);

        // when
        this.entityManager
                .createQuery("select nai from NoAutoIncrement nai")
                .getResultList();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert", "Hibernate: delete", "Hibernate: select"
        ));
    }

    @Test
    public void Entity삭제하고_JPQL수행하면_flush호출되면서_데이터베이스에는삭제되지않음() throws Exception {
        // given
        NoAutoIncrement entity = this.entityManager.find(NoAutoIncrement.class, 111);
        this.entityManager.remove(entity);

        // when
        this.entityManager
                .createQuery("select nai from NoAutoIncrement nai")
                .getResultList();

        // then
        String query = "select count(1) from no_auto_increment where id = ?";
        PreparedStatement psmt = this.connection.prepareStatement(query);
        psmt.setInt(1, entity.getId());
        ResultSet rs = psmt.executeQuery();
        rs.next();
        assertThat(rs.getInt(1), is(1));
    }
}
```  


### detached
`EntityManager` 에서 `detach()` 메소드를 사용하면 `Entity` 를 준영속 상태로 만들 수 있다. 
준영속 상태가 되면 `Persistence Context` 의 1차 캐시, 쿼리 저장소에서 해당 `Entity` 의 내용을 모두 지우게 되고, 
더이상 `EntityManager` 에의해 관리되지 않는 상태가 된다.  

준영속 상태는 아래와 같은 메소드를 사용해서 만들 수 있다. 
- `detach()` : 특정 `Entity` 를 준영속 상태로 만든다. 
- `clear()` : `EntityManager` 에서 사용하는 `Persistence Context` 에 있는 모든 `Entity` 를 준영속 상태로 만든다. 

준영속 상태는 비영속 상태와 거의 비슷한 상태이지만, 
차이점이 있다면 비영속은 식별자(`@Id`)가 없을 수도 있지만 비영속 상태는 한번 영속상태가 된 `Entity` 이기 때문에 식별자 값이 반드시 존재한다. 


```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class DetachTest {
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void 영속Entity_detach하면_Commit해도_Insert쿼리수행되지않는다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);

        // when
        this.entityManager.detach(entity);
        this.entityManager.getTransaction().commit();

        // then
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: insert"
        )));
    }

    @Test
    public void 영속Entity_detach하면_Commit해도_Update쿼리수행되지않는다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();
        this.entityManager.getTransaction().begin();
        entity.setNum(2222);
        this.entityManager.detach(entity);

        // when
        this.entityManager.detach(entity);
        this.entityManager.getTransaction().commit();

        // then
        assertThat(this.capture.getOut(), allOf(
                containsString("Hibernate: insert"),
                not(containsString("Hibernate: update"))
        ));
    }

    @Test
    public void 영속Entity_detach하면_Commit해도_Delete쿼리수행되지않는다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();
        this.entityManager.getTransaction().begin();
        this.entityManager.remove(entity);
        this.entityManager.detach(entity);

        // when
        this.entityManager.detach(entity);
        this.entityManager.getTransaction().commit();

        // then
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: remove"
        )));
    }

    @Test
    public void 영속Entity_clear하면_Commit해도_모든쿼리수행하지않는다() {
        // given
        NoAutoIncrement entity1 = new NoAutoIncrement();
        entity1.setId(11);
        entity1.setNum(111);
        entity1.setStr("str111");
        this.entityManager.persist(entity1);
        NoAutoIncrement entity2 = new NoAutoIncrement();
        entity2.setId(22);
        entity2.setNum(222);
        entity2.setStr("str222");
        this.entityManager.persist(entity2);

        // when
        this.entityManager.clear();
        this.entityManager.getTransaction().commit();

        // then
        assertThat(this.capture.getOut(), not(containsString(
                "Hibernate: insert"
        )));
    }
}
```  

### merge
준영속 상태가 된 `Entity` 는 `merge()` 메소드를 사용해서 다시 영속 상태로 만들 수 있다. 
정확하게 말하면 `merge()` 메소드의 인자로 준영속 상태인 `Entity` 를 전달하면, 
`merge()` 메소드는 영속 상태인 새로운 `Entity` 인스턴스를 리턴한다. 
즉 인자 값으로 넘겨준 `Entity` 인스턴스는 계속해서 준영속 상태이고, 
해당 `Entity` 를 복사해서 `Persistence Context` 에 추가한 영속 인스턴스를 리턴한다. 


```java
@DataJpaTest(
        properties = {
                "spring.jpa.hibernate.ddl-update=update",
                "spring.jpa.properties.hibernate.show_sql=true"
        }
)
@RunWith(SpringRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class MergeTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TestEntityManager testEntityManager;
    private EntityManager entityManager;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        this.entityManager = this.testEntityManager.getEntityManager();
    }

    @Test
    public void 준영속Entity인스턴스와_merge리턴인스턴스는_같지않다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.detach(entity);

        // when
        NoAutoIncrement actual = this.entityManager.merge(entity);

        // then
        assertThat(actual, not(entity));
    }

    @Test
    public void 준영속Entity인스턴스와_merge후_find한영속인스턴스는_같지않다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.detach(entity);

        // when
        this.entityManager.merge(entity);
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual, not(entity));
    }

    @Test
    public void 준영속Entity_merge리턴인스턴스와_find한영속인스턴스는_같다() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.detach(entity);
        NoAutoIncrement mergedEntity = this.entityManager.merge(entity);

        // when
        NoAutoIncrement actual = this.entityManager.find(NoAutoIncrement.class, entity.getId());

        // then
        assertThat(actual, is(mergedEntity));
    }

    @Test
    public void 준영속Entity_merge하고_commit하면_Update쿼리수행() {
        // given
        NoAutoIncrement entity = new NoAutoIncrement();
        entity.setId(11);
        entity.setNum(111);
        entity.setStr("str111");
        this.entityManager.persist(entity);
        this.entityManager.getTransaction().commit();
        this.entityManager.getTransaction().begin();
        entity.setNum(222);
        this.entityManager.detach(entity);

        // when
        NoAutoIncrement mergedEntity = this.entityManager.merge(entity);
        this.entityManager.getTransaction().commit();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "Hibernate: insert", "Hibernate: select", "Hibernate: update"
        ));
    }
}
```  



---
## Reference
[Getting started with Spring Data JPA](https://spring.io/blog/2011/02/10/getting-started-with-spring-data-jpa)  
[JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)  
[Package javax.persistence](https://javaee.github.io/javaee-spec/javadocs/javax/persistence/package-summary.html)  
[Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)  
[Spring Boot JPA](https://www.javatpoint.com/spring-boot-jpa)  