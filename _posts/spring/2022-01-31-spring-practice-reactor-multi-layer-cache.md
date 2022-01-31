--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Reactor Multi Layer Cache 구현"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 Reactor Extra 의 CacheMono, CacheFlux 를 사용해서 다중 캐싱 레이어 동작을 구현해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Reactor
    - Spring Webflux
    - Cache
    - Reactor Cache
    - Reactor Extra
toc: true
use_math: true
---  

## Spring Reactor Multi Layer Cache
본 포스트에서는 `WebFlux`, `Reactor` 와 같은 환경에서
[CacheMono, CacheFlux]({{site.baseurl}}{% link _posts/java/2021-12-25-java-concept-reactor-extra-cache-mono-flux.md %}) 
를 사용해서 다중 레이어 캐시를 구현하는 방법에 대해 알아본다.  

일반적으로 `Cache` 를 사용해서 우리는 애플리케이션의 성능을 끌어올린다. 
캐시 사용에 있어서 가장 빠른 것은 누가 뭐래도 `Local` 캐시일 것이다. 
하지만 `Kubernetes` 환경 처럼 여러 `Scale out` 기반인 경우 `Local` 캐시의 단점이 드러날 수 있다. 
여러 동일한 역할을 수행하는 여러 애플리케이션에서 `Local` 에 캐싱된 데이터가 각기 다를 수 있기 때문이다. 
만약 캐시마다 `TTL` 이 존재하는 경우에는 이 현상은 더욱 잘 드러나게 되는데, 
캐시 관리가 각 `Container` 마다 독립적으로 수행되기 때문이다.  

위 현상을 해결할 수 있는 방법은 `Global` 한 외부 캐시 저장소를 두는 방법으로 해결할 수 있다. 
하지만 이는 `Local` 캐시에 비해 `Network` 비용이 추가된다는 단점이 있기 때문에, 
대부분 `Global` 캐시는 `Local` 캐시에 비해 성능적으론 불리하다.  

이런 경우 `Local`, `Global` 캐시를 모두 사용해서 정답은 아니지만, 
해답 정도로 문제를 해결할 수 있다. 
만약 `DB` 결과를 캐싱한다고 가정한다면, 1차적으로 `Global Cache` 에 `DB` 결과는 캐싱된다. 
그리고 다음으로는 `Local Cache` 에 다시 캐싱되는 순서이다. 
위 와 같이 사용할 경우 `Global Cache` 를 사용해서는 캐시된 데이터의 통일성을 보장하고, 
`Local` 캐시를 통해 추후 반복적인 요청에 대해서 성능을 끌어올릴 수 있다.  

이를 그림으로 그려보면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/practice-reactor-multi-layer-cache-1.png)  

`Spring Cache` 에서 캐시 동작을 `AOP + Annotation` 기반으로 수행하기 때문에, 
테스트 구현 또한 `AOP + Annotation` 기반으로 수행한다.  

위 그림을 보면 `LocalCache` 가 먼저 수행되고, `GlobalCache` 가 그 다음에 수행되는 순서임을 기억해야 한다. 
그리고 이 구현은 `AOP` 를 바탕으로 구현되기 때문에 `AOP` 가 실행되는 순서가 중요한데 이에 대한 자세한 내용은
[Spring AOP Order]({{site.baseurl}}{% link _posts/spring/2022-01-31-spring-practice-reactor-multi-layer-cache.md %})
에서 확인할 수 있다. 
간단하게 설명하면 `@Order` 를 사용해서도 순서를 지정할 수 있고, `Bean` 이 등록되는 순서를 가지고도 순서 지정이 가능하다.  


### build.gradle

```groovy
dependencies {
    implementation 'org.springframework:spring-context'
    implementation 'org.springframework.boot:spring-boot-autoconfigure'
    implementation 'org.aspectj:aspectjweaver'
    implementation 'org.springframework.boot:spring-boot-starter-json'
    implementation 'org.slf4j:slf4j-api'

    // reactor
    implementation 'io.projectreactor:reactor-core:3.4.6'
    // reactor-extra(CacheMono, CacheFlux)
    implementation 'io.projectreactor.addons:reactor-extra:3.4.6'
    
    testImplementation 'io.projectreactor:reactor-test'
    testImplementation 'io.projectreactor.tools:blockhound:1.0.6.RELEASE'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'

    testCompileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    testAnnotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'

}
```  

### Annotation
`Local` 캐시를 동작하도록 하는 `Annotation` 은 아래와 같다. 

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LocalLayerCacheable {
}
```  

`Global` 캐시를 동작하도록 하는 `Annotation` 은 아래와 같다.  

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExternalLayerCacheable {
}
```  

### AOP(Aspect)
`LocalLayerCacheable` 어노테이션을 `Pointcut` 으로 잡아 로컬 캐싱 동작이 수행되는 `Aspect` 클래스는 아래와 같다. 
로컬 캐시 저장소는 `ConcurrentHashMap` 타입의 `localCacheMap` 을 사용한다. 

```java
@Slf4j
@Aspect
@RequiredArgsConstructor
public class LocalLayerCacheAspect {
    public static ConcurrentHashMap<String, Object> localCacheMap = new ConcurrentHashMap<>();
    private final LayerCacheManager layerCacheManager;
    
    @Pointcut("@annotation(com.windowforsun.springcache.reactormulticachelayer.LocalLayerCacheable)")
    public void localLayerCacheablePointcut() {

    }

    @Around("localLayerCacheablePointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();

        Object target = joinPoint.getTarget();
        ParameterizedType parameterizedType = (ParameterizedType) method.getGenericReturnType();
        Type rawType = parameterizedType.getRawType();

        Object[] args = joinPoint.getArgs();
        ThrowingSupplier retriever = () -> joinPoint.proceed(args);
        Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
        Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();
        
        String key = "LocalLayer::" + target.getClass().getSimpleName()  + "::" + method.getName();
        log.info("[before local layer]");
        
        return this.layerCacheManager
                .executeCacheable(localCacheMap, rawType, key, retriever, returnClass);
    }
}
```  

`ExternalLayerCacheable` 어노테이션을 `Pointcut` 으로 잡아 외부 캐싱 동작이 수행되는 `Aspect` 클래스는 아래와 같다. 
외부 캐시 저장소는 테스트를 위해서 `ConcurrentHashMap` 타입의 `externalCacheMap` 을 사용한다. 

```java
@Slf4j
@Aspect
@RequiredArgsConstructor
public class ExternalLayerCacheAspect {
    public static ConcurrentHashMap<String, Object> externalCacheMap = new ConcurrentHashMap<>();
    private final LayerCacheManager layerCacheManager;

    @Pointcut("@annotation(com.windowforsun.springcache.reactormulticachelayer.ExternalLayerCacheable)")
    public void firstLayerCacheablePointcut() {

    }

    @Around("firstLayerCacheablePointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();

        Object target = joinPoint.getTarget();
        ParameterizedType parameterizedType = (ParameterizedType) method.getGenericReturnType();
        Type rawType = parameterizedType.getRawType();

        Object[] args = joinPoint.getArgs();
        ThrowingSupplier retriever = () -> joinPoint.proceed(args);
        Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
        Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();

        String key = "ExternalLayer::" + target.getClass().getSimpleName() + "::" + method.getName();
        log.info("[before external layer]");
        return this.layerCacheManager
                .executeCacheable(externalCacheMap, rawType, key, retriever, returnClass);
    }
}
```  

`AOP` 에서 메소드 동작을 `Supplier` 로 랩핑하는 `ThrowingSupplier` 내용은 아래와 같다.  

```java
@FunctionalInterface
public interface ThrowingSupplier<T> extends Supplier<T> {
    @Override
    default T get() {
        try {
            return getThrows();
        } catch(Throwable t) {
            throw new RuntimeException(t);
        }
    }

    T getThrows() throws Throwable;
}
```  

### CustomCacheManager
`Reactive Stream` 흐름에서 `CacheMono`, `CacheFlux` 와 캐시 저장소를 사용해서 캐싱 동작을 수행하는 `LayerCacheManager` 의 내용은 아래와 같다.  

```java
@Slf4j
@Component
public class LayerCacheManager {

    public CorePublisher executeCacheable(Map<String, Object> cacheMap, Type rawType, String key, Supplier supplier, Class classType) {
        if (rawType.equals(Mono.class)) {
            return this.findCacheMono(cacheMap, key, supplier, classType);
        } else {
            return this.findCacheFlux(cacheMap, key, supplier);
        }
    }

    public <T> Mono<T> findCacheMono(Map<String, Object> cacheMap, String key, Supplier<Mono<T>> retriever, Class<T> classType) {
        return CacheMono
                .lookup(k -> {

                    T result = (T) cacheMap.get(key);
                    log.info("[mono cache get] key: {}, value: {}", k, result);

                    return Mono.justOrEmpty(result).map(Signal::next);
                }, key)
                .onCacheMissResume(Mono.defer(retriever))
                .andWriteWith((k, signal) -> Mono.fromRunnable(() -> {
                    if (!signal.isOnError()) {
                        T value = (T) signal.get();
                        log.info("[mono cache put] key: {}, value: {}", k, value);

                        cacheMap.put(k, value);
                    }
                }));
    }

    public <T> Flux<T> findCacheFlux(Map<String, Object> cacheMap, String key, Supplier<Flux<T>> retriever) {
        return CacheFlux
                .lookup(k -> {
                    List<T> result = (List<T>) cacheMap.get(key);
                    log.info("[flux cache get] key: {}, value: {}", k, result);

                    return Mono.justOrEmpty(result)
                            .flatMap(list -> Flux.fromIterable(list).materialize().collectList());
                }, key)
                .onCacheMissResume(Flux.defer(retriever))
                .andWriteWith((k, signals) -> Flux.fromIterable(signals)
                        .dematerialize()
                        .collectList()
                        .doOnNext(list -> {
                            log.info("[flux cache put] key: {}, value: {}", k, list);
                            cacheMap.put(k, list);
                        })
                        .then());

    }
}
```  

캐시 저장소를 인자로 받아 캐싱 처리를 수행하고, 자세한 동작과 관련 설명은
[CacheMono, CacheFlux]({{site.baseurl}}{% link _posts/java/2021-12-25-java-concept-reactor-extra-cache-mono-flux.md %}), 
[Spring Reactor Cache]({{site.baseurl}}{% link _posts/spring/2022-01-07-spring-concept-spring-cache-reactor.md %})
에서 확인 할 수 있다.  


### Repository
구현된 캐싱 동작이 `Annotation` 을 사용해서 적용되는 `Repository` 클래스이다.  

```java
@Repository
@Slf4j
public class MonoRepository {
    public static String STR = "";

    @LocalLayerCacheable
    @ExternalLayerCacheable
    public Mono<String> getCacheStr() {
        log.info("MonoRepository.getCacheStr : {}", STR);
        return Mono.just(STR);
    }
}
```  

```java
@Repository
@Slf4j
public class FluxRepository {
    public static String STR = "";

    @ExternalLayerCacheable
    @LocalLayerCacheable
    public Flux<String> getCacheStr() {
        log.info("FluxRepository.getCacheStr : {}", STR);
        return Flux.just(STR, STR, STR);
    }
}
```  

### Test
테스트 코드에서 `LocalLayerCacheAspect` 빈을 먼저 선언하고 그 다음 `ExternalLayerCacheAspect` 을 선언했다. 
이렇게 되면 `LocalLayerCacheAspect` 의 우선순위가 더 높아지기 때문에 의도한 바와 동일하게, 
`LocalCache` 가 먼서 수행 된후 `ExternalCache` 가 수행된다.  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class LocalFirstExternalSecondMonoRepositoryTest {
    @TestConfiguration
    public static class TestConfig {
        @Autowired
        private LayerCacheManager layerCacheManager;

        @Bean
        public LocalLayerCacheAspect localLayerCacheAspect() {
            return new LocalLayerCacheAspect(this.layerCacheManager);
        }

        @Bean
        public ExternalLayerCacheAspect externalLayerCacheAspect() {
            return new ExternalLayerCacheAspect(this.layerCacheManager);
        }
    }

    @Autowired
    private MonoRepository monoRepository;

    @BeforeEach
    public void setUp() {
        LocalLayerCacheAspect.localCacheMap.clear();
        ExternalLayerCacheAspect.externalCacheMap.clear();
    }

    @Test
    public void getCacheStr() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     */

    @Test
    public void getCacheStr_cached() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();

        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: a
     */

    @Test
    public void getCacheStr_evict_all() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();

        LocalLayerCacheAspect.localCacheMap.clear();
        ExternalLayerCacheAspect.externalCacheMap.clear();
        MonoRepository.STR = "b";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("b")
                .verifyComplete();
    }
    /**
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : b
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: b
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: b
     */

    @Test
    public void getCacheStr_evict_local() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();

        LocalLayerCacheAspect.localCacheMap.clear();

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     */

    @Test
    public void getCacheStr_evict_external() {
        MonoRepository.STR = "a";

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();

        ExternalLayerCacheAspect.externalCacheMap.clear();

        StepVerifier
                .create(this.monoRepository.getCacheStr())
                .expectNext("a")
                .verifyComplete();
    }
    /**
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: ExternalLayer::MonoRepository::getCacheStr, value: null
     * INFO 21676 --- [           main] c.w.s.r.MonoRepository                   : MonoRepository.getCacheStr : a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: ExternalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache put] key: LocalLayer::MonoRepository::getCacheStr, value: a
     * INFO 21676 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 21676 --- [           main] c.w.s.r.LayerCacheManager                : [mono cache get] key: LocalLayer::MonoRepository::getCacheStr, value: a
     */
}
```  

```java
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class LocalFirstExternalSecondFluxRepositoryTest {
    @TestConfiguration
    public static class TestConfig {
        @Autowired
        private LayerCacheManager layerCacheManager;

        @Bean
        public LocalLayerCacheAspect localLayerCacheAspect() {
            return new LocalLayerCacheAspect(this.layerCacheManager);
        }

        @Bean
        public ExternalLayerCacheAspect externalLayerCacheAspect() {
            return new ExternalLayerCacheAspect(this.layerCacheManager);
        }
    }

    @Autowired
    private FluxRepository fluxRepository;

    @BeforeEach
    public void setUp() {
        LocalLayerCacheAspect.localCacheMap.clear();
        ExternalLayerCacheAspect.externalCacheMap.clear();
    }

    @Test
    public void getCacheStr() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : a
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     */

    @Test
    public void getCacheStr_cached() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();

        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : a
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     */

    @Test
    public void getCacheStr_evict_all() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();

        LocalLayerCacheAspect.localCacheMap.clear();
        ExternalLayerCacheAspect.externalCacheMap.clear();
        FluxRepository.STR = "b";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("b", "b", "b")
                .verifyComplete();
    }
    /**
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : a
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : b
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [b, b, b]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [b, b, b]
     */

    @Test
    public void getCacheStr_evict_local() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();

        LocalLayerCacheAspect.localCacheMap.clear();

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : a
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     */

    @Test
    public void getCacheStr_evict_external() {
        FluxRepository.STR = "a";

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();

        ExternalLayerCacheAspect.externalCacheMap.clear();

        StepVerifier
                .create(this.fluxRepository.getCacheStr())
                .expectNext("a", "a", "a")
                .verifyComplete();
    }
    /**
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.ExternalLayerCacheAspect         : [before external layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: ExternalLayer::FluxRepository::getCacheStr, value: null
     * INFO 10568 --- [           main] c.w.s.r.FluxRepository                   : FluxRepository.getCacheStr : a
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: ExternalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache put] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     * INFO 10568 --- [           main] c.w.s.r.LocalLayerCacheAspect            : [before local layer]
     * INFO 10568 --- [           main] c.w.s.r.LayerCacheManager                : [flux cache get] key: LocalLayer::FluxRepository::getCacheStr, value: [a, a, a]
     */
}
```  

---
## Reference