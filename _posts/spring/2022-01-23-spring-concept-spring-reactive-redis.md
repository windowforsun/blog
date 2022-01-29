--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Reactive Redis"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 Reactive Redis 를 사용하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Redis
    - Spring Reactive Redis
    - Reactor
    - Reactive Stream
    - ReactiveValueOperations
    - ReactiveListOperations
    - ReactiveHashOperations
    - ReactiveSetOperations
    - ReactiveZSetOperations
    - ReactiveGeoOperations
    - ReactiveHyperLogLogOperations
toc: true
use_math: true
---  

## Spring Data Redis Reactive
`Spring Data` 에서 공식적으로 지원하는 `Reactive` 라이브러리 중 `Redis` 를 `Reactive` 하게 사용가능한 `Spring Data Redis Reactive` 에 대해서 알아본다. 
`WebFlux` 를 사용한다면 외부 저장소에 대해서 `Spring Data` 가 `Reactive Stream` 을 지원하는 지는 중요한 부분이라고 할 수 있다. 
애플리케이션에서 모든 흐름은 `Reactive` 하게 구성됐지만, 매번 접근해 데이터 `CRUD` 연산을 수행하는 부분이 `Reactive` 하지 못하면, 
`Reactive Stream` 전체에 악영향을 가져다 줄 수 있기 때문이다.  

`Reactive Redis` 를 사용하면 여러 데이터를 각 명령으로 저장하더라도, 
실제 데이터가 저장되는 순서가 상이할 수 있다는 점이 있지만 애플리케이션 전체의 자원을 효율적으로 사용할 수 있다는 큰 장점이 있다.  

테스트 환경아래와 같다. 
- `Spring Boot 2.6.2`
- `Redis 6.2.0`

### build.gradle
`Reactive Redis` 를 사용하기 위해서는 `Spring Data Redis Reactive` 의존성만 추가해 주면 된다.  

```groovy
dependencies {
    implementation 'org.springframework:spring-context'
    implementation 'org.springframework.boot:spring-boot-autoconfigure'
    implementation 'org.springframework.boot:spring-boot-starter-json'
    implementation 'org.slf4j:slf4j-api'
    implementation 'io.projectreactor:reactor-core:3.4.6'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    // Reactive Redis
    implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'
    testCompileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testAnnotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'


    testImplementation 'io.projectreactor:reactor-test'
    testImplementation 'io.projectreactor.tools:blockhound:1.0.6.RELEASE'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testCompile "org.testcontainers:testcontainers:1.14.3"
    testCompile "org.testcontainers:junit-jupiter:1.14.3"
}
```  

### Redis 환경 구성
`Redis` 환경은 `Docker` 를 기반으로 하는 `TestContainers` 라이브러리를 사용한다. 
`JUnit` 테스트가 수행될때 테스트에 필요한 외부 환경을 `Docker` 기반으로 코드를 사용해서 구성할 수 있기 때문이다.  

```java
@ContextConfiguration(initializers = DockerRedisTest.Initializer.class)
@Testcontainers
@TestConfiguration
public interface DockerRedisTest {
    @Container
    GenericContainer redis = new GenericContainer("redis:6.2.0-alpine").withExposedPorts(6379);

    class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            TestPropertyValues.of(
                    "spring.redis.host=" + redis.getHost(),
                    "spring.redis.port=" + redis.getFirstMappedPort()
            ).applyTo(applicationContext.getEnvironment());
        }
    }
}
```  

실제 사용은 테스트 클래스에서 `implements` 를 사용해서 함께 선언해 준다.  

```java
public class TestClass implements DockerRedisTest {
    // ..
}
```  

### DefaultReactiveRedis
위와 같이 의존성만 추가해주면 `Spring AutoConfigure` 가 `spring.redis.host`, `spring.redis.port` 프로퍼티를 읽어, 
자동으로 기본 `ReactiveRedisConnectionFactory`, `ReactiveRedisTemplate` 등과 같이 사용에 필요한 빈들을 자동 생성해 준다.  

`Redis` 에서 데이터를 저장할때는 주로 객체를 `Serialize` 한 결과를 저장하고, 
이를 다시 애플리케이션에서 로드 할때는 `Deserialize` 된 객체의 인스턴스를 생성한다.  

`ReactiveRedisTemplate`(`RedisTemplate` 동일)은 `Key`, `Value`, `Hashd`, `List` 등 데이터 구조에 따라 `Serializer` 를 등록할 수 있다. 
자동으로 생성되는 `ReactiveRedisTemplate` 은 기본으로 `JdkSerializationRedisSerializer` 를 사용하는데, 
실제 저장되는 값을 보면 클래스와 필드 정보 등의 부가적인 데이터들도 함께 저장된다.  

자동 생성되는 `ReactiveRedisConnectionFactory` 에서 사용하는 `Redis Client` 는 기본으로 `Lettuce` 를 사용한다.   

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class DefaultReactiveRedisTest implements DockerRedisTest{
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;

    @Test
    public void basicTest() {
        Mono<Boolean> setResult = this.reactiveRedisTemplate.opsForValue().set("testKey", "testValue");

        StepVerifier
                .create(setResult)
                .expectNext(true)
                .verifyComplete();

        Mono<String> getResult = this.reactiveRedisTemplate.opsForValue().get("testKey");

        StepVerifier
                .create(getResult)
                .expectNext("testValue")
                .verifyComplete();
    }
}
```  

### CustomReactiveRedis
`Spring Data Redis Reactive` 를 사용한다고 해서 기존 `Spring Data Redis` 와 구성 및 설정은 거의 동일하다. 
이번에는 직접 `ReactiveRedisConnectionFactory` 와 `ReactiveRedisTemplate` 을 빈으로 등록해 커스텀한 설정을 해보도록 한다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class CustomReactiveRedisTest implements DockerRedisTest {
    @TestConfiguration
    public static class ReactiveRedisConfig {
        @Value("${spring.redis.host}")
        private String redisHost;
        @Value("${spring.redis.port}")
        private int redisPort;

        @Bean
        public ReactiveRedisConnectionFactory reactiveRedisConnectionFactory() {
            return new LettuceConnectionFactory(this.redisHost, this.redisPort);
        }

        @Bean
        public ReactiveRedisTemplate<String, Object> reactiveRedisTemplate(ReactiveRedisConnectionFactory reactiveRedisConnectionFactory) {
            StringRedisSerializer keySerializer = new StringRedisSerializer();
            GenericJackson2JsonRedisSerializer valueSerializer = new GenericJackson2JsonRedisSerializer();
            RedisSerializationContext.RedisSerializationContextBuilder<String, Object> builder = RedisSerializationContext.newSerializationContext(keySerializer);
            RedisSerializationContext<String, Object> context = builder.value(valueSerializer).build();

            return new ReactiveRedisTemplate<>(reactiveRedisConnectionFactory, context);
        }
    }

    @Autowired
    private ReactiveRedisTemplate<String, Object> reactiveRedisTemplate;

    @Test
    public void basicTest() {
        Mono<Boolean> setResult = this.reactiveRedisTemplate.opsForValue().set("testKey", "testValue");

        StepVerifier
                .create(setResult)
                .expectNext(true)
                .verifyComplete();

        Mono<Object> getResult = this.reactiveRedisTemplate.opsForValue().get("testKey");

        StepVerifier
                .create(getResult)
                .expectNext("testValue")
                .verifyComplete();
    }
}
```  


### ReactiveValueOperations
`Redis` 에서 가장 기본적인 `key-value` 구조 데이터를 조작하는 인터페이스이다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveValueOperationsTest implements DockerRedisTest {
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveValueOperations<String, String> reactiveValueOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveValueOperations = this.reactiveRedisTemplate.opsForValue();
        this.reactiveRedisTemplate.scan().subscribe(o -> {
            this.reactiveRedisTemplate.delete(o).subscribe();
        });
    }

    @Test
    public void set_get() throws Exception {
        Mono<Boolean> setResult = this.reactiveValueOperations.set("testKey", "testValue");
        StepVerifier
                .create(setResult)
                .expectNext(true)
                .verifyComplete();

        Mono<String> getResult = this.reactiveValueOperations.get("testKey");
        StepVerifier
                .create(getResult)
                .expectNext("testValue")
                .verifyComplete();

        setResult = this.reactiveValueOperations.set("testKey", "newTestValue", Duration.ofSeconds(1));
        StepVerifier
                .create(setResult)
                .expectNext(true)
                .verifyComplete();

        Thread.sleep(1000);
        getResult = this.reactiveValueOperations.get("testKey");
        StepVerifier.create(getResult)
                .verifyComplete();
    }

    @Test
    public void delete() {
        this.reactiveValueOperations.set("existsKey", "value").subscribe();

        Mono<Boolean> existsKeyDelete = this.reactiveValueOperations.delete("existsKey");

        StepVerifier
                .create(existsKeyDelete)
                .expectNext(true)
                .verifyComplete();

        Mono<Boolean> notExistsKeyDelete = this.reactiveValueOperations.delete("notExistsKey");

        StepVerifier
                .create(notExistsKeyDelete)
                .expectNext(false)
                .verifyComplete();
    }

    @Test
    public void append() {
        Mono<Long> appendResult = this.reactiveValueOperations.append("testKey", "testValue");
        StepVerifier
                .create(appendResult)
                .expectNext((long) "testValue".length())
                .verifyComplete();

        appendResult = this.reactiveValueOperations.append("testKey", " newValue");
        StepVerifier
                .create(appendResult)
                .expectNext((long) "testValue newValue".length())
                .verifyComplete();
    }

    @Test
    public void increment_decrement() {
        Mono<Long> decrementMono = this.reactiveValueOperations.decrement("testKey");
        StepVerifier
                .create(decrementMono)
                .expectNext(-1L)
                .verifyComplete();

        Mono<Long> incrementMono = this.reactiveValueOperations.increment("testKey");
        StepVerifier
                .create(incrementMono)
                .expectNext(0L)
                .verifyComplete();

        incrementMono = this.reactiveValueOperations.increment("testKey");
        StepVerifier
                .create(incrementMono)
                .expectNext(1L)
                .verifyComplete();

        decrementMono = this.reactiveValueOperations.decrement("testKey");
        StepVerifier
                .create(decrementMono)
                .expectNext(0L)
                .verifyComplete();
    }

    @Test
    public void getAndDelete_getAndExpire_getAndPersist() throws Exception {
        this.reactiveValueOperations.set("testKey", "a").subscribe();

        // Spring 2.6, Redis 6.2
        Mono<String> getAndDeleteMono = this.reactiveValueOperations.getAndDelete("testKey");
        StepVerifier
                .create(getAndDeleteMono)
                .expectNext("a")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveValueOperations.get("testKey"))
                .verifyComplete();

        this.reactiveValueOperations.set("testKey", "a").subscribe();
        // Spring 2.6, Redis 6.2
        Mono<String> getAndExpireMono = this.reactiveValueOperations.getAndExpire("testKey", Duration.ofMillis(500));
        StepVerifier
                .create(getAndExpireMono)
                .expectNext("a")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveValueOperations.get("testKey"))
                .expectNext("a")
                .verifyComplete();
        Thread.sleep(600);
        StepVerifier
                .create(this.reactiveValueOperations.get("testKey"))
                .verifyComplete();

        this.reactiveValueOperations.set("testKey", "a", Duration.ofMillis(500)).subscribe();
        // Spring 2.6, Redis 6.2
        Mono<String> getAndPersistMono = this.reactiveValueOperations.getAndPersist("testKey");
        StepVerifier
                .create(getAndPersistMono)
                .expectNext("a")
                .verifyComplete();
        Thread.sleep(600);
        StepVerifier
                .create(this.reactiveValueOperations.get("testKey"))
                .expectNext("a")
                .verifyComplete();
    }
}
```  

### ReactiveListOperations
`Redis` 에서 `Key-List(e, e, ..)` 구조의 데이터를 조작할 수 있는 인터페이스이다. 
`List` 내에 순서는 보장되고 중복데이터 저장도 가능하다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveListOperationsTest implements DockerRedisTest {
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveListOperations<String, String> reactiveListOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveListOperations = this.reactiveRedisTemplate.opsForList();
        this.reactiveRedisTemplate.scan().subscribe(o -> {
            this.reactiveRedisTemplate.delete(o);
        });
    }

    @Test
    public void rightPush_leftPush_rightPop_leftPop() {
        // rightPush1
        Mono<Long> rightPushMono = this.reactiveListOperations.rightPush("testKey", "rightPush1");
        StepVerifier
                .create(rightPushMono)
                .expectNext(1L)
                .verifyComplete();

        // leftPush1, rightPush1
        Mono<Long> leftPushMono = this.reactiveListOperations.leftPush("testKey", "leftPush1");
        StepVerifier
                .create(leftPushMono)
                .expectNext(2L)
                .verifyComplete();

        // leftPush1, leftPush2, rightPush1
        rightPushMono = this.reactiveListOperations.leftPush("testKey", "rightPush1", "leftPush2");
        StepVerifier
                .create(rightPushMono)
                .expectNext(3L)
                .verifyComplete();

        // leftPush2, rightPush1
        Mono<String> leftPopMono = this.reactiveListOperations.leftPop("testKey");
        StepVerifier
                .create(leftPopMono)
                .expectNext("leftPush1")
                .verifyComplete();

        // leftPush2
        Mono<String> rightPopMono = this.reactiveListOperations.rightPop("testKey");
        StepVerifier
                .create(rightPopMono)
                .expectNext("rightPush1")
                .verifyComplete();

        rightPopMono = this.reactiveListOperations.rightPop("testKey");
        StepVerifier
                .create(rightPopMono)
                .expectNext("leftPush2")
                .verifyComplete();
    }

    @Test
    public void rightPushAll_leftPushAll_range_size() {
        // rightA, rightB, rightC
        Mono<Long> rightPushAllMono = this.reactiveListOperations.rightPushAll("testKey", Arrays.asList("rightA", "rightB", "rightC"));
        StepVerifier
                .create(rightPushAllMono)
                .expectNext(3L)
                .verifyComplete();

        Mono<Long> sizeMono = this.reactiveListOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(3L)
                .verifyComplete();

        // leftC, leftB, leftA, rightA, rightB, rightC
        Mono<Long> leftPushAllMono = this.reactiveListOperations.leftPushAll("testKey", Arrays.asList("leftA", "leftB", "leftC"));
        StepVerifier
                .create(leftPushAllMono)
                .expectNext(6L)
                .verifyComplete();

        sizeMono = this.reactiveListOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(6L)
                .verifyComplete();

        Flux<String> rangeFlux = this.reactiveListOperations.range("testKey", 0, -1);
        StepVerifier
                .create(rangeFlux)
                .expectNext("leftC", "leftB", "leftA", "rightA", "rightB", "rightC")
                .verifyComplete();

        rangeFlux = this.reactiveListOperations.range("testKey", 0, 0);
        StepVerifier
                .create(rangeFlux)
                .expectNext("leftC")
                .verifyComplete();

        rangeFlux = this.reactiveListOperations.range("testKey", 2, 4);
        StepVerifier
                .create(rangeFlux)
                .expectNext("leftA", "rightA", "rightB")
                .verifyComplete();
    }

    @Test
    public void set_remove() {
        // a, a, a, a, a, a, a, a
        this.reactiveListOperations.rightPushAll("testKey", Arrays.asList("a", "a", "a", "a", "a", "a", "a", "a")).subscribe();

        // a, b, a, a, a, a, a, a
        Mono<Boolean> setMono = this.reactiveListOperations.set("testKey", 1, "b");
        StepVerifier
                .create(setMono)
                .expectNext(true)
                .verifyComplete();

        setMono = this.reactiveListOperations.set("testKey", 999, "z");
        StepVerifier
                .create(setMono)
                .expectErrorMatches(throwable -> throwable.getMessage().contains("ERR index out of range"))
                .verify();

        // a, b, c, a, a, a, a, a
        this.reactiveListOperations.set("testKey", 2, "c").subscribe();
        // a, b, c, b, a, a, a, a
        this.reactiveListOperations.set("testKey", 3, "b").subscribe();
        // a, b, c, b, c, a, a, a
        this.reactiveListOperations.set("testKey", 4, "c").subscribe();
        // a, b, c, b, c, c, a, a
        this.reactiveListOperations.set("testKey", 5, "c").subscribe();
        // a, b, c, b, c, c, b, a
        this.reactiveListOperations.set("testKey", 6, "b").subscribe();

        StepVerifier
                .create(this.reactiveListOperations.range("testKey", 0, -1))
                .expectNext("a", "b", "c", "b", "c", "c", "b", "a")
                .verifyComplete();

        // b, c, b, c, c, b, a
        Mono<Long> removeMono = this.reactiveListOperations.remove("testKey", 1, "a");
        StepVerifier
                .create(removeMono)
                .expectNext(1L)
                .verifyComplete();

        // b, c, b, c, c, b
        removeMono = this.reactiveListOperations.remove("testKey", -1, "a");
        StepVerifier
                .create(removeMono)
                .expectNext(1L)
                .verifyComplete();

        // c, c, c, b
        removeMono = this.reactiveListOperations.remove("testKey", 2, "b");
        StepVerifier
                .create(removeMono)
                .expectNext(2L)
                .verifyComplete();

        // b
        removeMono = this.reactiveListOperations.remove("testKey", 0, "c");
        StepVerifier
                .create(removeMono)
                .expectNext(3L)
                .verifyComplete();
    }

    @Test
    public void index_indexOf_lastIndexOf() {
        this.reactiveListOperations.rightPushAll("testKey", Arrays.asList("a", "b", "b", "c", "c", "c")).subscribe();

        Mono<String> indexMono = this.reactiveListOperations.index("testKey", 0L);
        StepVerifier
                .create(indexMono)
                .expectNext("a")
                .verifyComplete();

        indexMono = this.reactiveListOperations.index("testKey", -1);
        StepVerifier
                .create(indexMono)
                .expectNext("c")
                .verifyComplete();

        // Redis 6.0.6
        Mono<Long> indexOfMono = this.reactiveListOperations.indexOf("testKey", "b");
        StepVerifier
                .create(indexOfMono)
                .expectNext(1L)
                .verifyComplete();

        // Redis 6.0.6
        indexOfMono = this.reactiveListOperations.indexOf("testKey", "c");
        StepVerifier
                .create(indexOfMono)
                .expectNext(3L)
                .verifyComplete();

        // Redis 6.0.6
        Mono<Long> lastIndexOfMono = this.reactiveListOperations.lastIndexOf("testKey", "b");
        StepVerifier
                .create(lastIndexOfMono)
                .expectNext(2L)
                .verifyComplete();

        // 6.0.6
        lastIndexOfMono = this.reactiveListOperations.lastIndexOf("testKey", "c");
        StepVerifier
                .create(lastIndexOfMono)
                .expectNext(5L)
                .verifyComplete();
    }

    @Test
    public void move() {
        this.reactiveListOperations.rightPushAll("testKey1", Arrays.asList("a", "b", "c", "d")).subscribe();
        this.reactiveListOperations.rightPushAll("testKey2", Arrays.asList("1", "2", "3", "4")).subscribe();

        // Spring 2.6, Redis 6.2.0
        Mono<String> moveMono = this.reactiveListOperations.move(ListOperations.MoveFrom.fromHead("testKey1"), ListOperations.MoveTo.toTail("testKey2"));
        StepVerifier
                .create(moveMono)
                .expectNext("a")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveListOperations.range("testKey1", 0, -1))
                .expectNext("b", "c", "d")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveListOperations.range("testKey2", 0, -1))
                .expectNext("1", "2", "3", "4", "a")
                .verifyComplete();

        moveMono = this.reactiveListOperations.move(ListOperations.MoveFrom.fromTail("testKey2"), ListOperations.MoveTo.toHead("testKey3"));
        StepVerifier
                .create(moveMono)
                .expectNext("a")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveListOperations.range("testKey2", 0, -1))
                .expectNext("1", "2", "3", "4")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveListOperations.range("testKey3", 0, -1))
                .expectNext("a")
                .verifyComplete();

        moveMono = this.reactiveListOperations.move(ListOperations.MoveFrom.fromTail("testKey2"), ListOperations.MoveTo.toHead("testKey2"));
        StepVerifier
                .create(moveMono)
                .expectNext("4")
                .verifyComplete();
        StepVerifier
                .create(this.reactiveListOperations.range("testKey2", 0, -1))
                .expectNext("4", "1", "2", "3")
                .verifyComplete();
    }
}
```  

### ReactiveHashOperations
`Redis` 에서 `Key-Hash(hashKey1=hashValue, hashKey2=hashValue ...)` 구조의 데이터를 조작할 수 있는 인터페이스이다. 
`Hash` 내 원소의 순서는 보장되지 않지고, `hashKey` 는 중복 허용하지 않고 `hashValue` 는 중복이 가능하다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveHashOperationsTest implements DockerRedisTest {
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveHashOperations<String, String, String> reactiveHashOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveHashOperations = this.reactiveRedisTemplate.opsForHash();
        this.reactiveRedisTemplate.scan().subscribe(o -> {
            this.reactiveRedisTemplate.delete(o);
        });
    }

    @Test
    public void put_putAll_get_multiGet_hasKey() {
        Mono<Boolean> putMono = this.reactiveHashOperations.put("testKey", "a", "aValue");
        StepVerifier
                .create(putMono)
                .expectNext(true)
                .verifyComplete();

        Mono<Boolean> putAllMono = this.reactiveHashOperations.putAll("testKey", Map.of("b", "bValue", "a", "aValue2", "c", "cValue"));
        StepVerifier
                .create(putAllMono)
                .expectNext(true)
                .verifyComplete();

        Mono<String> getMono = this.reactiveHashOperations.get("testKey", "a");
        StepVerifier
                .create(getMono)
                .expectNext("aValue2")
                .verifyComplete();

        Mono<List<String>> multiGetMono = this.reactiveHashOperations.multiGet("testKey", Arrays.asList("b", "c"));
        StepVerifier
                .create(multiGetMono)
                .recordWith(ArrayList::new)
                .consumeNextWith(strings -> assertThat(strings, containsInAnyOrder("bValue", "cValue")))
                .verifyComplete();

        Mono<Boolean> hasKeyMono = this.reactiveHashOperations.hasKey("testKey", "a");
        StepVerifier
                .create(hasKeyMono)
                .expectNext(true)
                .verifyComplete();

        hasKeyMono = this.reactiveHashOperations.hasKey("testKey", "notExists");
        StepVerifier
                .create(hasKeyMono)
                .expectNext(false)
                .verifyComplete();
    }

    @Test
    public void increment() {
        Mono<Double> doubleIncrementMono = this.reactiveHashOperations.increment("testKey", "double", 1.1);
        StepVerifier
                .create(doubleIncrementMono)
                .expectNext(1.1d)
                .verifyComplete();

        Mono<Long> longIncrementMono = this.reactiveHashOperations.increment("testKey", "long", 11);
        StepVerifier
                .create(longIncrementMono)
                .expectNext(11L)
                .verifyComplete();

        doubleIncrementMono = this.reactiveHashOperations.increment("testKey", "double", 1.1);
        StepVerifier
                .create(doubleIncrementMono)
                .expectNext(2.2d)
                .verifyComplete();

        longIncrementMono = this.reactiveHashOperations.increment("testKey", "long", 11);
        StepVerifier
                .create(longIncrementMono)
                .expectNext(22L)
                .verifyComplete();
    }

    @Test
    public void entries_keys_values_scan() {
        this.reactiveHashOperations.putAll("testKey", Map.of("a", "aValue", "ab", "abValue", "c", "cValue")).subscribe();

        Flux<Map.Entry<String, String>> entriesFlux = this.reactiveHashOperations.entries("testKey");
        StepVerifier
                .create(entriesFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringStringEntry -> true)
                .consumeRecordedWith(entries ->  {
                    assertThat(entries, hasSize(3));
                    assertThat(entries, everyItem(oneOf(
                            Map.entry("a", "aValue"),
                            Map.entry("ab", "abValue"),
                            Map.entry("c", "cValue")
                    )));
                })
                .verifyComplete();

        Flux<String> keysFlux = this.reactiveHashOperations.keys("testKey");
        StepVerifier
                .create(keysFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(3));
                    assertThat(strings, everyItem(oneOf("a", "ab", "c")));
                })
                .verifyComplete();

        Flux<String> valuesFlux = this.reactiveHashOperations.values("testKey");
        StepVerifier
                .create(valuesFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(3));
                    assertThat(strings, everyItem(oneOf("aValue", "abValue", "cValue")));
                })
                .verifyComplete();

        Flux<Map.Entry<String, String>> scanFlux = this.reactiveHashOperations.scan("testKey");
        StepVerifier
                .create(scanFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringStringEntry -> true)
                .consumeRecordedWith(entries ->  {
                    assertThat(entries, hasSize(3));
                    assertThat(entries, everyItem(oneOf(
                            Map.entry("a", "aValue"),
                            Map.entry("ab", "abValue"),
                            Map.entry("c", "cValue")
                    )));
                })
                .verifyComplete();

        scanFlux = this.reactiveHashOperations.scan("testKey", ScanOptions.scanOptions().match("*a*").count(2).build());
        StepVerifier
                .create(scanFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringStringEntry -> true)
                .consumeRecordedWith(entries ->  {
                    assertThat(entries, hasSize(2));
                    assertThat(entries, everyItem(oneOf(
                            Map.entry("a", "aValue"),
                            Map.entry("ab", "abValue")
                    )));
                })
                .verifyComplete();
    }

    @Test
    public void size_hasKey_randomKey() {
        this.reactiveHashOperations.putAll("testKey", Map.of("a", "aValue", "b", "bValue", "c", "cValue")).subscribe();

        Mono<Long> sizeMono = this.reactiveHashOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(3L)
                .verifyComplete();

        Mono<Boolean> hasKeyMono = this.reactiveHashOperations.hasKey("testKey", "a");
        StepVerifier
                .create(hasKeyMono)
                .expectNext(true)
                .verifyComplete();

        hasKeyMono = this.reactiveHashOperations.hasKey("testKey", "notExists");
        StepVerifier
                .create(hasKeyMono)
                .expectNext(false)
                .verifyComplete();
    }

    @Test
    public void randomKey_randomEntry_randomKeys_randomEntries() {
        this.reactiveHashOperations.putAll("testKey", Map.of("a", "aValue", "b", "bValue", "c", "cValue", "d", "dValue")).subscribe();

        // Spring 2.6, Redis 6.2
        Mono<String> randomKeyMono = this.reactiveHashOperations.randomKey("testKey");
        StepVerifier
                .create(randomKeyMono)
                .consumeNextWith(s -> {
                    assertThat(s, oneOf("a", "b", "c", "d"));
                })
                .verifyComplete();

        // Spring 2.6, Redis 6.2
        Mono<Map.Entry<String, String>> randomEntryMono = this.reactiveHashOperations.randomEntry("testKey");
        StepVerifier
                .create(randomEntryMono)
                .consumeNextWith(stringStringEntry -> {
                    assertThat(stringStringEntry, oneOf(
                            Map.entry("a", "aValue"),
                            Map.entry("b", "bValue"),
                            Map.entry("c", "cValue"),
                            Map.entry("d", "dValue")
                    ));
                })
                .verifyComplete();

        // Spring 2.6, Redis 6.2
        Flux<String> randomKeysFlux = this.reactiveHashOperations.randomKeys("testKey", 3);
        StepVerifier
                .create(randomKeysFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(3));
                    assertThat(strings, everyItem(oneOf("a", "b", "c", "d")));
                })
                .verifyComplete();

        // Spring 2.6, Redis 6.2
        Flux<Map.Entry<String, String>> randomEntriesFlux = this.reactiveHashOperations.randomEntries("testKey", 3);
        StepVerifier
                .create(randomEntriesFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringStringEntry -> true)
                .consumeRecordedWith(entries -> {
                    assertThat(entries, hasSize(3));
                    assertThat(entries, everyItem(oneOf(
                            Map.entry("a", "aValue"),
                            Map.entry("b", "bValue"),
                            Map.entry("c", "cValue"),
                            Map.entry("d", "dValue")
                    )));
                })
                .verifyComplete();
    }
}
```  

### ReactiveSetOperations
`Redis` 에서 `Key-Set(value1, value2 ...)` 구조의 데이터를 조작할 수 있는 인터페이스이다. 
집합 관련 연산이 제공되고(`difference`, `intersect`, `union`) 내부 원소는 순서 보장이 되지 않고, 중복값은 허용하지 않는다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveSetOperationsTest implements DockerRedisTest {
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveSetOperations<String, String> reactiveSetOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveSetOperations = this.reactiveRedisTemplate.opsForSet();
        this.reactiveRedisTemplate.scan().subscribe(o -> {
            this.reactiveRedisTemplate.delete(o);
        });
    }

    @Test
    public void add_members_pop_size() {

        Mono<Long> addMono = this.reactiveSetOperations.add("testKey", "a");
        StepVerifier
                .create(addMono)
                .expectNext(1L)
                .verifyComplete();

        addMono = this.reactiveSetOperations.add("testKey", "a");
        StepVerifier
                .create(addMono)
                .expectNext(0L)
                .verifyComplete();

        addMono = this.reactiveSetOperations.add("testKey", "b", "c");
        StepVerifier
                .create(addMono)
                .expectNext(2L)
                .verifyComplete();

        Flux<String> membersFlux = this.reactiveSetOperations.members("testKey");
        StepVerifier
                .create(membersFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(3));
                    assertThat(strings, everyItem(oneOf("a", "b", "c")));
                })
                .verifyComplete();

        Mono<Long> sizeMono = this.reactiveSetOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(3L)
                .verifyComplete();

        Mono<String> popMono = this.reactiveSetOperations.pop("testKey");
        StepVerifier
                .create(popMono)
                .recordWith(ArrayList::new)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(1));
                    assertThat(strings, everyItem(oneOf("a", "b", "c")));
                })
                .verifyComplete();

        sizeMono = this.reactiveSetOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(2L)
                .verifyComplete();

        Flux<String> popFlux = this.reactiveSetOperations.pop("testKey", 2);
        StepVerifier
                .create(popFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(2));
                    assertThat(strings, everyItem(oneOf("a", "b", "c")));
                })
                .verifyComplete();
    }

    @Test
    public void difference_differenceAndStore() {
        this.reactiveSetOperations.add("testKey1", "a", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey2", "aa", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey3", "a", "d").subscribe();

        Flux<String> differenceFlux = this.reactiveSetOperations.difference("testKey1", "testKey2");
        StepVerifier
                .create(differenceFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(1));
                    assertThat(strings, contains("a"));
                })
                .verifyComplete();

        differenceFlux = this.reactiveSetOperations.difference("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(differenceFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(0));
                })
                .verifyComplete();

        Mono<Long> differenceAndStoreMono = this.reactiveSetOperations.differenceAndStore("testKey3", "testKey1", "testKey3-testKey1");
        StepVerifier
                .create(differenceAndStoreMono)
                .expectNext(1L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveSetOperations.members("testKey3-testKey1"))
                .recordWith(ArrayList::new)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(1));
                    assertThat(strings, contains("d"));
                })
                .verifyComplete();
    }

    @Test
    public void intersect_intersectAndStore() {
        this.reactiveSetOperations.add("testKey1", "a", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey2", "aa", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey3", "a", "c", "d").subscribe();

        Flux<String> intersectFlux = this.reactiveSetOperations.intersect("testKey1", "testKey2");
        StepVerifier
                .create(intersectFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(2));
                    assertThat(strings, containsInAnyOrder("b", "c"));
                })
                .verifyComplete();

        intersectFlux = this.reactiveSetOperations.intersect("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(intersectFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(1));
                    assertThat(strings, contains("c"));
                })
                .verifyComplete();

        Mono<Long> intersectAndStoreMono = this.reactiveSetOperations.intersectAndStore("testKey3", "testKey1", "testKey3-testKey1");
        StepVerifier
                .create(intersectAndStoreMono)
                .expectNext(2L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveSetOperations.members("testKey3-testKey1"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(2));
                    assertThat(strings, containsInAnyOrder("a", "c"));
                })
                .verifyComplete();
    }

    @Test
    public void union_unionAndStore() {
        this.reactiveSetOperations.add("testKey1", "a", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey2", "aa", "b", "c").subscribe();
        this.reactiveSetOperations.add("testKey3", "a", "d").subscribe();

        Flux<String> unionFlux = this.reactiveSetOperations.union("testKey1", "testKey2");
        StepVerifier
                .create(unionFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(4));
                    assertThat(strings, containsInAnyOrder("a", "b", "c", "aa"));
                })
                .verifyComplete();

        unionFlux = this.reactiveSetOperations.union("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(unionFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(5));
                    assertThat(strings, containsInAnyOrder("a", "b", "c", "d", "aa"));
                })
                .verifyComplete();

        Mono<Long> unionAndStore = this.reactiveSetOperations.unionAndStore("testKey3", "testKey1", "testKey3-testKey1");
        StepVerifier
                .create(unionAndStore)
                .expectNext(4L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveSetOperations.members("testKey3-testKey1"))
                .recordWith(ArrayList::new)
                .thenConsumeWhile(s -> true)
                .consumeRecordedWith(strings -> {
                    assertThat(strings, hasSize(4));
                    assertThat(strings, containsInAnyOrder("a", "b", "c", "d"));
                })
                .verifyComplete();
    }

    @Test
    public void isMember() {
        this.reactiveSetOperations.add("testKey", "a", "b", "c").subscribe();

        // Spring 2.6, Redis 6.2
        Mono<Boolean> isMemberMono = this.reactiveSetOperations.isMember("testKey", "notExists");
        StepVerifier
                .create(isMemberMono)
                .expectNext(false)
                .verifyComplete();

        isMemberMono = this.reactiveSetOperations.isMember("testKey", "a");
        StepVerifier
                .create(isMemberMono)
                .expectNext(true)
                .verifyComplete();
    }
}
```  


### ReactiveZSetOperations
`Redis` 에서 `Key-Set((value1, score1), (value2, score2) ...)` 구조의 데이터를 조작할 수 있는 인터페이스로 
`ReactiveSetOperations` 와 동일한 인터페이스를 제공하면서 추가적으로 `score` 과 `value` 를 기준으로 정렬된 `Set` 연산을 수행할 수 있다. 
`ReactiveSetOperations` 과 동일한 집합연산과 중복된 `value` 를 허용하지 않는 다는 점이 동일하다. 
그리고 추가적으로 `rank`, `reverseRange`, `rageByScore` 등과 같은 정렬 관련 연산을 제공한다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveZSetOperationsTest implements DockerRedisTest {
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveZSetOperations<String, String> reactiveZSetOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveZSetOperations = this.reactiveRedisTemplate.opsForZSet();
        this.reactiveRedisTemplate.scan().subscribe(o -> {
            this.reactiveRedisTemplate.delete(o);
        });
    }

    @Test
    public void add_addAll_popMax_popMin() {
        Mono<Boolean> addMono = this.reactiveZSetOperations.add("testKey", "a", 0d);
        StepVerifier
                .create(addMono)
                .expectNext(true)
                .verifyComplete();

        addMono = this.reactiveZSetOperations.add("testKey", "a", 1d);
        StepVerifier
                .create(addMono)
                .expectNext(false)
                .verifyComplete();

        addMono = this.reactiveZSetOperations.add("testKey", "abcd", 1d);
        StepVerifier
                .create(addMono)
                .expectNext(true)
                .verifyComplete();

        // TypedTuple.of 2.5
        Mono<Long> addAllMono = this.reactiveZSetOperations.addAll("testKey", Arrays.asList(ZSetOperations.TypedTuple.of("b", 2d), ZSetOperations.TypedTuple.of("c", 3d)));
        StepVerifier
                .create(addAllMono)
                .expectNext(2L)
                .verifyComplete();

        // 2.6
        Mono<ZSetOperations.TypedTuple<String>> podMaxMono = this.reactiveZSetOperations.popMax("testKey");
        StepVerifier
                .create(podMaxMono)
                .expectNext(ZSetOperations.TypedTuple.of("c", 3d))
                .verifyComplete();

        // 2.6
        Mono<ZSetOperations.TypedTuple<String>> popMinMono = this.reactiveZSetOperations.popMin("testKey");
        StepVerifier
                .create(popMinMono)
                .expectNext(ZSetOperations.TypedTuple.of("a", 1d))
                .verifyComplete();
    }

    @Test
    public void differenceWithScores_intersectWithScores_unionWithScores() {
        this.reactiveZSetOperations.addAll(
                "testKey1",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("a", 1d),
                        ZSetOperations.TypedTuple.of("b", 2d),
                        ZSetOperations.TypedTuple.of("c", 3d),
                        ZSetOperations.TypedTuple.of("d", 4d),
                        ZSetOperations.TypedTuple.of("e", 5d)
                )
        ).subscribe();
        this.reactiveZSetOperations.addAll(
                "testKey2",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("a", 11d),
                        ZSetOperations.TypedTuple.of("b", 12d),
                        ZSetOperations.TypedTuple.of("c", 13d)
                        )
        ).subscribe();
        this.reactiveZSetOperations.addAll(
                "testKey3",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("a", 21d),
                        ZSetOperations.TypedTuple.of("c", 23d),
                        ZSetOperations.TypedTuple.of("d", 24d)
                )
        ).subscribe();

        // 2.6
        Flux<ZSetOperations.TypedTuple<String>> differenceWithScoresFlux = this.reactiveZSetOperations.differenceWithScores("testKey1", "testKey2");
        StepVerifier
                .create(differenceWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(2));
                    assertThat(typedTuples, contains(
                            ZSetOperations.TypedTuple.of("d", 4d),
                            ZSetOperations.TypedTuple.of("e", 5d)));
                })
                .verifyComplete();

        differenceWithScoresFlux = this.reactiveZSetOperations.differenceWithScores("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(differenceWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(1));
                    assertThat(typedTuples, contains(ZSetOperations.TypedTuple.of("e", 5d)));
                })
                .verifyComplete();

        // 2.6
        Flux<ZSetOperations.TypedTuple<String>> intersectWithScoresFlux = this.reactiveZSetOperations.intersectWithScores("testKey1", "testKey2");
        StepVerifier
                .create(intersectWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(3));
                    assertThat(typedTuples, containsInAnyOrder(
                            ZSetOperations.TypedTuple.of("a", 12d),
                            ZSetOperations.TypedTuple.of("b", 14d),
                            ZSetOperations.TypedTuple.of("c", 16d)));
                })
                .verifyComplete();

        intersectWithScoresFlux = this.reactiveZSetOperations.intersectWithScores("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(intersectWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(2));
                    assertThat(typedTuples, containsInAnyOrder(
                            ZSetOperations.TypedTuple.of("a", 33d),
                            ZSetOperations.TypedTuple.of("c", 39d)));
                })
                .verifyComplete();

        // 2.6
        Flux<ZSetOperations.TypedTuple<String>> unionWithScoresFlux = this.reactiveZSetOperations.unionWithScores("testKey1", "testKey2");
        StepVerifier
                .create(unionWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(5));
                    assertThat(typedTuples, containsInAnyOrder(
                            ZSetOperations.TypedTuple.of("a", 12d),
                            ZSetOperations.TypedTuple.of("b", 14d),
                            ZSetOperations.TypedTuple.of("c", 16d),
                            ZSetOperations.TypedTuple.of("d", 4d),
                            ZSetOperations.TypedTuple.of("e", 5d)
                    ));
                })
                .verifyComplete();

        unionWithScoresFlux = this.reactiveZSetOperations.unionWithScores("testKey1", Arrays.asList("testKey2", "testKey3"));
        StepVerifier
                .create(unionWithScoresFlux)
                .recordWith(ArrayList::new)
                .thenConsumeWhile(stringTypedTuple -> true)
                .consumeRecordedWith(typedTuples -> {
                    assertThat(typedTuples, hasSize(5));
                    assertThat(typedTuples, containsInAnyOrder(
                            ZSetOperations.TypedTuple.of("a", 33d),
                            ZSetOperations.TypedTuple.of("b", 14d),
                            ZSetOperations.TypedTuple.of("c", 39d),
                            ZSetOperations.TypedTuple.of("d", 28d),
                            ZSetOperations.TypedTuple.of("e", 5d)
                    ));
                })
                .verifyComplete();
    }

    @Test
    public void range_reverseRange_rangeByScore_rangeWithScores_rangeByScoreWithScores() {
        this.reactiveZSetOperations.addAll(
                "testKey",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("a", 11d),
                        ZSetOperations.TypedTuple.of("d", 44d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("e", 55d)
                )
        ).subscribe();

        // all rank
        Flux<String> rangeFlux = this.reactiveZSetOperations.range("testKey", Range.unbounded());
        StepVerifier
                .create(rangeFlux)
                .expectNext("a", "b", "c", "d", "e")
                .verifyComplete();

        // 1 <= rank <= 3
        rangeFlux = this.reactiveZSetOperations.range("testKey", Range.open(1L, 3L));
        StepVerifier
                .create(rangeFlux)
                .expectNext("b", "c", "d")
                .verifyComplete();

        // 1 <= rank <= 3
        rangeFlux = this.reactiveZSetOperations.range("testKey", Range.closed(1L, 3L));
        StepVerifier
                .create(rangeFlux)
                .expectNext("b", "c", "d")
                .verifyComplete();

        // reverse all rank
        Flux<String> reverseRangeFlux = this.reactiveZSetOperations.reverseRange("testKey", Range.unbounded());
        StepVerifier
                .create(reverseRangeFlux)
                .expectNext("e", "d", "c", "b", "a")
                .verifyComplete();

        // reverse 1 <= rank <= 3
        reverseRangeFlux = this.reactiveZSetOperations.reverseRange("testKey", Range.closed(1L, 3L));
        StepVerifier
                .create(reverseRangeFlux)
                .expectNext("d", "c", "b")
                .verifyComplete();

        // reverse 1 <= rank <= 3
        reverseRangeFlux = this.reactiveZSetOperations.reverseRange("testKey", Range.open(1L, 3L));
        StepVerifier
                .create(reverseRangeFlux)
                .expectNext("d", "c", "b")
                .verifyComplete();

        // 22 < score < 44
        Flux<String> rangeByScoreFlux = this.reactiveZSetOperations.rangeByScore("testKey", Range.open(22d, 44d));
        StepVerifier
                .create(rangeByScoreFlux)
                .expectNext("c")
                .verifyComplete();

        // 22 <= score <= 44
        rangeByScoreFlux = this.reactiveZSetOperations.rangeByScore("testKey", Range.closed(22d, 44d));
        StepVerifier
                .create(rangeByScoreFlux)
                .expectNext("b", "c", "d")
                .verifyComplete();

        // 1 <= rank <= 3
        Flux<ZSetOperations.TypedTuple<String>> rangeWithScoresFlux = this.reactiveZSetOperations.rangeWithScores("testKey", Range.open(1L, 3L));
        StepVerifier
                .create(rangeWithScoresFlux)
                .expectNext(
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("d", 44d)
                )
                .verifyComplete();

        // 1 <= rank <= 3
        rangeWithScoresFlux = this.reactiveZSetOperations.rangeWithScores("testKey", Range.closed(1L, 3L));
        StepVerifier
                .create(rangeWithScoresFlux)
                .expectNext(
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("d", 44d)
                )
                .verifyComplete();

        // all score
        Flux<ZSetOperations.TypedTuple<String>> rangeByScoreWithScoresFlux = this.reactiveZSetOperations.rangeByScoreWithScores("testKey", Range.unbounded());
        StepVerifier
                .create(rangeByScoreWithScoresFlux)
                .expectNext(
                        ZSetOperations.TypedTuple.of("a", 11d),
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("d", 44d),
                        ZSetOperations.TypedTuple.of("e", 55d)
                )
                .verifyComplete();

        // 22 < score < 44
        rangeByScoreWithScoresFlux = this.reactiveZSetOperations.rangeByScoreWithScores("testKey", Range.open(22d, 44d));
        StepVerifier
                .create(rangeByScoreWithScoresFlux)
                .expectNext(ZSetOperations.TypedTuple.of("c", 33d))
                .verifyComplete();
    }

    @Test
    public void rank_reverseRank() {
        this.reactiveZSetOperations.addAll(
                "testKey",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("a", 11d),
                        ZSetOperations.TypedTuple.of("d", 44d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("e", 55d)
                )
        ).subscribe();

        Mono<Long> rankMono = this.reactiveZSetOperations.rank("testKey", "a");
        StepVerifier
                .create(rankMono)
                .expectNext(0L)
                .verifyComplete();

        rankMono = this.reactiveZSetOperations.rank("testKey", "e");
        StepVerifier
                .create(rankMono)
                .expectNext(4L)
                .verifyComplete();

        Mono<Long> reverseRankMono = this.reactiveZSetOperations.reverseRank("testKey", "a");
        StepVerifier
                .create(reverseRankMono)
                .expectNext(4L)
                .verifyComplete();

        reverseRankMono = this.reactiveZSetOperations.reverseRank("testKey", "e");
        StepVerifier
                .create(reverseRankMono)
                .expectNext(0L)
                .verifyComplete();
    }

    @Test
    public void remove_removeRange_removeRankByScore() {
        this.reactiveZSetOperations.addAll(
                "testKey",
                Arrays.asList(
                        ZSetOperations.TypedTuple.of("b", 22d),
                        ZSetOperations.TypedTuple.of("a", 11d),
                        ZSetOperations.TypedTuple.of("d", 44d),
                        ZSetOperations.TypedTuple.of("c", 33d),
                        ZSetOperations.TypedTuple.of("e", 55d),
                        ZSetOperations.TypedTuple.of("f", 66d),
                        ZSetOperations.TypedTuple.of("g", 77d),
                        ZSetOperations.TypedTuple.of("h", 88d),
                        ZSetOperations.TypedTuple.of("i", 99d)
                )
        ).subscribe();

        Mono<Long> removeMono = this.reactiveZSetOperations.remove("testKey", "a", "h");
        StepVerifier
                .create(removeMono)
                .expectNext(2L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveZSetOperations.range("testKey", Range.unbounded()))
                .expectNext("b", "c", "d", "e", "f", "g", "i")
                .verifyComplete();

        Mono<Long> removeRangeMono = this.reactiveZSetOperations.removeRange("testKey", Range.closed(0L, 2L));
        StepVerifier
                .create(removeRangeMono)
                .expectNext(3L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveZSetOperations.range("testKey", Range.unbounded()))
                .expectNext("e", "f", "g", "i")
                .verifyComplete();

        Mono<Long> removeRangeWithScoreMono = this.reactiveZSetOperations.removeRangeByScore("testKey", Range.open(11d, 99d));
        StepVerifier
                .create(removeRangeWithScoreMono)
                .expectNext(3L)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveZSetOperations.range("testKey", Range.unbounded()))
                .expectNext("i")
                .verifyComplete();

    }
}
```  

### ReactiveGeoOperations


### ReactiveHyperLogLogOperations
`Redis` 에서 `Key-HyperLogLog(value1, value2, ..)` 와 같은 데이터 구조를 가지면서, 
`Key` 에 해당하는 집합의 원소 개수를 추정하는 인터페이스를 제공한다. 
말 그대로 `Key` 에 해당하는 유니크한 `value` 의 수를 연산하는데 최적화된 데이터 구조이다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
@EnableRedisRepositories
public class ReactiveHyperLogLogOperationsTest implements DockerRedisTest{
    @Autowired
    private ReactiveRedisTemplate reactiveRedisTemplate;
    private ReactiveHyperLogLogOperations reactiveHyperLogLogOperations;

    @BeforeEach
    public void setUp() {
        this.reactiveHyperLogLogOperations = this.reactiveRedisTemplate.opsForHyperLogLog();
        this.reactiveRedisTemplate.scan().subscribe(o -> this.reactiveRedisTemplate.delete(o));
    }

    @Test
    public void add() {
        Mono<Long> addMono = this.reactiveHyperLogLogOperations.add("testKey", "a");
        StepVerifier
                .create(addMono)
                .expectNext(1L)
                .verifyComplete();

        addMono = this.reactiveHyperLogLogOperations.add("testKey", "b", "c");
        StepVerifier
                .create(addMono)
                .expectNext(1L)
                .verifyComplete();

        addMono = this.reactiveHyperLogLogOperations.add("testKey", "a");
        StepVerifier
                .create(addMono)
                .expectNext(0L)
                .verifyComplete();

        addMono = this.reactiveHyperLogLogOperations.add("testKey", "b", "d");
        StepVerifier
                .create(addMono)
                .expectNext(1L)
                .verifyComplete();
    }

    @Test
    public void size() {
        Mono<Long> sizeMono = this.reactiveHyperLogLogOperations.size("testKey");
        StepVerifier
                .create(sizeMono)
                .expectNext(0L)
                .verifyComplete();

        this.reactiveHyperLogLogOperations.add("testKey", "a", "a").subscribe();
        StepVerifier
                .create(sizeMono)
                .expectNext(1L)
                .verifyComplete();

        this.reactiveHyperLogLogOperations.add("testKey", "b", "c", "d").subscribe();
        StepVerifier
                .create(sizeMono)
                .expectNext(4L)
                .verifyComplete();
    }

    @Test
    public void union() {
        this.reactiveHyperLogLogOperations.add("testKey1", "a1", "b1", "c1").subscribe();
        this.reactiveHyperLogLogOperations.add("testKey2", "a2", "b2", "c2").subscribe();
        this.reactiveHyperLogLogOperations.add("testKey3", "a3", "b3", "c3").subscribe();

        Mono<Boolean> unionMono = this.reactiveHyperLogLogOperations.union("newKey", "testKey1");
        StepVerifier
                .create(unionMono)
                .expectNext(true)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveHyperLogLogOperations.size("newKey"))
                .expectNext(3L)
                .verifyComplete();

        unionMono = this.reactiveHyperLogLogOperations.union("newKey", "testKey2", "testKey3");
        StepVerifier
                .create(unionMono)
                .expectNext(true)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveHyperLogLogOperations.size("newKey"))
                .expectNext(9L)
                .verifyComplete();

        unionMono = this.reactiveHyperLogLogOperations.union("newKey", "testKey1");
        StepVerifier
                .create(unionMono)
                .expectNext(true)
                .verifyComplete();
        StepVerifier
                .create(this.reactiveHyperLogLogOperations.size("newKey"))
                .expectNext(9L)
                .verifyComplete();
    }
}
```  




---
## Reference
[ReactiveValueOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveValueOperations.html)  
[ReactiveListOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveListOperations.html)  
[ReactiveHashOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveHashOperations.html)  
[ReactiveSetOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveSetOperations.html)  
[ReactiveZSetOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveZSetOperations.html)  
[ReactiveGeoOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveGeoOperations.html)  
[ReactiveHyperLogLogOperations](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/ReactiveHyperLogLogOperations.html)  