--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Redis Repository 기본 사용기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Redis Repository 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Redis
    - Spring Boot
    - Repository
---  

# 목표
- Spring Redis Data 에서 Repository 사용법에 대해 익힌다.

# 방법
- 데이터 모델 혹은 도메인에 `@RedisHash(<키>)` 를 통해 저장 시에 사용할 키를 지정한다.
- `CrudRepository` 를 상속받는 인터페이스를 만들어준다.
- `JPA` 의 메소드와 비슷하게 사용 가능하다.
	- 아직 몇몇의 키워드에 대한 커스텀 메서드는 지원하지 않는다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springbootredisrepository-1.png)

## pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-json</artifactId>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>RELEASE</version>
    </dependency>
    
    <!-- Embedded Redis -->
    <dependency>
        <groupId>it.ozimov</groupId>
        <artifactId>embedded-redis</artifactId>
        <version>0.7.2</version>
    </dependency>
</dependencies>
```  

- Redis 가 설치되지 않은 환경을 위해 `it.ozimov.embedded-redis` 를 사용했다.

## Application.properties

```
spring.redis.port=6379
spring.redis.host=localhost
```  

## Configuration

```java
@Configuration
@EnableRedisRepositories
public class RedisConfiguration {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();

        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);
        return lettuceConnectionFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(this.redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        return redisTemplate;
    }
}
```  

- `application.properties` 에 설정된 `spring.redis.port`, `spring.redis.hostname` 등의 값을 통해 Redis Server 와 연결을 설정한다.

## Domain

```java
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
@ToString
@Setter
@Getter
@RedisHash("Device")
public class Device implements Serializable {
    @Id
    private long serial;
    @Indexed
    private String name;
    private int price;
}
```  

- `@RedisHash` Annotation 을 이용해서 `Device` 도메인을 Redis Repository 의 `Device` 키로 저장한다.
- `@Indexed` 를 사용해서 `name` 필드로 쿼리 수행이 가능하도록 한다.

## Repository

```java
@Repository
public interface DeviceCrudRepository extends CrudRepository<Device, Long> {
    List<Device> findByName(String name);
}
```  

- `DeviceCrudRepository` 인터페이스는 `CrudRepository` 를 상속 받고, 데이터의 타입과 키의 타입(<Device, Long>)을 선언해 주었다.
- 기본적으로 `findById`, `count`, `delete` 등의 메서드는 상속 받고 있어서 별도의 구현없이 사용할 수 있다.
- 커스텀 메서드로 인덱스를 지정한 name 필드를 기준으로 찾는 `findByName` 을 추가하였다.
- Redis Repository 쿼리에 대한 더 자세한 내용은 [Spring Data Redis #Query by Example](https://docs.spring.io/spring-data/redis/docs/current/reference/html/#query-by-example.execution) 에서 확인 가능하다.

## Redis Repository Test

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DeviceCrudRepositoryTest {
    @Autowired
    private DeviceCrudRepository deviceCrudRepository;

    @Before
    public void init() {
        this.deviceCrudRepository.deleteAll();
    }

    @Test
    public void testSaveFind() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d2", 3);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);

        Assert.assertEquals(d1, this.deviceCrudRepository.findById(d1.getSerial()).get());
        Assert.assertEquals(d2, this.deviceCrudRepository.findById(d2.getSerial()).get());
        Assert.assertFalse(this.deviceCrudRepository.findById(112222l).isPresent());
    }

    @Test
    public void testExistsById() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d2", 3);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);


        Assert.assertTrue(this.deviceCrudRepository.existsById(d1.getSerial()));
        Assert.assertTrue(this.deviceCrudRepository.existsById(d2.getSerial()));
        Assert.assertFalse(this.deviceCrudRepository.existsById(1112222l));
    }

    @Test
    public void testFindAll() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d2", 3);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);

        List<Device> deviceList = new ArrayList<>();
        Iterable<Device> deviceIterable = this.deviceCrudRepository.findAll();
        deviceIterable.forEach(deviceList::add);

        Assert.assertEquals(2, deviceList.size());
        Assert.assertTrue(deviceList.contains(d1));
        Assert.assertTrue(deviceList.contains(d2));
    }

    @Test
    public void testDelete() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d2", 3);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);

        Assert.assertTrue(this.deviceCrudRepository.existsById(1l));
        Assert.assertTrue(this.deviceCrudRepository.existsById(2l));

        this.deviceCrudRepository.deleteById(2l);

        Assert.assertTrue(this.deviceCrudRepository.existsById(1l));
        Assert.assertFalse(this.deviceCrudRepository.existsById(2l));
    }

    @Test
    public void testCount() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d2", 3);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);

        Assert.assertEquals(2, this.deviceCrudRepository.count());
    }

    @Test
    public void testFindByName() {
        Device d1 = new Device(1l, "d1", 1);
        Device d2 = new Device(2l, "d1", 3);
        Device d3 = new Device(3l, "c2", 4);
        Device d4 = new Device(4l, "c2", 5);

        d1 = this.deviceCrudRepository.save(d1);
        d2 = this.deviceCrudRepository.save(d2);
        d3 = this.deviceCrudRepository.save(d3);
        d4 = this.deviceCrudRepository.save(d4);

        List<Device> deviceList = this.deviceCrudRepository.findByName("d1");

        Assert.assertEquals(2, deviceList.size());
        Assert.assertTrue(deviceList.contains(d1));
        Assert.assertTrue(deviceList.contains(d2));
    }
}
```  

## Redis 데이터 구조
- 위 테스트 코드를 돌렸을 때, Redis Server 에 저장된 키는 아래와 같다.

	```
	127.0.0.1:6379> keys *
	 1) "Device:4"
	 2) "Device:1:idx"
	 3) "Device:3:idx"
	 4) "Device:name:d1"
	 5) "Device:3"
	 6) "Device:name:c2"
	 7) "Device"
	 8) "Device:1"
	 9) "Device:4:idx"
	10) "Device:2:idx"
	11) "Device:2"
	```  

	Key|Type|Key Desc|Stored Data Desc|Key Ex
	---|---|---|---|---
	`Device`|Sets|Device 의 Redis 키|Device 키로 저장된 데이터들의 <id>를 저장|`Device`
	`Device:<index-field>:<field-value>`|Sets|<index-field> 에서 값이 <field-value> 인 인덱스 키|index 로 지정된 <index-field> 가 <field-value> 인 Device 데이터의 키 저장|`Device:name:d1`
	`Device:<id>:idx`|Sets|Device 데이터에서 <id> 의 인덱스 키|Device 데이터에서 <id> 가 가지고 있는 <index-key> 저장|`Device:1:idx`
	`Device:<id>`|hashes|Device 데이터에서 <id> 의 데이터 키|Device 데이터에서 <id> 의 데이터 정보를 저장|`Device:1`

- `Device` 의 키에는 Device 데이터 의 `<id>` 가 저장된다.

	```
	127.0.0.1:6379> type Device
	set
	127.0.0.1:6379> smembers Device
	1) "1"
	2) "2"
	3) "3"
	4) "4"
	```  

- `Device:<index-field>:<fiend-value>` 의 키에는 해당 인덱스 필드 값이 `<field-value>` 인 `<id>` 가 저장된다.

	```
	127.0.0.1:6379> type Device:name:d1
	set
	127.0.0.1:6379> smembers Device:name:d1
	1) "1"
	2) "2"
	```  
	
- `Device:<id>:idx` 의 키에는 `<id>` 의 데이터가 해당되는 인덱스의 키 값이 저장된다.

	```
	127.0.0.1:6379> type Device:1:idx
	set
	127.0.0.1:6379> smembers Device:1:idx
	1) "Device:name:d1"
	```  
	
- `Device:<id>` 의 키에는 `<id>` 의 데이터 정보가 저장된다.

	```
	127.0.0.1:6379> type Device:1
	hash
	127.0.0.1:6379> hgetall Device:1
	1) "_class"
	2) "com.example.demo.domain.Device"
	3) "serial"
	4) "1"
	5) "name"
	6) "d1"
	7) "price"
	8) "1"
	```  
	
---
## Reference
[Spring Data Redis](https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/#redis.repositories)  
[Spring Data – Add custom method to Repository](https://www.mkyong.com/spring-data/spring-data-add-custom-method-to-repository/)  
[[Spring Boot #27] 스프링 부트 레디스(Redis) 연동하기](https://engkimbs.tistory.com/796)  
[SpringBoot Data Redis 로컬/통합 테스트 환경 구축하기](https://jojoldu.tistory.com/297)  
[Springboot,redis - Redis repository 간단한사용법!](https://coding-start.tistory.com/130)  