--- 
layout: single
classes: wide
title: "[Java 실습] JPA Architecture 와 Persistence Context"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
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



## EntityManager, EntityManagerFactory ?????
[](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fxn0pW%2FbtqyjdzFtfd%2F9tmnEVN7UPnOFqGSIu0GZ1%2Fimg.png)  




## Persistence Context
`Persistence Context` 는 `Entity` 를 영구 저장하는 논리적인 공간이면서 개념이다. 
`EntityManager` 와 `Persistence Context` 가 1:1 관계는 갖는 경우도 있지만, 
트랜잭션, 스레드를 기준으로 여러 `EntityManager` 가 `Persistence Context` 를 공유해서 사용하는 경우도 있다. 

### 생명주기
`Persistence Context` 의 생명주기에는 `New(Transient)`, `Managed`, `Detached`, `Removed` 가 있는데 
전체적인 흐름은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept_jpa_3.png)  

- `New(Transient)` : `Entity` 에 해당하는 객체가 인스턴스로 생성만 된 상태로 아직 `Persistence Context` 와는 관련이 없는 상태이다. 

    ```java    
    NoAutoIncrement entity = new NoAutoIncrement();
    entity.setId(11);
    entity.setNum(111);
    entity.setStr("str111");
    ```  

- `Managed` : `Entity` 가 `Persistence Context` 에 저장된 상태로 `EntityManager` 에 의해 관리 된다. 
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
  
- `Detached` : `Persistence Context` 에 저장돼 있다가 분리된 상태로, `EntityManager` 의 관리 대상에서 벗어난 `Entity` 를 의미한다. 
`detach()` 메소드 호출로 `Detached` 상태를 만들 수 있다. 

    ```java
    entityManager.detach(entity);
    ```  

- `Removed` : `Persistence Context` 와 데이터베이스에서도 제거되는 상태이다. 
`remove()` 메소드 호출로 `Entity` 를 삭제할 수 있다. 

    ```java
    entityManager.remove(entity);
    ```  


<!-- ## Persistence Context 장점 ????????????????? -->
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


```  


### 동일성
`Persistence Context` 에서 같은 키로 반환하는 `Entity` 의 인스턴스는 동일성을 보장한다. 
이 또한 1차 캐시를 사용하기 때문에 가능하다. 
반복적인 읽기 동작에 대해서 애플리케이션 수준에서 `REPEATABLE READ` 격리 수준을 제공한다.  

```java


```  

### 쓰기 지연
`persist()` 메소드를 사용해서 `Entity` 를 `Persistence Context` 에 추가하면 바로 데이터베이스에 `INSERT` 쿼리가 수행되지 않는다. 
`EntityManager` 는 이를 쿼리 저장소라는 곳에 추가된 `Entity` 의 `INSERT` 쿼리을 저장해 놓게 된다. 
그리고 `commit()`, `flush()` 가 호출될때 저장해 둔 `INSERT` 쿼리를 데이터베이스에 보내 실제로 저장한다.  

<!-- 이러한 쓰기 지연을 사용하게 되면 추가되는 `Entity` 가 있을 때마다 `INSERT` 쿼리를 수행하는 것이 아니라, -->
<!-- 모아 뒀다가 한번에 `INSERT` 쿼리를 데이터베이스로 보내 수행하기 때문에 성능을 높일 수 있다. -->

`EntityManager` 에서 `commit()` 메소드를 수행하게 되면, 
내부에서 먼저 `flush()` 를 호출해서 `Persistence Context` 와 데이터베이스의 동기화하는 작업을 수행한다. 
그리고 작업이 완료되면 최종적으로 `commit` 을 수행해서 데이터베이스에 영구적으로 반영한다.  

`hibernate.jdbc.batch_size` 의 키 값으로 한번에 처리할 `INSERT` 쿼리의 크기를 지정할 수 있다.   

```java


```  



### Dirty Checking
`Persistence Context` 에 저장된 `Entity` 


<!-- ## Persistence Context 특징 ????????????????? -->
### flush

### remove

### detached

### merge




























### Persistence
`Persistence` 는 `EntityManager` 인스턴스를 생성하는 정적 메소드를 가지고 있는 클래스이다. 

- Persistence (javax.persistence.Persistence) class contains java static methods to get EntityManagerFactory instances.
- Since JPA 2.1 you can generatedatabase schemas and/or tables and/or create DDL scripts as determined by the supplied properties.
- It is a class that contains static methods to obtain an EntityManagerFactory instance.
- The javax.persistence.Persistence class contains static helper methods to obtain EntityManagerFactory instances in a vendor-neutral fashion.
- This class contains static methods to obtain EntityManagerFactory instance.
- This class contains static methods to obtain the EntityManagerFactory instance.














































### Persistence Unit










- A persistence unit defines a set of all entity classes that are managed by EntityManager instances in an application.
- This set of entity classes represents the data contained within a single data store.
- Persistence units are defined by the persistence.xml configuration file.
- JPA uses the persistence.xml file to create the connection and setup the required environment. Provides information which is necessary for making database connections.
- It defines a set of all entity classes. In an application, EntityManager instances manage it. The set of entity classes represents the data contained within a single data store.








### EntityManagerFactory
- It is a factory class of EntityManager. It creates and manages multiple instances of EntityManager.
- EntityManagerFactory (javax.persistence.EntityManagerFactory) class is a factory for EntityManagers.
- During the application startup time EntityManagerFactory is created with the help of Persistence-Unit.
- Typically, EntityManagerFactory is created once (One EntityManagerfactory object per Database) and kept alive for later use.
- Responsible for creating EntityManager instances.
- Initialized with the help of persistence context.
- This is a factory class of EntityManager. It creates and manages multiple EntityManager instances.
- Interface used to interact with the entity manager factory for the persistence unit.
- When the application has finished using the entity manager factory, and/or at application shutdown, the application should close the entity manager factory. Once an EntityManagerFactory has been closed, all its entity managers are considered to be in the closed state.
- EntityManager 인스턴스 관리 역할
- Persistence 를 통해 생성할 수 있음
- EntityManagerFactory 가 생성될때 Connection Pool 생성을 포함한 여러 작업이 수행되기 때문에 비용이 매우 큼
- DataSource 당 하나의 EntityManagerFactory 생성
- EntityManager 를 생성하는 역할
- Thread-safe





### EntityManager
- It is an interface. It controls the persistence operations on objects. It works for the Query instance.
- EntityManager is an interface to perform main actual database interactions.
- Creates persistence instance.
- Removes persistence instance.
- Finds entities by entity’s primary key.
- Allows queries to be run on entities.
- Each EntityManager instance is associated with a PersistenceContext.
- Performs the real work of persistence.
- Performs create, update and delete operations on entities in persistence context.
- Brings entities into persistence context and then entity is supposed to be managed by EntityManager .
- Multiple EntityManager instances can be created from EntityManagerFactory .
- The javax.persistence.EntityManager is the primary JPA interface used by applications. Each EntityManager manages a set of persistent objects, and has APIs to insert new objects and delete existing ones. When used outside the container, there is a one-to-one relationship between an EntityManager and an EntityTransaction. EntityManagers also act as factories for Query instances.
- It is an Interface, it manages the persistence operations on objects. It works like a factory for Query instance.
- EntityManager 는 Persistence Context 와 상호 작용하는 인터페이스로, 
- 해당 인스턴스는 Persistence Context 와 연관있다. 
- 여기서 Persistence Context 는 엔티티에서 고유한 id 를 가지고 있는 엔티티 인스턴스의 집합이다. 
- Persistence Context 에서 엔티티 인스턴스는 정해진 라이프 사이클을 통해 관리된다. 
- EntityManager API 는 Persistence Context 와 연관되어 엔티티 인스턴스를 생성, 제거, 찾는 동작이나, 
- 엔티티에 대해 쿼리를 수행할 수 있다. 
- EntityManager 에서 관리하는 엔티티 집합은 지속성 단위에 의해 정의된다. 
- 지속성 단위는 모든 클래스에서 애플리케이션에 의해 그룹화 되면서 단일 데이터베이스에 매핑될 있는 것들로 정의된 집합이다. 
- Entity 생성, 수정, 삭제, 조회 등 Entity 관리 역할
- EntityManager 생성 비용은 거의 들지 않음
- EntityManager 는 데이터베이스 연결이 꼭 필요한 시점까지 커넥션을 얻지 않음
- 실제 DB Connection 과 밀접한 관계가 있음
- EntityManager 관련 동작은 Transaction 안에서 작업이 수행되야 함(트랜잭션이 없으면 예외발생)
- JPA 에서는 EntityManager 는 Thread 단위(요청?)로 생성됨
- Thread-unsafe 이므로 Thread 간 공유되지 않도록 주의가 필요함






### Persistence Unit
- It defines a set of all entity classes. In an application, EntityManager instances manage it. The set of entity classes represents the data contained within a single data store.
- A persistence unit defines a set of all entity classes that are managed by EntityManager instances in an application.
- This set of entity classes represents the data contained within a single data store.
- Persistence units are defined by the persistence.xml configuration file.
- JPA uses the persistence.xml file to create the connection and setup the required environment. Provides information which is necessary for making database connections.
- The persistence.xml configuration file is used to configure a given JPA Persistence Unit. The Persistence Unit defines all the metadata required to bootstrap an EntityManagerFactory, like entity mappings, data source, and transaction settings, as well as JPA provider configuration properties.
- The goal of the EntityManagerFactory is used to create EntityManager objects we can for entity state transitions.
- So, the persistence.xml configuration file defines all the metadata we need in order to bootstrap a JPA EntityManagerFactory.









### Persistence Context












































































### EntityManagerFactory

### EntityManager



## Entity LifeCycle






## JPA Persistence Context
`JPA` 를 사용하면 `JPA` 와 데이터베이스 사이에 영속성 컨텍스트(`Persistence Context`) 
라는 논리적인 개념을 두고 데이터를 관리한다. 
`Persistence Context` 는 데이터베이스의 데이터인 `Entity` 를 영구 저장하는 환경이다. 
이런 `Persistence Context` 는 `EntityManager` 를 통해 `Entity` 를 관리하는데, 
이러한 `EntityManager` 를 생성하는 것이 바로 `EntityManagerFactory` 이다. 

### EntityManagerFactory 
`EntityManagerFactory` 를 생성하는 비용은 비교적 


## Entity LifeCycle


## Persistence Context 장점

### 1차캐시

### 동일성

### 쓰기 지연

### Dirty Checking

## Persistence Context 특징
### flush

### remove

### detached

### merge
















EntityManagerFactory
EntityManager 인스턴스 관리 역할
Persistence 를 통해 생성할 수 있음
EntityManagerFactory 가 생성될때 Connection Pool 생성을 포함한 여러 작업이 수행되기 때문에 비용이 매우 큼
DataSource 당 하나의 EntityManagerFactory 생성
EntityManager 를 생성하는 역할
Thread-safe


EntityManager 
Entity 생성, 수정, 삭제, 조회 등 Entity 관리 역할
EntityManager 생성 비용은 거의 들지 않음
EntityManager 는 데이터베이스 연결이 꼭 필요한 시점까지 커넥션을 얻지 않음
실제 DB Connection 과 밀접한 관계가 있음
EntityManager 관련 동작은 Transaction 안에서 작업이 수행되야 함
JPA 에서는 EntityManager 는 Thread 단위(요청?)로 생성됨
Thread-unsafe 이므로 Thread 간 공유되지 않도록 주의가 필요함

Persistence Context
논리적인 개념으로 EntityManager 를 통해 Persistence Context 에 접근해서 Entity를 관리하는 역할
Entity를 영구 저장하는 환경
여러 EntityManager 가 같은 Persistence Context 에 접근할 수 있음
    같은 트랜잭션의 범위에 있는 EntityManager 는 동일한 Persistence Context 에 접근한다. 







---
## Reference
[[JPA] 영속성 컨텍스트와 플러시 이해하기](https://ict-nroo.tistory.com/130)  
[JPA - Persistence Context (영속성 컨텍스트)](https://heowc.tistory.com/55)  
[더티 체킹 (Dirty Checking)이란?](https://jojoldu.tistory.com/415)  
[JPA 더티 체킹(Dirty Checking)이란?](https://interconnection.tistory.com/121)  
[JPA 변경 감지와 스프링 데이터](https://medium.com/@SlackBeck/jpa-%EB%B3%80%EA%B2%BD-%EA%B0%90%EC%A7%80%EC%99%80-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-2e01ad594b82)  
[(JPA - 2) 영속성(Persistence) 관리](https://kihoonkim.github.io/2017/01/27/JPA(Java%20ORM)/2.%20JPA-%EC%98%81%EC%86%8D%EC%84%B1%20%EA%B4%80%EB%A6%AC/)  
[[Spring JPA] 영속 환경 ( Persistence Context )](https://victorydntmd.tistory.com/207)  


[Getting started with Spring Data JPA](https://spring.io/blog/2011/02/10/getting-started-with-spring-data-jpa)  
[JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)  
[Package javax.persistence](https://javaee.github.io/javaee-spec/javadocs/javax/persistence/package-summary.html)  
[Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)  
[Spring Boot JPA](https://www.javatpoint.com/spring-boot-jpa)  