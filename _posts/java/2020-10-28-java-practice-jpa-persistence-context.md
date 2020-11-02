--- 
layout: single
classes: wide
title: "[Java 실습] JPA Architecture 와 Persistence Context"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
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

`JPA` 구조를 클래스 레벨로 도식화 하면 아래와 같다.  

.. 그림 ..  
https://2.bp.blogspot.com/-zQCYV-qJCU0/XAPuPHYDHSI/AAAAAAAAFEs/B2_uskqyAss_ZPX_EzsJujEYtirEtu49gCLcBGAs/s1600/jpa_class_level_architecture.png  
혹은 아래 그림이랑 합쳐서   
https://www.programmersought.com/images/965/15788f6b0fda48a893a27ed20f228cc5.jpg  
https://javabydeveloper.com/wp-content/uploads/2019/12/Jpa-architecture-768x403.png?ezimgfmt=ng:webp/ngcb102  

- `Persistence` : 
- `EntityManagerFactory` : 
- `EntityManager` : 
- `Entity` : 
- `Query` : 

`javax.persistence` 패키지를 구성하는 주요 클래스와 인터페이스의 관계를 도식화 하면 아래와 같다. 

.. 그림 ..  
https://4.bp.blogspot.com/-fVjv79bHZMA/XAPvFu2YTDI/AAAAAAAAFE4/iaL19yzi2boDDhVTffQWInYFlNVwYDDOQCLcBGAs/s1600/jpa_class_relationships.png  


## EntityManager, EntityManagerFactory ?????
https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fxn0pW%2FbtqyjdzFtfd%2F9tmnEVN7UPnOFqGSIu0GZ1%2Fimg.png  

## Persistence Context

-  A persistence context handles a set of entities which hold data to be persisted in some persistence store (e.g. database).




논리적인 개념으로 EntityManager 를 통해 Persistence Context 에 접근해서 Entity를 관리하는 역할
Entity를 영구 저장하는 환경
여러 EntityManager 가 같은 Persistence Context 에 접근할 수 있음
    같은 트랜잭션의 범위에 있는 EntityManager 는 동일한 Persistence Context 에 접근한다. 




### 생명주기

https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fcc3i97%2FbtqxX8e4Rgz%2Fm2kuh5rH6LKk831Rv2eNyk%2Fimg.png  
https://github.com/namjunemy/TIL/blob/master/Jpa/tacademy/img/12_entity_lifecycle.PNG?raw=true  
https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=http%3A%2F%2Fcfile21.uf.tistory.com%2Fimage%2F99CA54415AE4627A2C7BE0  



<!-- ## Persistence Context 장점 ????????????????? -->
### 1차캐시

### 동일성

### 쓰기 지연

### Dirty Checking

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