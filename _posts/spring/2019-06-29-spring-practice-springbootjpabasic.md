--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot JPA 사용기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 JPA 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - JPA
    - ORM
    - Spring Boot
    - Spring Data JPA
---  

# 목표
- Spring JPA 사용법에 대해 익힌다.
- JPA 기본 개념에 대해 학습한다.
- JPA 기본 동작 외에 커스텀 메서드를 사용해본다.

# 방법
## JPA 란
- Java Persistence API 의 약자로 자바 진영의 ORM 기술 표준이다.
- ORM 이란 Object Relational Mapping 의약자로 객체와 테이블의 매핑을 이루는 것을 의미한다.
- ORM 을 이용하면 SQL Query 가 아닌 직관적인 코드(메서드)로 데이터를 조작할 수 있다.
- Query 를 직접 사용하지 않고 메서드 호출로 Query 를 수행하기 때문에 생산성이 매우 높아진다.
- Query 가 복잡해지면 ORM 으로 표현하는대에 한계가 있고, 성능이 일반 Query 에 비해 드리다. 

# 예제

## 프로젝트 구조
- 완성된 프로젝트의 구조는 아래와 같다.

![그림 1]({{site.baseurl}}/img/spring/practice-springbootjpabasic-1.png)


## pom.xml
- pom.xml 에서 사용된 의존성은 아래와 같다.

```xml
<dependencies>
	<!-- Spring JPA 의존성 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

	<!-- 데이터 저장소로 사용할 Java Embedded DB H2 의존성 -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Java Dev Tool Lombok 의존성 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- Spring Boot Test 의존성 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```  

- 데이터 저장소(DB) 는 간단한 구현을 위해 Java Embedded DB 인 H2 를 사용한다.

## Entity
- 데이터를 저장할 객체인 `Device`  Entity 는 아래와 같다.

```java
@RequiredArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
@ToString
@Setter
@Getter
@Entity
public class Device {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long serial;
    @NonNull
    private String name;
    @NonNull
    private int price;
}
```  

- 위 코드에서 `Entity`, `Id`, `GeneratedValue` Annotation 을 제와하면 모두 `Lombok` 관련 Annotation 이다.
- `@Entity` 는 해당 클래스를 DB에 매핑시킬 클래스라는 것을 선언하는 Annotation 이다.
- `@Id` 를 통해 해당 클래스의 필드를 기본키(PK) 로 지정한다.
- `@GenerateValue` 로 해당 필드의 PK 값을 자동으로 생성한다.
- `@GenerateValue` 로 키를 생성 할때는 아래와 같은 `strategy` 가 있다.
	- IDENTITY
		- 키 생성을 DB에 위임하는 방법이다.(DB에 의존적)
		- 주로 MySQL, PostgresSQL, SQL Server, DB2 에서 사용한다.
	- SEQUENCE
		- DB 시퀀스를 사용해서 기본 키를 할당하는 방법이다(DB에 의존적)
		- 주로 Oracle, PostgresSQL, DB2, H2 에서 사용한다.
		- `@SequenceGenerator` 를 사용하여 시퀀스 생성기를 등록하고, 실제 DB에 생성될 시퀀스 이름을 지정해야 한다.
	- TABLE 
		- 키 생성 테이블을 사용하는 방법이다.
		- 키 생성 전용 테이블을 하나 만들고, 이름과 값으로 사용할 컬럼을 만든다.
		- 테이블을 사용하기 때문에 DB에 의존하지 않고 모든 DB에 적용가능하다.
	- AUTO 
		- DB에 관계없이 식별자를 자동 생성 한다. 
		- DB가 변경되더라도 수정할 필요 없다.
		- IDENTITY, SEQUENCE, TABLE 중 하나를 자동으로 선택해 준다.
- `@Column` 은 DB 컬럼으로 지정해주는 Annotation 이다.
	- `lenght` 로 문자열의 길이를 지정 할 수 있다.
	- `nullabe` 로 널 여부를 설정할 수 있다.
	- 그 외에도 다양한 설정이 가능하다.
- `@Column` 은 DB가 컬럼 명에서 대소문자를 구분해야할 때는 필수로 사용해야 한다. 필드명을 사용해서 DB의 테이블과 매핑하기 때문이다.

## Repository
- 데이터 조작관련 메서드들이 있는 `DeviceRepository` Repository 인터페이스는 아래와 같다.

```java
@Repository
public interface DeviceRepository extends JpaRepository<Device, Long> {
	
}
```  

- `DeviceRepository` 인터페이스는 `JpaRepository` 를 상속 받고, JPA 에서 사용할 데이터의 타입과, 키의 타입을(`<Device, Long>`)제네릭 형태로 넣어주고 있다.
- `DeviceRepository` 인터페이스에 선언된 메서드 없지만 `JpaRepository` 인터페이스에는 기본적인 데이터 동작관련 메서드들이 선언되어 있고 이를 바로 사용할 수 있다.

## JPA 기본 동작 Test

- JPA 의 기본 동작을 테스트하는 테스트 클리스 `DeviceRepositoryTest` 클래스는 아래와 같다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DeviceRepositoryTest {

    @Autowired
    private DeviceRepository deviceRepository;

    @Before
    public void init() {
        this.deviceRepository.deleteAll();
    }

    @Test
    public void testFindById() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 3);
        d2 = this.deviceRepository.save(d2);

        Assert.assertEquals(d1, this.deviceRepository.findById(d1.getSerial()).get());
        Assert.assertEquals(d2, this.deviceRepository.findById(d2.getSerial()).get());
        Assert.assertTrue(this.deviceRepository.findById(d1.getSerial()).isPresent());
        Assert.assertFalse(this.deviceRepository.findById(111111111l).isPresent());
    }

    @Test
    public void testExistById() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 3);
        d2 = this.deviceRepository.save(d2);

        Assert.assertTrue(this.deviceRepository.existsById(d1.getSerial()));
        Assert.assertTrue(this.deviceRepository.existsById(d2.getSerial()));
    }

    @Test
    public void testFindAll() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 3);
        d2 = this.deviceRepository.save(d2);

        List<Device> deviceList = this.deviceRepository.findAll();
        Assert.assertEquals(2, deviceList.size());
        Assert.assertTrue(deviceList.contains(d1));
        Assert.assertTrue(deviceList.contains(d2));
    }

    @Test
    public void testDelete() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 3);
        d2 = this.deviceRepository.save(d2);

        this.deviceRepository.delete(d1);
        Assert.assertFalse(this.deviceRepository.existsById(d1.getSerial()));

        this.deviceRepository.deleteById(d2.getSerial());
        Assert.assertFalse(this.deviceRepository.existsById(d2.getSerial()));
    }
}
```  

## Repository 에 커스텀 동작(메서드) 추가

```java
@Repository
public interface DeviceRepository extends JpaRepository<Device, Long> {
    List<Device> findByName(String name);

    List<Device> findByPriceGreaterThanEqual(int price);

    List<Device> findByNameLike(String name);
    
    int countByName(String name);

    Device getById(long id);
}
```  

- 커스텀 메서드를 작성할때 구조는 아래와 같다.
	- `<동작>` + `<필드명>` + `<조건>` 으로 구성되어 있고 `<논리연산>` + `<필드명>` + `<조건>` 가 반복될 수 있다.

	Keyword|Sample|SQL
	---|---|---
	And|findByNameAndPrice|where name = ?1 and price = ?2
	Or|findByNameOrPrice|where ename = ?1 or price = ?2
	Is,Equals|findByName,findByNameIs,findByNameEquals|where name = ?1
	Between|findByPriceBetween|where price between ?1 and ?2
	LessThan|findByPriceLessThan|where price < ?1
	LessThanEqual|findByPriceLessThanEqual|where price <= ?1
	GreaterThan|findByPriceGreaterThan|where price > ?1
	GreaterThanEqual|findByPriceGreatherThanEqual|where price >= ?1
	After|findByDateAfter|where date > ?1
	Before|findByDateBefore|where date < ?1
	IsNull|findByNameIsNull|where name is null
	IsNotNull,NotNull|findByNameIsNotNull,findByNameNotNull|where name not null
	Like|findByNameLike|where name like ?1
	NotLike|findByNameNotLike|where name not like ?1
	StartingWith|findByNameStartingWith|where name like ?1(str%)
	EndingWith|findByNameEndingWith|where name like ?1(%str)
	Containing|findByNameContaining|where name like ?1(%str%)
	OrderBy|findByNameOrderByPriceDesc|where name = ?1 order by price desc
	Not|findByNameNot|where name <> ?1
	In|findByNameIn(Collections<> name)|where name in ?1
	NotIn|findByNameNotIn(Collectons<> name|where name not in ?1
	True|findByIsOnTrue|where isOne = true
	False|findByIsOnFalse|where isOne = false
	IgnoreCase|findByNameIgnoreCase|where UPPER(name) = UPPER(?1)
	
## 커스텀 동작(메서드) Test

```java
    @Test
    public void testFindByName() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 3);
        d2 = this.deviceRepository.save(d2);

        Device d3 = new Device("d3", 3);
        d3 = this.deviceRepository.save(d3);

        Device d4 = new Device("d3", 4);
        d4 = this.deviceRepository.save(d4);

        List<Device> deviceList = this.deviceRepository.findByName("d1");
        Assert.assertEquals(1, deviceList.size());
        Assert.assertTrue(deviceList.contains(d1));

        deviceList = this.deviceRepository.findByName("d3");
        Assert.assertEquals(2, deviceList.size());
    }
    
    @Test
    public void testFinByPriceGTE() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 2);
        d2 = this.deviceRepository.save(d2);

        Device d3 = new Device("d3", 3);
        d3 = this.deviceRepository.save(d3);

        Device d4 = new Device("d3", 4);
        d4 = this.deviceRepository.save(d4);

        List<Device> deviceList = this.deviceRepository.findByPriceGreaterThanEqual(1);
        Assert.assertEquals(4, deviceList.size());

        deviceList = this.deviceRepository.findByPriceGreaterThanEqual(2);
        Assert.assertEquals(3, deviceList.size());

        deviceList = this.deviceRepository.findByPriceGreaterThanEqual(3);
        Assert.assertEquals(2, deviceList.size());

        deviceList = this.deviceRepository.findByPriceGreaterThanEqual(4);
        Assert.assertEquals(1, deviceList.size());
    }

    @Test
    public void testFindByNameLike() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 2);
        d2 = this.deviceRepository.save(d2);

        Device d3 = new Device("c2", 3);
        d3 = this.deviceRepository.save(d3);

        Device d4 = new Device("e2", 4);
        d4 = this.deviceRepository.save(d4);

        List<Device> deviceList = this.deviceRepository.findByNameLike("d%");
        Assert.assertEquals(2, deviceList.size());
        Assert.assertTrue(deviceList.contains(d1));
        Assert.assertTrue(deviceList.contains(d2));

        deviceList = this.deviceRepository.findByNameLike("%2");
        Assert.assertEquals(3, deviceList.size());
        Assert.assertTrue(deviceList.contains(d2));
        Assert.assertTrue(deviceList.contains(d3));
        Assert.assertTrue(deviceList.contains(d4));
    }

    @Test
    public void testCountByName() {
        Device d1 = new Device("d1", 1);
        d1 = this.deviceRepository.save(d1);

        Device d2 = new Device("d2", 2);
        d2 = this.deviceRepository.save(d2);

        Device d3 = new Device("d2", 3);
        d3 = this.deviceRepository.save(d3);

        Device d4 = new Device("d3", 4);
        d4 = this.deviceRepository.save(d4);

        Assert.assertEquals(1, this.deviceRepository.countByName("d1"));
        Assert.assertEquals(2, this.deviceRepository.countByName("d2"));
        Assert.assertEquals(1, this.deviceRepository.countByName("d3"));
    }
```  

- JPA 에서 Get/Find 차이 추가하기

---
## Reference
[Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/data-jpa/docs/2.1.9.RELEASE/reference/html/)  
[Spring Boot JPA 사용해보기](https://velog.io/@junwoo4690/Spring-Boot-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0-erjpw41nl7)  
[JPA 식별자 자동 생성 (@GeneratedValue)](https://www.slideshare.net/topcredu/spring-data-jpaprimary-key-generatedvalue-strategy-generationtypeautogenerationtypetable-generationtypeidentityjpa)  
[[Spring] Spring Annotation의 종류와 그 역할](https://gmlwjd9405.github.io/2018/12/02/spring-annotation-types.html)  
[Spring Data JPA 기본키 매핑하는 방법](https://ithub.tistory.com/24)  