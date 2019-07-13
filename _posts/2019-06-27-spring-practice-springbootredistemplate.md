--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Redis 기본 사용기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Redis 기본 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Redis
    - Spring Boot
    - Spring Data Redis
---  

# 목표
- Spring Boot 에서 Redis 를 사용하는 기본 방법에 대해 익힌다.
- Redis 에서 제공하는 자료구조에 대한 동작을 익힌다.
- Redis 의 Pub/Sub 동작에 대해 익힌다.

# 방법
- Spring Boot 의존성인 `spring-boot-starter-data-redis` 를 사용한다.
- Redis 관련 추상화된 클래스들을 사용해서 동작을 구현한다.

# 예제
- 완성된 프로젝트의 구조는 아래와 같다.

![그림 1]({{site.baseurl}}/img/practice-springbootredistemplate-1.png)

- pom.xml 에서 사용한 의존성은 아래와 같다.

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-json</artifactId>
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
</dependencies>
```  

- JavaConfig 파일은 아래와 같다.

```java
@Configuration
@PropertySource("classpath:redis.properties")
public class RedisConfiguration {

    @Value("${spring.redis.port}")
    private int redisPort;
    @Value("${spring.redis.host}")
    private String redisHost;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(this.redisHost);
        redisStandaloneConfiguration.setPort(this.redisPort);

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

- RedisConnectionFactory 객체 생성은 LettuceConnectionFactory, JedisConnectionFactory 모두 가능하다.
- `classpath:redis.properties` 파일에 Redis 설정관련 프로퍼티 값을 설정해 두고 JavaConfig 파일에서 불러와서 설정을 수행한다.
	- 기본 프로퍼티 파일인 `application.properties` 에 값이 있을 경우 별도로 `PropertySource` 로 프로퍼티 파일의 경로 설정을 해줄 필요는 없다.
	
	```properties
	spring.redis.port=6379
    spring.redis.host=localhost
	```  
	
- Serialize(직렬화) 설정은 키의 경우 `StringRedisSerializer` 를 사용하고 값의 경우 `GenericJackson2JsonRedisSerializer` 를 사용한다.
	- 각 도메인(모델) 마다 RedisTemplate 을 관리해야 할 경우 아래와 같은 방법으로 사용할 수 있다.
	
		```java
		
	    @Bean
	    public RedisTemplate<String, Device> redisTemplate() {
	        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
	        redisTemplate.setConnectionFactory(this.redisConnectionFactory());
	        redisTemplate.setKeySerializer(new StringRedisSerializer());
	        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<Device>(Device.class));
	
	        return redisTemplate;
	    }
		```  
	
	- Redis 에 저장하는 값이 문자열만 있을 경우 값의 직렬화도 `StringRedisSerializer` 를 사용해도 무방하다. 하지만 키의 경우 `GenericJackson2JsonRedisSerializer` 를 사용하게 되면 예상과 다른 키 형식이 Redis 에 저장되므로 주의해야 한다.

- Spring Data Redis 에서는 아래와 같은 오퍼레이션을 제공한다.

	Interface|Desc
	---|---
	GeoOperations|좌표 자료구조 데이터 관련 연산
	HashOperations|해시 자료구조 데이터 관련 연산
	HyperLogLogOperations|Redis 로그 자료구조 데이터  관련 연산
	ListOperations|리스트 자료구조 데이터 관련 연산
	SetOperations|집합 자료구조 데이터 관련 연산
	ValueOperations|값 형태의 데이터 관련 연산
	ZSetOperations|정렬된 집합 자료구조 데이터 관련 연산
	BoundGeoOperations|설정된 키에 대한 좌표 자료구조 데이터 관련 연산
	BoundHashOperations|설정된 키에 대한 해시 자료구조 데이터 관련 연산
	BoundListOperations|설정된 키에 대한 리스트 자료구조 데이터 관련 연산
	BoundSetOperations|설정된 키에 대한 집합 자료구조 데이터 관련 연산
	BoundValueOperations|설정된 키에 대한 값 형태의 데이터 관련 연산
	BoundZSetOperations|설정된 키에 대한 정렬된 집합 자료구조 데이터 관련 연산

- Redis 에 저장할 객체인 `Device` 는 아래와 같다.

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@ToString
@EqualsAndHashCode
public class Device implements Serializable{
    private String serial;
    private String name;
    private int price;
}
```  

- Redis 에서 제공하는 자료구조 및 사용 방법에 대한 테스트 코드는 아래와 같다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RedisOperationTest {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    private ValueOperations<String, Object> valueOperations;
    private ListOperations<String, Object> listOperations;
    private HashOperations<String, String, Object> hashOperations;
    private SetOperations<String, Object> setOperations;
    private ZSetOperations<String, Object> zSetOperations;

    @Before
    public void init() {
        this.valueOperations = this.redisTemplate.opsForValue();
        this.listOperations = this.redisTemplate.opsForList();
        this.hashOperations = this.redisTemplate.opsForHash();
        this.setOperations = this.redisTemplate.opsForSet();
        this.zSetOperations = this.redisTemplate.opsForZSet();
    }

    @Test
    public void testValue() {
        Device d = new Device("s1", "n1", 1);
        this.valueOperations.set(d.getSerial(), d);

        d = new Device("s2", "n2", 2);
        this.valueOperations.set(d.getSerial(), d);

        d = new Device("s3", "n2", 3);
        this.valueOperations.set(d.getSerial(), d);

        Assert.assertEquals(new Device("s1", "n1", 1), this.valueOperations.get("s1"));
        Assert.assertEquals(new Device("s2", "n2", 2), this.valueOperations.get("s2"));
        Assert.assertEquals(new Device("s3", "n2", 3), this.valueOperations.get("s3"));
    }

    @Test
    public void testList() {
        String key = "test-list";
        Device d = new Device("s1", "n1", 1);
        this.listOperations.leftPush(key, d);

        d = new Device("s2", "n2", 2);
        this.listOperations.leftPush(key, d);

        d = new Device("s3", "n2", 3);
        this.listOperations.leftPush(key, d);

        Assert.assertEquals(3, Integer.parseInt(this.listOperations.size(key) + ""));
        Assert.assertEquals(new Device("s3", "n2", 3), this.listOperations.leftPop(key));
        Assert.assertEquals(new Device("s2", "n2", 2), this.listOperations.leftPop(key));
        Assert.assertEquals(new Device("s1", "n1", 1), this.listOperations.leftPop(key));
    }

    @Test
    public void testHash() {
        String key = "test-hash";
        Device d = new Device("s1", "n1", 1);
        this.hashOperations.put(key, d.getSerial(), d);

        d = new Device("s2", "n2", 2);
        this.hashOperations.put(key, d.getSerial(), d);

        d = new Device("s3", "n2", 3);
        this.hashOperations.put(key, d.getSerial(), d);

        Assert.assertEquals(3, Integer.parseInt(this.hashOperations.size(key) + ""));
        Assert.assertTrue(this.hashOperations.hasKey(key, "s1"));
        Assert.assertTrue(this.hashOperations.hasKey(key, "s2"));
        Assert.assertTrue(this.hashOperations.hasKey(key, "s2"));
        Assert.assertEquals(new Device("s1", "n1", 1), this.hashOperations.get(key, "s1"));
        Assert.assertEquals(new Device("s2", "n2", 2), this.hashOperations.get(key, "s2"));
        Assert.assertEquals(new Device("s3", "n2", 3), this.hashOperations.get(key, "s3"));
    }

    @Test
    public void testSet() {
        String key = "test-set";
        Device d = new Device("s1", "n1", 1);
        this.setOperations.add(key, d);

        d = new Device("s2", "n2", 2);
        this.setOperations.add(key, d);

        d = new Device("s3", "n2", 3);
        this.setOperations.add(key, d);

        Assert.assertEquals(3, Integer.parseInt(this.setOperations.size(key) + ""));
        Assert.assertTrue(this.setOperations.isMember(key, new Device("s1", "n1", 1)));
        Assert.assertTrue(this.setOperations.isMember(key, new Device("s2", "n2", 2)));
        Assert.assertTrue(this.setOperations.isMember(key, new Device("s3", "n2", 3)));
    }

    @Test
    public void testZSet() {
        String key = "test-zset";

        this.zSetOperations.add(key, "s1", 11);
        this.zSetOperations.add(key, "s2", 22);
        this.zSetOperations.add(key, "s3", 33);
        this.zSetOperations.add(key, "s4", 44);
        this.zSetOperations.add(key, "s5", 55);

        Assert.assertEquals(5, Integer.parseInt(this.zSetOperations.size(key) + ""));

        Set<Object> set = this.zSetOperations.range(key, 1, 3);
        Assert.assertEquals(3, set.size());
        Assert.assertTrue(set.contains("s2"));
        Assert.assertTrue(set.contains("s3"));
        Assert.assertTrue(set.contains("s4"));

        set = this.zSetOperations.rangeByScore(key, 22, 44);
        Assert.assertEquals(3, set.size());
        Assert.assertTrue(set.contains("s2"));
        Assert.assertTrue(set.contains("s3"));
        Assert.assertTrue(set.contains("s4"));
    }    

    @Test
    public void testRedisCallback() {
        String key = "callback1";

        this.redisTemplate.delete(key);

        List<Object> list = this.redisTemplate.execute(new SessionCallback<List<Object>>() {
            @Override
            public List<Object> execute(RedisOperations redisOperations) throws DataAccessException {
                redisOperations.multi();
                redisOperations.opsForValue().increment(key);
                redisOperations.opsForValue().increment(key);
                redisOperations.opsForValue().increment(key);

                return redisOperations.exec();
            }
        });

        Assert.assertNotNull(list);

        int count = Integer.parseInt(this.valueOperations.get(key) + "");
        Assert.assertEquals(3, count);
    }
}
```  

- `testRedisCallback` 의 경우 `execute` 메서드의 인자값으로 `SessionCallback` 또는 `RedisCallback` 인터페이스를 사용해서 Redis Server 에 직접 명령어를 내릴 수 있다.
	- 명렁어를 한번에 수행시키는 `multi` 연산시에 활용 할 수 있다.
	- 한 Key 를 기준으로 값을 동기화를 시켜야 할때, Key 의 변경 여부를 확인하는 `watch` 연산에 활용할 수 있다.

- Redis 의 메시지 Pub/Sub 을 테스트하기 위해 `RedisConfiguration` 파일에 아래 설정을 추가한다.

```java
@Bean
public MessageListener messageListener() {
    return new MessageListenerAdapter(new RedisMessageSubscriber());
}

@Bean
public RedisMessageListenerContainer redisMessageListenerContainer() {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(this.redisConnectionFactory());
    container.addMessageListener(this.messageListener(), this.topic());

    return container;
}

@Bean
public ChannelTopic topic() {
    return new ChannelTopic("test-event");
}
```  

- 어디선가 Publish 할 메시지를 Subscribe 할 클래스를 `MessageListener` 설정에 추가한다.
- `RedisMessageListenerContainer` 를 통해 Subscribe 할 `MessageListener` 와 생성한 `Topic`(test-event) 를 설정해준다.
	
- Subscribe 역할을 하는 `MessageListener` 클래스는 아래와 같다.

```java
@Service
@Log
public class RedisMessageSubscriber implements MessageListener {
    public static List<String> messageList = new ArrayList<>();

    @Override
    public void onMessage(Message message, byte[] bytes) {
        log.info("receive message : " + message.toString());
        messageList.add(message.toString());
    }
}
```  

- log 를 통해 Subscribe 한 메시지를 콘솔에 찍어주고, 이를 전역변수인 `messageList` 에 추가해 준다.

- Pub/Sub 을 테스트할 코드는 아래와 같다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RedisPubSubTest {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Test
    public void test() {
        this.redisTemplate.convertAndSend("test-event", "event 1");
        this.redisTemplate.convertAndSend("test-event", "event 2");
        this.redisTemplate.convertAndSend("test-event", "event 3");
    }
}
```  

- `RedisTemplate` 의 `convertAndSend` 를 통해 인자 값으로 설정된 토픽의 값과, 메시지로 전달할 값을 넣어 주면된다.
- 실행을 하면 아래와 같은 로그가 출력되는 것을 확인 할 수 있다.

```
c.e.s.service.RedisMessageSubscriber     : receive message : "event 1"
c.e.s.service.RedisMessageSubscriber     : receive message : "event 3"
c.e.s.service.RedisMessageSubscriber     : receive message : "event 2"
```  
	
---
## Reference
[Spring Data Redis](https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/)
[Redis설치 및 SpringBoot로 Redis 서버 접속 설정](https://imjehoon.github.io/redis_spring_config/)
[spring-boot-starter-data-redis 사용기](https://jeong-pro.tistory.com/175)
[[스프링부트] SpringBoot 개발환경 구성 #3 - Redis 설정](https://yonguri.tistory.com/entry/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-SpringBoot-%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%84%B1-3-Redis-%EC%84%A4%EC%A0%95)
