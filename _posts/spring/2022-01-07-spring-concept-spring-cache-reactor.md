--- 
layout: single
classes: wide
title: "[Spring 개념] "
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
`Spring Cache` 에는 `@Cacheable`, `@CachePut`, `@CacheEvict`



---
## Reference