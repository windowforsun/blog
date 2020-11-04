--- 
layout: single
classes: wide
title: "[Spring 개념] JPA, Hibernate, Spring Data JPA 관계"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Java 와 Spring 에서 ORM 을 제공하는 JPA, Hibernate, Spring Data JPA 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - JPA
    - Hibernate
    - Spring Data JPA
    - ORM
toc: true
use_math: true
---  

## JPA
[여기]({{site.baseurl}}{% link _posts/spring/2020-10-28-spring-concept-jpa-persistence-context.md %})
에서도 관련 내용이 있지만, 
`JPA` 는 `Java Persistence API` 의 약자로 자바 애플리케이션에서 관계형 데이터베이스를 사용하는 방식을 정의한 인터페이스의 모음이다. 
여기서 인터페이스의 모음이라는 의미가 말그대로 `JPA` 관련 클래스와 인터페이스가 있는 `javax.persistence` 패키지를 보면, 
말그대로 인터페이스만으로 정의된 것을 확인 할 수 있다. 
아래는 `javax.persistence` 패키지에서 핵심이 될 수 있는 `EntityManager` 내용의 일부이다. 

```java
public interface EntityManager {

    public void persist(Object entity);
 
    public <T> T merge(T entity);
 
    public void remove(Object entity);

    public <T> T find(Class<T> entityClass, Object primaryKey);

    public <T> T find(Class<T> entityClass, Object primaryKey, 
                      Map<String, Object> properties); 
    
    public <T> T find(Class<T> entityClass, Object primaryKey,
                      LockModeType lockMode);

    public <T> T find(Class<T> entityClass, Object primaryKey,
                      LockModeType lockMode, 
                      Map<String, Object> properties);

    public <T> T getReference(Class<T> entityClass, 
                                  Object primaryKey);
    public void flush();

    .. 생략 ..
}
```  

위와 같이 `JPA` 라는 것은 `Java ORM` 의 표준 인터페이스의 역할만 수행한다. 
이는 마치 서버 `API` 는 클라이언트에서 서버를 사용할 수 있는 방법을 정의한 것과 같이 
`JPA` 또한 관계형 데이터베이스를 어떻게 사용해야하는지 정의한 방법일 뿐이다.  


## Hibernate
앞서 `JPA` 는 말그대로 인터페이스라고 설명했다. 
이 인터페이스를 실제로 구현한 구현체가 바로 `Hibernate` 이다. 
간단하게 비유하면 `JPA` 는 `interface` 이고, `Hiberate` 는 `ConcreteClas` 로 비유할 수 있다. 
즉 실제 데이터베이스 관련 구체적인 동작과 구현 등은 모두 `Hibernate` 에서 구현돼있다. 
`Hibernate` 문서에서 사용하고 있는 아래 그림을 보면 더욱 명확하게 이해할 수 있다. 

![그림 1]({{site.baseurl}}/img/spring/concept-jpa-hibernate-spring-data-jpa-1.svg)  

위 그림을 보면 `javax.psersistence` 패키지의 주요 인터페이스인 
`EntityManagerFactory`, `EntityManager`, `EntityTransaction` 을 상속하는 
`SessionFactory`, `Session`, `Transaction` 을 확인할 수 있다.  

`Hibernate` 공식문서의 그림을 빌려 전체적인 데이터베이스 접근을 위한 레이어를 표현하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-jpa-hibernate-spring-data-jpa-2.svg)  

애플리케이션에서 데이터베이스의 `Entity` 에 접근하기 위해서는 `JPA` 와 `Hibernate Native API` 두 가지 모두 사용할 수 있다. 
다른 두 `API` 를 사용하더라도 실제로 데이터베이스와 `Entity` 관련 동작은 `Hibernate` 를 사용해서 수행하게 된다.   



## Spring Data JPA
`Spring Data JPA` 는 `Spring` 에서 제공하는 여러가지 모듈 중 하나로, 
`JPA` 를 보다 쉽고 간편하게 사용할 수 있도록 한다. 
`JPA` 를 `Repository` 라는 인터페이스로 한번 더 추상화해 `Repository` 단위로 데이터를 조작할 수 있다. 
그리고 `Repository` 는 정해진 규책대로 메소드 이름과 리턴타입, 인자값을 정의해주면 모듈 내부에서 `JPA` 를 사용해 
적절한 쿼리를 수행하는 방식을 사용한다.  

실제로 `Spring Data JPA` 를 사용하면 `JPA` 의 핵심 인터페이스인 `EntityManager` 의 존재를 느낄 수 없다. 
`Repository` 인터페이스의 기본 구현체인 `SimpleJpaRepository` 클래스를 확인하면 아래와 같다. 

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
    
    .. 생략 ..

	private final EntityManager em;

	public SimpleJpaRepository(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {

		Assert.notNull(entityInformation, "JpaEntityInformation must not be null!");
		Assert.notNull(entityManager, "EntityManager must not be null!");

		this.entityInformation = entityInformation;
		this.em = entityManager;
		this.provider = PersistenceProvider.fromEntityManager(entityManager);
	}

    public void delete(T entity) {

        Assert.notNull(entity, "Entity must not be null!");

        if (entityInformation.isNew(entity)) {
            return;
        }

        Class<?> type = ProxyUtils.getUserClass(entity);

        T existing = (T) em.find(type, entityInformation.getId(entity));

        // if the entity to be deleted doesn't exist, delete is a NOOP
        if (existing == null) {
            return;
        }

        em.remove(em.contains(entity) ? entity : em.merge(entity));
	}

    public Optional<T> findById(ID id) {

		Assert.notNull(id, ID_MUST_NOT_BE_NULL);

		Class<T> domainType = getDomainClass();

		if (metadata == null) {
			return Optional.ofNullable(em.find(domainType, id));
		}

		LockModeType type = metadata.getLockModeType();

		Map<String, Object> hints = getQueryHints().withFetchGraphs(em).asMap();

		return Optional.ofNullable(type == null ? em.find(domainType, id, hints) : em.find(domainType, id, type, hints));
	}

    public <S extends T> S save(S entity) {

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
    
    .. 생략 ..
}
```  

`SimpleJpaRepository` 클래스의 멤버 필드에 `EntityManager` 가 선언된 것을 확인할 수 있고, 
생성자에서 `EntityManager` 를 인자로 받아 멤버 필드에 설정해 준다.  
그리고 주요 메소드인 `delete`, `findById`, `save` 메소드를 확인하면, 
메소드 구현에서 `EntityManager` 를 사용해서 메소드에 맞는 관련처리를 수행하는 것을 확인 할 수 있다.  

## 계층 구조
`JPA`, `Hibernate,` `Spring Data JPA` 모두 `Java ORM` 사용을 위한 스펙이고 라이브러리이자 모듈이다. 
이를 계층 구조로 표현하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/concept_jpa_hibernate_spring_data_jpa.svg) 

---
## Reference
[Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)  
[Spring Boot JPA](https://www.javatpoint.com/spring-boot-jpa)  
[Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference)  