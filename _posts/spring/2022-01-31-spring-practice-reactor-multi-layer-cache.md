--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
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

### 






---
## Reference