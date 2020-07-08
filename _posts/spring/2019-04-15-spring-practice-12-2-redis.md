--- 
layout: single
classes: wide
title: "[Spring 실습] Redis 사용하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Redis 를 사용하는 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Redis
---  

# 목표
- Redis 를 사용해서 데이터를 저장하고 가져와보자.

# 방법
- Redis Client 를 설치하고 Redis 관련 의존성 및 관련 Third Party 라이브러리 의존성을 추가한다.
	
# 예제
- Redis 는 Key-Value 형식으로 저장하는 Cash 또는 데이터 저장소로, 문자열 및 다양한 데이터 구조를 지원한다.
- 기본 포트는 6379 를 사용한다.
- [Redis 란]({{site.baseurl}}{% link _posts/redis/2019-04-10-redis-concept-intro.md %})

## Redis 설치하기
- [Linux]({{site.baseurl}}{% link _posts/linux/2019-04-03-linux-practice-redisinstall.md %})
- [Windows](http://redis.io/download)
	
## Redis 접속하기
- Java Application 에서 Redis 에 접속하기 위해 Java 용 Redis Client 의존성을 추가한다.
	- 몇가지 Redis Client 가 있지만 [Jedis](https://github.com/xetorthio/jedis) 를 사용한다.
	- [Redis Client 목록](http://redis.io/clients)
- Maven(pom.xml)
	
	```xml
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-redis</artifactId>
        <version>2.0.3.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.9.0</version>
    </dependency>
	```  
	
	- spring-data-redis 2 버전은 Spring Framework 5 이상 부터 사용할 수 있다.
	
- Gradle(build.gradle)

	```
	dependencies {
		compile "org.springframework.data:spring-data-redis:2.0.3.RELEASE"
		compile "redis.clients:jedis:2.9.0"
	}
	```  
	
- 간단한 예제로 접속 테스트를 해본다.

```java
public class Main {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost");
        jedis.set("msg", "Hello World, from Redis!");
        System.out.println(jedis.get("msg"));
        jedis.close();
    }
}
```  

- 단순 문자열의 데이터 외에 List, Map 에 대한 테스트도 해본다.

```java
public class Main {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost");
        jedis.rpush("authors", "Marten Deinum", "Josh Long", "Daniel Rubio", "Gary Mak");
        System.out.println("Authors: " + jedis.lrange("authors",0,-1));

        jedis.hset("sr_3", "authors", "Gary Mak, Danial Rubio, Josh Long, Marten Deinum");
        jedis.hset("sr_3", "published", "2014");

        jedis.hset("sr_4", "authors", "Josh Long, Marten Deinum");
        jedis.hset("sr_4", "published", "2017");


        System.out.println("Spring Recipes 3rd: " + jedis.hgetAll("sr_3"));
        System.out.println("Spring Recipes 4th: " + jedis.hgetAll("sr_4"));

    }
}
```  

- List 관련 메서드인 rpush(), lpush() 는 List 관련 명령어로 각각 List 의 좌, 우에 데이터를 추가한다.
	- List 에서 데이터를 가져올 때는 lrang(), rrange() 메서드를 사용한다.
	- 인수중 중 0, -1 은 List 데이터 전부를 가져오게 된다.
- Map 관련 메서드인 hset() 은 엘리먼트를 추가하는 메서드로 key, field, value 세 인수를 받는다.
	- Map 형식을 인수로 받는 hmset() 멀티셋 메서드도 있다.
	
## Redis 에 Object 저장하기
- Redis 에 Object 를 저장하기 위해서는 Serialize(직렬화) 가 필요하다.
- Serialize 를 적용한 클래스는 아래와 같다.

```java
public class Vehicle implements Serializable {
    private String vehicleNo;
    private String color;
    private int wheel;
    private int seat;

    public Vehicle() {
    }

    public Vehicle(String vehicleNo, String color, int wheel, int seat) {
        this.vehicleNo = vehicleNo;
        this.color = color;
        this.wheel = wheel;
        this.seat = seat;
    }

	// getter, setter
}
```  

- Vehicle 클래스는 Serialize 를 위해 Serializable 인터페이스를 구현 하였다.
	- Java 객체 Serialize 를 위해서는 이 인터페이스를 구현해야 한다.
	- 
- Serializable 는 Java Object 를 byte[] 로 변환해 준다.
	- Serializable 를 구현한 Object 를 byte[] 로 변환하기 위해서 객체를 ObjectOutputStream 으로 출력하고 ByteArrayOutputStream 을 통해 byte[] 변환한다.
	- byte[] 를 다시 Object 로 변환하기 위해 byte[] 를 ByteArrayInputStream 을 통해 입력을 받아 ObjectInputStream 을 통해 Object 로 변환하게 된다.
- Spring 에서는 org.springframework.util.SerializationUtil 를 사용해 serialize()/deserialize() 메서드를 제공한다.
- Vehicle 클래스의 객체를 Serialize 해 Redis 저장하는 예제는 아래와 같다.

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Jedis jedis = new Jedis("localhost");

        final String vehicleNo = "TEM0001";
        Vehicle vehicle = new Vehicle(vehicleNo, "RED", 4,4);

        jedis.set(vehicleNo.getBytes(), SerializationUtils.serialize(vehicle));

        byte[] vehicleArray = jedis.get(vehicleNo.getBytes());

        System.out.println("Vehicle: " + SerializationUtils.deserialize(vehicleArray));

        jedis.close();
    }
}
```  

- byte[] 로 변환하는 방법보다는 XML, Json 형식을 가질 수 있는 String 이 편의성을 높일 수 있다.
- Java Object 를 Json 으로 변환하기 위해 Jasckson Json 라이브러리를 사용한다.
- 라이브러리 사용을 위해 Jackson 의존성을 추가한다.
- Maven(pom.xml)

	```xml
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.9.3</version>
    </dependency>
	```  

- Gradle(build.gradle)

	```
	dependencies {
		compile "com.fasterxml.jackson.core:jackson-databind:2.9.3"
	}
	```  
	
- Jackson Json 라이브러리를 사용해서 Redis 에 저장하는 예제는 아래와 같다.

```java
public class Main {

    public static void main(String[] args) throws Exception {
        Jedis jedis = new Jedis("localhost");
        ObjectMapper mapper = new ObjectMapper();
        final String vehicleNo = "TEM0001";
        Vehicle vehicle = new Vehicle(vehicleNo, "RED", 4,4);

        jedis.set(vehicleNo, mapper.writeValueAsString(vehicle));

        String vehicleString = jedis.get(vehicleNo);

        System.out.println("Vehicle: " + mapper.readValue(vehicleString, Vehicle.class));

        jedis.close();
    }
}
```  

- Jackson 에서 Json 변환을 수행하는 클래스는 ObjectMapper 이다.
	- writeValueAsString() 메서드는 객체를 Json String 으로 변환하는 메서드이다.
	- readValue() 메서드는 문자열, 변환할 객체의 클래스를 인수로 넘기면 객체로 변환해주는 메서드이다.
	
## RedisTemplate 
- RedisTemplate 는 Redis Client 에 대한 동작과 Redis API 사용에 대한 부분을 Template 화 시킨 것이다.
- 실행 도중 발생한 예외를 Spring DataAccessException 계열 예외로 주기 때문에 기존 데이터 액세스 코드와도 호환이 가능하다.
- Spring Transaction 도 사용 가능하다.
- RedisTemplate 을 이용하기 위해 RedisConnectionFactory 로 Redis 에 연결해야 한다.
- RedisConnectionFactory 의 구현체는 여러가지 있지만, 그 중 JedisConnectionFactory 를 사용한다.

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Vehicle> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Vehicle> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
		template.setKeySerializer(new StringRedisSerializer());
		template.setValueSerializer(new StringRedisSerializer());
        return template;
    }

    // spring-data-redis 2 이상
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName("localhost");
        redisStandaloneConfiguration.setPort(6379);
//        redisStandaloneConfiguration.setPassword(RedisPassword.of("password"));

        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(redisStandaloneConfiguration);
        return jedisConnectionFactory;
    }
    
    // spring-data-redis 2 미만    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory();
        jedisConnectionFactory.setHostName("35.227.20.44");
        jedisConnectionFactory.setPort(6379);
        return jedisConnectionFactory;
    }
}
```  

- redisTemplate() 메서드가 반환하는 RedisTemplate 는 Generic 형이기 때문에 Key, Value 를 String, Vehicle 로 지정했다.
	- 객체 조회/저장 시에 발생하는 형변환은 RedisTemplate 이 RedisSerializer 인터페이스 구현체를 사용하여 처리 한다.
- template 설정에서 setKeySerializer(), setValueSerializer() 메서드로 설정을 하지 않을 경우, redis-cli 에서 조회할때 key 혹은 value 값이 `/xac/xed/x00/x15` 와 같은 식으로 나온다.([관련링크](https://stackoverflow.com/questions/31608394/get-set-value-from-redis-using-redistemplate))

이름|설명
---|---
GenericToStringSerializer|String->byte[] 직렬화, Spring ConversionService 를 이용해 byte[] 로 변환하기 전, 객체를 String 으로 변환한다.
Jackson2JsonRedisSerializer|잭슨2 ObjectMapper 로 Json 을 읽고 쓴다.
JacksonJsonRedisSerializer|잭슨 ObjectMapper 로 Json 을 읽고 쓴다.
OxmSerializer|Spring Marshaller 및 Unmarshaller 로 XML 을 읽고 쓴다.
StringRedisSerializer|String->byte[] 직렬화  

- RedisTemplate 을 사용한 예제는 아래와 같다.

```java
public class Main {
    public static void main(String[] args) throws Exception {
        ApplicationContext context = new AnnotationConfigApplicationContext(RedisConfig.class);

        @SuppressWarnings("unchecked")
        RedisTemplate<String, Vehicle> template = context.getBean(RedisTemplate.class);

        final String vehicleNo = "TEM0001";
        Vehicle vehicle = new Vehicle(vehicleNo, "RED", 4,4);
        template.opsForValue().set(vehicleNo, vehicle);
        System.out.println("Vehicle: " + template.opsForValue().get(vehicleNo));
    }
}
```  

- Jackson 라이브러리를 사용한 하기위해서는 RedisSerializer 를 Jackson2JsonRedisSerializer 혹은 JacksonJsonRedisSerializer 를 사용해야 한다.

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Vehicle> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Vehicle> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setDefaultSerializer(new Jackson2JsonRedisSerializer(Vehicle.class));
        return template;
    }
    
    // ...
}
```  

- 설정 클래스를 위와 같이 변경하면 RedisTemplate 은 Jackson ObjectMapper 를 사용해서 Serialize/UnSerialize 를 수행한다.
- RedisTemplate 의 enableTransactionSupport 프로퍼티를 true 로 설정하면 Redis 에 트랜잭션을 사용할 수 있다.
	- 트랜잭션을 커밋하면 내부 트랜잭션 처리는 RedisTemplate 에서 처리 해준다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
[Spring에서 Redis 설정](https://yookeun.github.io/java/2017/05/21/spring-redis/)  
