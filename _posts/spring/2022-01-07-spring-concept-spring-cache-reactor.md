--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Reactor Cache 구현"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Reactor Extra의 CacheMono, CacheFlux 를 사용해서 Spring Cache 와 같이 Mono, Flux 를 캐싱할 수 있도록 구현해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Cache
    - Cache
    - Reactor
    - Reactor Cache
    - Caffeine
    - Reactor Extra
toc: true
use_math: true
---  

## Spring Reactor Cache
[Spring Cache](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache) 
에서는 아직 공식적으로 `Reactive Stream` 과 관련된 부분은 지원하지 않는다. 
그러므로 기존에 `Spring Cache` 에서 지원해주던 `@Cacheable` 과 같은 간편한 `Annotation` 을 사용한 캐싱은 불가능 한 상황이다. 
`WebFlux` 에서 `Mono`, `Flux` 을 사용해서 값을 처리하다 보면
[Reactor Extra 의 CacheMono, CacheFlux]({{site.baseurl}}{% link _posts/java/2021-12-25-java-concept-reactor-extra-cache-mono-flux.md %}) 
에서 기술한 것처럼 `Mono`, `Flux` 에서 방출되는 값이 캐싱되는 것이 아니라, 객체 그 자체가 캐싱이 돼버려서 기대하는 동작을 수행하지 못할 수 있다.  

가장 간단한 방법은 아래와 같이 `block()` 메소드를 사용해서 값 그 자체를 리턴해서 캐싱이 가능하도록 하는 것이지만, 
이는 `Reactive Stream` 에 대한 모든 장점이 사라지기 때문에 해서는 안된다.  

```java
Mono.<>.block();
Flux.<>.block();
```  

다른 방법으로 고민해볼 수 있는게 [Reactor Extra 의 CacheMono, CacheFlux]({{site.baseurl}}{% link _posts/java/2021-12-25-java-concept-reactor-extra-cache-mono-flux.md %})
에서 구현한 방법을 기반으로 이를 `Spring Cache` 의 동작과 `@Cacheable` `Annotation` 의 사용성에 비슷하게 구현해 보는 방법이 있을 것이다. 
본 포스트에 위 내용과 같이 `CacheMono`, `CacheFlux` 와 `Spring Cache` 의 기본 동작과 특성들을 바탕으로 캐싱 라이브러리를 구현해 본다.  

## 구현하기
`Spring Cache` 에서 제공하는 무수히 많은 기능과 특징들이 있지만, 
그 모든 기능과 특징을 구현하기에는 양이 많기 때문에 1차적으로 몇가지 특징만 뽑아 실제로 동일하게 동작할 수 있도록 구현해 본다.  

- `Annotation` 기반 캐싱
- `Multiple CacheManager`
- `Multiple KeyGenerator`

### build.gradle
구현을 위해 추가한 의존성은 아래와 같다.  

```groovy
dependencies {
	// Spring Framework
	implementation 'org.springframework:spring-context'
	implementation 'org.springframework.boot:spring-boot-autoconfigure'
	implementation 'org.springframework.boot:spring-boot-starter-cache'
	testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
	// AOP
	implementation 'org.aspectj:aspectjweaver'
	// Log
	implementation 'org.slf4j:slf4j-api'
	// Cache
	implementation 'com.github.ben-manes.caffeine:caffeine:2.8.0'
	// Reactor
	implementation 'io.projectreactor:reactor-core:3.4.6'
	implementation 'io.projectreactor.addons:reactor-extra:3.4.6'
	testImplementation 'io.projectreactor:reactor-test'
	// Third Party
	compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
	annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'
	testCompileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
	testAnnotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'
}
```  

### Annotation
`Spring Cache` 에는 `@Cacheable`, `@CachePut`, `@CacheEvict` 등 다양한 동작을 수행하는 `Annotation` 을 제공한다. 
이 중에서 구현 예제는 `@Cacheable` 과 매칭되는 `@ReactorCacheable` 만 구현한다.  

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ReactorCacheable {
    String cacheName() default "";

    String cacheManager() default "";

    String keyGenerator() default "";
}
```  

### ReactorCacheManager
`ReactorCacheManager` 는 `CacheManager` 를 사용해서 `Reactor Cache` 가 적용 될 수 있도록 구현한 `CacheManager` 의 `Wrapper` 구현체다.  

```java
@RequiredArgsConstructor
@Slf4j
public class ReactorCacheManager {
    private final CacheManager cacheManager;

    public CorePublisher executeCacheable(CacheManager cacheManager, Type rawType, String cacheName, Object key, Supplier supplier, Class classType) {
        if(cacheManager == null) {
            cacheManager = this.cacheManager;
        }

        if(rawType.equals(Mono.class)) {
            return this.findCacheMono(cacheManager, cacheName, key, supplier, classType);
        } else {
            return this.findCacheFlux(cacheManager, cacheName, key, supplier);
        }
    }

    public <T> Mono<T> findCacheMono(CacheManager cacheManager, String cacheName, Object key, Supplier<Mono<T>> retriever, Class<T> classType) {
        Cache cache = cacheManager.getCache(cacheName);

        return CacheMono
                .lookup(k -> {
                    if(cache == null) {
                        return Mono.error(new IllegalArgumentException("Cannot find cache named '" + cacheName));
                    }
                    T result = cache.get(k, classType);
                    log.info("[mono cache get] name: {}, key:{}, value: {}", cacheName, k, result);

                    return Mono.justOrEmpty(result).map(Signal::next);
                }, key)
                .onCacheMissResume(Mono.defer(retriever))
                .andWriteWith((k, signal) -> Mono.fromRunnable(() -> {
                    if (!signal.isOnError()) {
                        Object value = signal.get();
                        log.info("[mono cache put] name: {}, key: {}, value: {}", cacheName, k, value);

                        cache.put(k, value);
                    }
                }));
    }

    public <T> Flux<T> findCacheFlux(CacheManager cacheManager, String cacheName, Object key, Supplier<Flux<T>> retriever) {
        Cache cache = cacheManager.getCache(cacheName);

        return CacheFlux
                .lookup(k -> {
                    if(cache == null) {
                        return Mono.error(new IllegalArgumentException("Cannot find cache named '" + cacheName));
                    }
                    List<T> result = cache.get(k, List.class);
                    log.info("[flux cache get] name: {}, key: {}, value: {}", cacheName, k, result);

                    return Mono.justOrEmpty(result)
                            .flatMap(list -> Flux.fromIterable(list).materialize().collectList());
                }, key)
                .onCacheMissResume(Flux.defer(retriever))
                .andWriteWith((k, signals) -> Flux.fromIterable(signals)
                        .dematerialize()
                        .collectList()
                        .doOnNext(list -> {
                            log.info("[flux cache put] name: {}, key:{}, value: {}", cacheName, k, list);
                            cache.put(k, list);
                        })
                        .then());
    }
}
```  

`ReactorCacheManager` 는 생성자를 통해서 기본으로 사용할 `CacheManager` 를 주입 받고, 
추가로 파리미터로도 `CacheManager` 객체를 전달 받는다. 
파라미터로 `CacheManager` 가 전달되지 않는 경우에만 기본 `CacheManager` 를 사용한다. 

이는 사용자가 커스텀한 `CacheManager` 를 설정한 경우에는 해당 객체를 사용하고, 
그렇지 않은 경우에는 기본으로 생성되는 `CacheManager` 를 사용하기 위함이다. 
커스텀한 `CacheManager` 를 생성할 수 있는 방법과 그 우선순위는 아래와 같다. 

1. `@ReactorCacheable` 의 `cacheManager` 필드에 빈으로 등록된 `CacheManager` 이름 설정
   
```java
@ReactorCacheable(cacheManager="testCacheManager")
```  

1. `CachingConfigurer` 에서 `cacheManager` 메소드 오버라이딩

```java
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
	@Override
	public CacheManager cacheManager() {
		SimpleCacheManager cacheManager = new SimpleCacheManager();
		List<CaffeineCache> caffeineCaches = new ArrayList<>();
		caffeineCaches.add(new CaffeineCache("testCache", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));

		cacheManager.setCaches(caffeineCaches);
		cacheManager.afterPropertiesSet();
		return cacheManager;
	}
}
```  

1. 빈으로 `CacheManager` 등록 

```java
@Configuration
public class MyConfig {
	@Bean
	public CacheManager testCacheManager() {
		SimpleCacheManager cacheManager = new SimpleCacheManager();
		List<CaffeineCache> caffeineCaches = new ArrayList<>();
		caffeineCaches.add(new CaffeineCache("testCache2", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(1)).build()));

		cacheManager.setCaches(caffeineCaches);
		cacheManager.afterPropertiesSet();

		return cacheManager;
	}
}
```  

### ReactorCacheAspect
`ReactorCacheAspect` 는 `@ReactorCacheable` 어노테이션이 선언된 메소드가 호출 될때 `Reactor Cache` 관련 로직이 수행될 수 있도록 한다.  

```java
@Slf4j
@Aspect
@RequiredArgsConstructor
public class ReactorCacheAspect {
    private final ReactorCacheManager reactorCacheManager;
    private final KeyGenerator keyGenerator;
    private final BeanFactory beanFactory;

    @Pointcut("@annotation(com.windowforsun.springcache.reactorcache.ReactorCacheable)")
    public void pointcut() {

    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();

        Object target = joinPoint.getTarget();
        ParameterizedType parameterizedType = (ParameterizedType) method.getGenericReturnType();
        Type rawType = parameterizedType.getRawType();

        if (!rawType.equals(Mono.class) && !rawType.equals(Flux.class)) {
            throw new IllegalArgumentException("Return type must be Mono or Flux method : " + method.getName());
        }

        Object[] args = joinPoint.getArgs();
        ThrowingSupplier retriever = () -> joinPoint.proceed(args);
        Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
        Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();
        ReactorCacheable reactorCacheable = method.getAnnotation(ReactorCacheable.class);
        String cacheableName = reactorCacheable.cacheName();
        KeyGenerator keyGenerator = this.keyGenerator;
        CacheManager cacheManager = null;

        if(StringUtils.hasText(reactorCacheable.keyGenerator())) {
            keyGenerator = this.getBean(reactorCacheable.keyGenerator(), KeyGenerator.class);
        }

        if(StringUtils.hasText(reactorCacheable.cacheManager())) {
            cacheManager = this.getBean(reactorCacheable.cacheManager(), CacheManager.class);
        }


        return this.reactorCacheManager
                .executeCacheable(cacheManager, rawType, cacheableName, keyGenerator.generate(target, method, args), retriever, returnClass);
    }

    private <T> T getBean(String beanName, Class<T> expectType) {
        if(!StringUtils.hasText(beanName)) {
            return this.beanFactory.getBean(expectType);
        } else {
            return this.beanFactory.getBean(beanName, expectType);
        }
    }

    @FunctionalInterface
    public interface ThrowingSupplier<T> extends Supplier<T> {
        @Override
        default T get() {
            try {
                return getThrows();
            } catch (Throwable t) {
                throw new RuntimeException(t);
            }
        }

        T getThrows() throws Throwable;
    }
}
```  

`around` 메소드에서는 `@ReactorCacheable` 어노테이션에 선언된 설정 정보를 가져와 필요한 객체를 설정하고 `ReactorCacheManager` 를 호출 한다. 
`cacheManager` 에 설정된 빈 이름이 있는 경우 이를 `ReactorCacheManager` 로 이를 전달한다. 
그리고 `keyGenerator` 또한 설정된 빈 이름이 있는 경우 이를 사용해서 키를 생성하고 있다. 
`KeyGenerator` 를 설정할 수 있는 방법과 그 우선순위는 아래와 같다.  

1. `@ReactorCacheable` 의 `keyGenerator` 에 빈이름으로 등록된 `KeyGenerator` 이름 설정

```java
@ReactorCacheable(keyGenerator="testKeyGenerator")
```  

1. `CachingConfigurer` 에서 `keyGenerator` 메소드 오버라이딩

```java
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
	@Override
	public KeyGenerator keyGenerator() {
		return new TestKeyGenerator();
	}
}
```  

1. 빈으로 `KeyGenerator` 등록

```java
@Configuration
public class MyConfig {
	@Bean
	public KeyGenerator testKeyGenerator() {
		return new TestKeyGenerator2();
	}
}
```  

### ReactorCacheConfig
`ReactorCacheConfig` 는 `Reactor Cache` 동작에 필요한 빈을 설정한다. 

```java
@Configuration
@RequiredArgsConstructor
@EnableAspectJAutoProxy
@Slf4j
public class ReactorCacheConfig {
    @Bean
    public ReactorCacheAspect reactorCacheAspect(ReactorCacheManager reactorCacheManager, Optional<CachingConfigurer> cachingConfigurer, KeyGenerator keyGenerator, BeanFactory beanFactory) {

        if(cachingConfigurer.isPresent() ){
            if(cachingConfigurer.get().keyGenerator() != null){
                keyGenerator = cachingConfigurer.get().keyGenerator();
            }
        }

        return new ReactorCacheAspect(reactorCacheManager, keyGenerator, beanFactory);
    }

    @Bean
    public ReactorCacheManager reactorCacheManager(Optional<CachingConfigurer> cachingConfigurer, CacheManager cacheManager) {

        if(cachingConfigurer.isPresent()) {
            if(cachingConfigurer.get().cacheManager() != null){
                cacheManager = cachingConfigurer.get().cacheManager();
            }
        }

        return new ReactorCacheManager(cacheManager);
    }

    @Bean
    @ConditionalOnMissingBean(KeyGenerator.class)
    public KeyGenerator keyGenerator() {
        return new DefaultKeyGenerator();
    }
}
```  

`reactorCacheAspect`, `reactorCacheManager` 를 보면 `CachingConfigurer` 를 `Optional` 을 통해 주입 받고 있다. 
`CachingConfigurer` 는 사용자가 구현하는 경우에만 존재하는 빈이기 때문에 `Optional` 로 받아 빈이 설정되지 않았을 때와 설정 됐을 때를 구분해서 `CacheManager`, `KeyGenerator` 를 경우에 맞게 설정해주고 있다.  

`defaultKeyGenerator` 는 기본으로 사용할 `KeyGenerator` 로 그 내용은 아래와 같다. 

```java
public class DefaultKeyGenerator extends SimpleKeyGenerator {
    @Override
    public Object generate(Object target, Method method, Object... params) { 
        // 테스트에서 간단하면서도 구분될 수 있도록 구현 
        return generateKey("(defaultKey)", method.getName(), buildParameterized(params));
//        return generateKey(target.getClass().getName(), method, buildParameterized(params));
    }

    private String buildParameterized(Object... objects) {
        return Arrays.stream(objects)
                .map(object -> object == null ? "" : object.toString())
                .collect(Collectors.joining(":"));
    }
}
```  

### 테스트
먼저 테스트로 사용할 `Mono`, `Flux` 를 리턴 타입으로 사용하는 `Repository` 클래스 내용은 아래와 같다.  

```java
@Repository
public class MonoRepository {
    public static String STR = "";
    public static AtomicInteger COUNT = new AtomicInteger();

    @ReactorCacheable(cacheName = "monoString")
    public Mono<String> getCacheString() {
        return Mono.just(STR);
    }

    public Mono<String> getString() {
        return Mono.just(STR);
    }

    @ReactorCacheable(cacheName = "monoString")
    public Mono<String> getCacheString(String str) {
        return Mono.just(str);
    }

    public Mono<String> getString(String str) {
        return Mono.just(str);
    }

    @ReactorCacheable(cacheName = "monoInt")
    public Mono<Integer> getCacheInt() {
        return Mono.just(COUNT.getAndIncrement());
    }

    public Mono<Integer> getInt() {
        return Mono.just(COUNT.getAndIncrement());
    }

    @ReactorCacheable(cacheName = "monoStringCustom", cacheManager = "testCacheManager")
    public Mono<String> getCacheStringCacheManager() {
        return Mono.just(STR);
    }

    @ReactorCacheable(cacheName = "monoStringCustom", keyGenerator = "testKeyGenerator")
    public Mono<String> getCacheStringKeyGenerator() {
        return Mono.just(STR);
    }

    @ReactorCacheable(cacheName = "monoStringCustom", cacheManager = "testCacheManager", keyGenerator = "testKeyGenerator")
    public Mono<String> getCacheStringCacheManagerKeyGenerator() {
        return Mono.just(STR);
    }
}
```  

```java
@Repository
public class FluxRepository {
    public static AtomicInteger COUNT = new AtomicInteger();
    public static String STR = "";

    @ReactorCacheable(cacheName = "fluxString")
    public Flux<String> getCacheString() {
        return Flux.just(STR, STR, STR);
    }

    public Flux<String> getString() {
        return Flux.just(STR, STR, STR);
    }

    @ReactorCacheable(cacheName = "fluxString")
    public Flux<String> getCacheString(String str) {
        return Flux.just(str, str, str);
    }

    public Flux<String> getString(String str) {
        return Flux.just(str, str, str);
    }

    @ReactorCacheable(cacheName = "fluxInt")
    public Flux<Integer> getCacheInt() {
        return Flux.range(COUNT.getAndIncrement(), 3);
    }

    public Flux<Integer> getInt() {
        return Flux.range(COUNT.getAndIncrement(), 3);
    }

    @ReactorCacheable(cacheName = "fluxStringCustom", cacheManager = "testCacheManager")
    public Flux<String> getCacheStringCacheManager() {
        return Flux.just(STR, STR, STR);
    }

    @ReactorCacheable(cacheName = "fluxStringCustom", keyGenerator = "testKeyGenerator")
    public Flux<String> getCacheStringKeyGenerator() {
        return Flux.just(STR, STR, STR);
    }

    @ReactorCacheable(cacheName = "fluxStringCustom", cacheManager = "testCacheManager", keyGenerator = "testKeyGenerator")
    public Flux<String> getCacheStringCacheManagerKeyGenerator() {
        return Flux.just(STR, STR, STR);
    }
}
```  

캐시 이름|캐시되는 값
---|---
`monoString`, `fluxString`|각 클래스의 `STR` 변수 값으로 방출되는 값 캐싱
`monoInt`, `fluxInt`| 각 클래스의 `COUNT` 변수 값이 메소드가 호출 될때 마다 증가하며 방출되는 값 캐싱
`monoStringCustom`, `fluxStringCustom`|`STR` 변수 값으로 방출 되는 값을 캐싱 하지만 커스텀한 `CacheManager` 또는 `KeyGenerator` 사용

테스트 용으로 사용되는 `TestKeyGenerator` 는 아래와 같다.  

```java
public class TestKeyGenerator implements KeyGenerator {
    @Override
    public Object generate(Object target, Method method, Object... params) {
        return new TestKey(params);
    }

    @Getter
    @Setter
    public static class TestKey extends SimpleKey {
        private Object[] params;

        public TestKey(Object... params) {
            super(params);
            this.params = params;
        }

        @Override
        public boolean equals(Object other) {
            return super.equals(other);
        }

        @Override
        public String toString() {
            return "(testKey)" + this.buildParamString(this.params);
        }

        private String buildParamString(Object... objects) {
            return Arrays.stream(objects)
                    .map(obj -> obj == null ? "" : obj.toString())
                    .collect(Collectors.joining(":"));
        }
    }
}
```  

아래는 `MonRepository` 와 `FluxRepository` 의 테스트로 테스트 코드와 각 테스트의 결과 로그도 함께 나열 한다.  


<details><summary>MonRepository 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class MonoRepositoryTest {
    @Autowired
    private MonoRepository monoRepository;

    @TestConfiguration
    public static class CacheConfig {
        @Bean
        @Primary
        public CacheManager cacheManager() {
            // Default CacheManager
            return new ConcurrentMapCacheManager();
        }

        @Bean
        public CacheManager testCacheManager() {
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료시간 3초로 설정
            caffeineCaches.add(new CaffeineCache("monoStringCustom", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();

            return cacheManager;
        }

        @Bean
        public KeyGenerator testKeyGenerator() {
            return new TestKeyGenerator();
        }
    }

    @Test
    public void getInt() {
        MonoRepository.COUNT = new AtomicInteger();
        StepVerifier
                .create(this.monoRepository.getInt())
                .expectNext(0)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getInt())
                .expectNext(1)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getInt())
                .expectNext(2)
                .verifyComplete();
    }

    @Test
    public void getCacheInt() {
        MonoRepository.COUNT = new AtomicInteger();
        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();
    }
    /**
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [defaultKey,getCacheInt,], value: null
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoInt, key: SimpleKey [defaultKey,getCacheInt,], value: 0
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [defaultKey,getCacheInt,], value: 0
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [defaultKey,getCacheInt,], value: 0
     */

    @Test
    public void getCacheStringParam() {
        StepVerifier
                .create(this.monoRepository.getCacheString("a"))
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("b"))
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("a"))
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("b"))
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [defaultKey,getCacheString,a], value: null
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: SimpleKey [defaultKey,getCacheString,a], value: a
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [defaultKey,getCacheString,b], value: null
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: SimpleKey [defaultKey,getCacheString,b], value: b
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [defaultKey,getCacheString,a], value: a
     * INFO 22624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [defaultKey,getCacheString,b], value: b
     */

    @Test
    public void getCacheStringCacheManager() throws Exception {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: b
     * INFO 22000 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: b
     */

    @Test
    public void getCacheStringKeyGenerator() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStringKeyGenerator())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringKeyGenerator())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 20624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: null
     * INFO 20624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: (testKey), value: a
     * INFO 20624 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: a
     */

    @Test
    public void getCacheStringCacheManagerKeyGenerator() throws Exception {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: null
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: (testKey), value: a
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: a
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: a
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: null
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: (testKey), value: b
     * INFO 40104 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: b
     */
}
```  


</div>
</details>



<details><summary>FluxRepository 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class FluxRepositoryTest {
    @Autowired
    private FluxRepository fluxRepository;

    @TestConfiguration
    public static class CacheConfig {
        @Bean
        @Primary
        public CacheManager cacheManager() {
            // Default CacheManager
            return new ConcurrentMapCacheManager();
        }

        @Bean
        public CacheManager testCacheManager() {
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료시간 3초로 설정
            caffeineCaches.add(new CaffeineCache("fluxStringCustom", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();

            return cacheManager;
        }

        @Bean
        public KeyGenerator testKeyGenerator() {
            return new TestKeyGenerator();
        }
    }

    @Test
    public void getInt() {
        FluxRepository.COUNT = new AtomicInteger();

        StepVerifier
                .create(this.fluxRepository.getInt())
                .expectNext(0, 1, 2)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getInt())
                .expectNext(1, 2, 3)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getInt())
                .expectNext(2, 3, 4)
                .verifyComplete();
    }

    @Test
    public void getCacheInt() {
        FluxRepository.COUNT = new AtomicInteger();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();
    }
    /**
     * INFO 23496 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: null
     * INFO 23496 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxInt, key:SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     * INFO 23496 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     * INFO 23496 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     */

    @Test
    public void getCacheStringParam() {
        StepVerifier
                .create(this.fluxRepository.getCacheString("a"))
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString("b"))
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString("a"))
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString("b"))
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,a], value: null
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:SimpleKey [(defaultKey),getCacheString,a], value: [a, a, a]
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,b], value: null
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:SimpleKey [(defaultKey),getCacheString,b], value: [b, b, b]
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,a], value: [a, a, a]
     * INFO 25328 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,b], value: [b, b, b]
     */

    @Test
    public void getCacheStringCacheManager() throws Exception {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [b, b, b]
     * INFO 26940 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [b, b, b]
     */

    @Test
    public void getCacheStringKeyGenerator() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStringKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 15420 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: null
     * INFO 15420 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:(testKey), value: [a, a, a]
     * INFO 15420 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: [a, a, a]
     */

    @Test
    public void getCacheStringCacheManagerKeyGenerator() throws Exception {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManagerKeyGenerator())
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: null
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:(testKey), value: [a, a, a]
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: [a, a, a]
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: [a, a, a]
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: null
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:(testKey), value: [b, b, b]
     * INFO 18688 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: [b, b, b]
     */
}
```  


</div>
</details>


다음은 `CachingConfigurer` 의 구현체를 사용해서 `CacheManager`, `KeyGenerator` 를 오버라이딩한 경우에 대한 테스트 이다.  


<details><summary>CachingConfigurer CacheManager 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class CachingConfigurerCacheManagerMonoRepositoryTest {
    @Autowired
    private MonoRepository monoRepository;

    @TestConfiguration
    public static class CacheConfig extends CachingConfigurerSupport {
        @Override
        public CacheManager cacheManager() {
            // CachingConfigurer 의 우선순위가 더 높기 때문에 기본으로 사용되는 CacheManager
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료시간 3초
            caffeineCaches.add(new CaffeineCache("monoInt", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));
            caffeineCaches.add(new CaffeineCache("monoString", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();
            return cacheManager;
        }

        @Bean
        public CacheManager testCacheManager() {
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료시간 1초
            caffeineCaches.add(new CaffeineCache("monoStringCustom", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(1)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();

            return cacheManager;
        }
    }

    @Test
    public void getCacheInt() throws Exception {
        MonoRepository.COUNT = new AtomicInteger();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(0)
                .verifyComplete();

        Thread.sleep(3100);


        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(1)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(1)
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheInt())
                .expectNext(1)
                .verifyComplete();
    }
    /**
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: null
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoInt, key: SimpleKey [(defaultKey),getCacheInt,], value: 0
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: 0
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: 0
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: null
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoInt, key: SimpleKey [(defaultKey),getCacheInt,], value: 1
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: 1
     * INFO 35564 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoInt, key:SimpleKey [(defaultKey),getCacheInt,], value: 1
     */

    @Test
    public void getCacheString() throws Exception {
        MonoRepository.STR = "a";
        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: SimpleKey [(defaultKey),getCacheString,], value: b
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: b
     * INFO 38788 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: b
     */

    @Test
    public void getCacheString_not_update() throws Exception {
        MonoRepository.STR = "a";
        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        Thread.sleep(1100);

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 45820 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 45820 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 45820 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 45820 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: a
     * INFO 45820 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:SimpleKey [(defaultKey),getCacheString,], value: a
     */

    @Test
    public void getCacheStringCacheManager() throws Exception {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("a")
                .verifyComplete();

        Thread.sleep(1100);

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringCacheManager())
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: a
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: b
     * INFO 19380 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: b
     */
}
```  


</div>
</details>



<details><summary>CachingConfigurer KeyGenerator 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class CachingConfigurerKeyGeneratorFluxRepositoryTest {
    @Autowired
    private FluxRepository fluxRepository;

    public static class MyKeyGenerator implements KeyGenerator {
        @Override
        public Object generate(Object target, Method method, Object... params) {
            return new StringBuilder().append("(MyKey)").append(Arrays.toString(params)).toString();
        }
    }

    @TestConfiguration
    public static class CacheConfig extends CachingConfigurerSupport {
        @Override
        public KeyGenerator keyGenerator() {
            // CachingConfigurer 의 우선순위가 더 높기 때문에 기본으로 사용되는 KeyGnerator
            return new MyKeyGenerator();
        }

        @Bean
        public KeyGenerator testKeyGenerator() {
            return new TestKeyGenerator();
        }
    }

    @Test
    public void getCacheString() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 29436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[], value: null
     * INFO 29436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:(MyKey)[], value: [a, a, a]
     * INFO 29436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[], value: [a, a, a]
     * INFO 29436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[], value: [a, a, a]
     * INFO 29436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[], value: [a, a, a]
     */

    @Test
    public void getCacheString_param() {
        StepVerifier
                .create(this.fluxRepository.getCacheString("a"))
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString("b"))
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString("c"))
                .expectNext("c", "c", "c")
                .verifyComplete();
    }
    /**
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[a], value: null
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:(MyKey)[a], value: [a, a, a]
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[b], value: null
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:(MyKey)[b], value: [b, b, b]
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: (MyKey)[c], value: null
     * INFO 40464 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:(MyKey)[c], value: [c, c, c]
     */

    @Test
    public void getCacheStringKeyGenerator() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStringKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringKeyGenerator())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 45872 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: null
     * INFO 45872 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:(testKey), value: [a, a, a]
     * INFO 45872 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: (testKey), value: [a, a, a]
     */
}
```  


</div>
</details>

마지막으로 `CacheManager` 와 `KeyGenerator` 를 `Bean` 으로 등록했을 때의 테스트 이다. 




<details><summary>Bean CacheManager 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class BeanCacheManagerFluxRepositoryTest {
    @Autowired
    private FluxRepository fluxRepository;

    @TestConfiguration
    public static class CacheConfig {
        @Bean
        @Primary
        public CacheManager cacheManager() {
            // @Primary 를 통해 기본으로 사용될 CacheManager
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료 시간 3초
            caffeineCaches.add(new CaffeineCache("fluxInt", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));
            caffeineCaches.add(new CaffeineCache("fluxString", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(3)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();
            return cacheManager;
        }

        @Bean
        public CacheManager testCacheManager() {
            SimpleCacheManager cacheManager = new SimpleCacheManager();
            List<CaffeineCache> caffeineCaches = new ArrayList<>();
            // 만료 시간 1초
            caffeineCaches.add(new CaffeineCache("fluxStringCustom", Caffeine.newBuilder().expireAfterWrite(Duration.ofSeconds(1)).build()));

            cacheManager.setCaches(caffeineCaches);
            cacheManager.afterPropertiesSet();

            return cacheManager;
        }
    }

    @Test
    public void getCacheInt() throws Exception {
        FluxRepository.COUNT = new AtomicInteger();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();
        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(0, 1, 2)
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(1, 2, 3)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(1, 2, 3)
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheInt())
                .expectNext(1, 2, 3)
                .verifyComplete();
    }
    /**
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: null
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxInt, key:SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [0, 1, 2]
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: null
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxInt, key:SimpleKey [(defaultKey),getCacheInt,], value: [1, 2, 3]
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [1, 2, 3]
     * INFO 24076 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxInt, key: SimpleKey [(defaultKey),getCacheInt,], value: [1, 2, 3]
     */

    @Test
    public void getCacheString() throws Exception {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        Thread.sleep(3100);

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:SimpleKey [(defaultKey),getCacheString,], value: [b, b, b]
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [b, b, b]
     * INFO 12276 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [b, b, b]
     */

    @Test
    public void getCacheString_not_update() throws Exception {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();

        Thread.sleep(1100);

        StepVerifier
                .create(this.fluxRepository.getCacheString())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 25224 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: null
     * INFO 25224 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxString, key:SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 25224 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 25224 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     * INFO 25224 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxString, key: SimpleKey [(defaultKey),getCacheString,], value: [a, a, a]
     */


    @Test
    public void getCacheStringCacheManager() throws Exception {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("a", "a", "a")
                .verifyComplete();

        Thread.sleep(1100);

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("b", "b", "b")
                .verifyComplete();

        StepVerifier
                .create(this.fluxRepository.getCacheStringCacheManager())
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [a, a, a]
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: null
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache put] name: fluxStringCustom, key:SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [b, b, b]
     * INFO 33916 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [flux cache get] name: fluxStringCustom, key: SimpleKey [(defaultKey),getCacheStringCacheManager,], value: [b, b, b]
     */
}
```  


</div>
</details>



<details><summary>Bean KeyGenerator 테스트</summary>
<div markdown="1">

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class BeanKeyGeneratorMonoRepositoryTest {
    @Autowired
    private MonoRepository monoRepository;

    public static class MyKeyGenerator implements KeyGenerator {
        @Override
        public Object generate(Object target, Method method, Object... params) {
            return new StringBuilder().append("(MyKey)").append(Arrays.toString(params)).toString();
        }
    }

    @TestConfiguration
    public static class CacheConfig {
        @Bean
        @Primary
        public KeyGenerator myKeyGenerator() {
            // @Primary 로 기본으로 사용될 KeyGenerator
            return new MyKeyGenerator();
        }

        @Bean
        public KeyGenerator testKeyGenerator() {
            return new TestKeyGenerator();
        }
    }

    @Test
    public void getCacheString() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 39952 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[], value: null
     * INFO 39952 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: (MyKey)[], value: a
     * INFO 39952 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[], value: a
     * INFO 39952 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[], value: a
     * INFO 39952 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[], value: a
     */

    @Test
    public void getCacheString_param() {
        StepVerifier
                .create(this.monoRepository.getCacheString("a"))
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("b"))
                .expectNext("b")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("a"))
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("c"))
                .expectNext("c")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheString("b"))
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[a], value: null
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: (MyKey)[a], value: a
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[b], value: null
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: (MyKey)[b], value: b
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[a], value: a
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[c], value: null
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoString, key: (MyKey)[c], value: c
     * INFO 35080 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoString, key:(MyKey)[b], value: b
     */

    @Test
    public void getCacheStringKeyGenerator() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStringKeyGenerator())
                .expectNext("a")
                .verifyComplete();

        StepVerifier
                .create(this.monoRepository.getCacheStringKeyGenerator())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 41436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: null
     * INFO 41436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache put] name: monoStringCustom, key: (testKey), value: a
     * INFO 41436 --- [           main] c.w.s.reactorcache.ReactorCacheManager   : [mono cache get] name: monoStringCustom, key:(testKey), value: a
     */
}
```  

</div>
</details>


### ToDo
지금까지 구현한 내용과 테스트를 보면 어느정도 `Reactor Cache` 동작이 `Annotation` 방식으로 수행되는 것을 확인 할 수 있다. 
하지만 이는 단순한 예제 일뿐 개선, 추가해야 하는 많은 부분들이 아직 남아 있는 상태이다. 

- `@CacheaPut`, `@CacheEvict` 과 같은 추가 `Annotation` 제공
- `Spring Cache` 와 비슷한 스펙에서 동일한 동작을 구현을 위해 추가 `Wrapping` 클래스 필요(`CacheOperator`, `CachingConfigurer`, ..) 
- `Annotation` 에서 구현되지 못한 기능(필드) 구현(`key`, `condition`, ..)
- `Local Cache`(Memory) 가 아닌 `Redis` 등과 같은 외부 저장소 지원(`Schedulers` 사용)

`Spring Cache` 에서 아직 공식적으로 `Reactor Cache` 를 지원하지 않는 모든 캐시 저장 동작을 비동기로 수행할 수 있는 환경이 아니기 때문이다. 
아직 많은 외부 저장소 라이브러리들이 비동기에 대한 연산을 지원하지 않고 있는 상태이기 때문에, 이를 커스텀하게 구현해서 사용한다면 더욱 해당 부분을 유의해서 사용해야 한다.  

---
## Reference