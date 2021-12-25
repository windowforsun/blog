--- 
layout: single
classes: wide
title: "[Java 개념] "
header:
  overlay_image: /img/java-bg.jpg 
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Reactor Extra
toc: true 
use_math: true
---  

## Mono, Flux Caching
`WebFlux` 기반 애플리케이션을 사용하면서 `Caching` 을 적용해본 사람이라면 알겠지만, 
보편적으로 사용하는 `Spring Cache` 라이브러리 만을 사용해서는 `Mono`, `Flux` 값을 기대하는 것과 같이 캐싱할 수 없다는 것을 알 수 있다.  

아래 `methodString` 에서 `String` 타입을 리턴하는 것과 `methodMonoString` 에서 `Mono<String>` 을 리턴하는 예제를 가지고 그 차이를 알아본다.  

```java
public String methodString() {
	return "abc";
}

public Mono<String> methodMonoString() {
	return Mono.just("abc");
}

@Test
public void diff_string_monostring_caching() {
	// given
	Map<String, Object> cache = new HashMap<>();

	// when
	cache.put("methodString", this.methodString());
	cache.put("methodMonoString", this.methodMonoString());

	// then
	assertThat(cache.get("methodString"), instanceOf(String.class));
	assertThat(cache.get("methodString"), is("abc"));
	assertThat(cache.get("methodMonoString"), instanceOf(Mono.class));
	StepVerifier
			.create((Mono<String>)cache.get("methodMonoString"))
			.expectNext("abc")
			.verifyComplete();
}
```  

검증 코드에서 알수 있듯이 `methodString` 은 우리가 기대하는 `abc` 문자열 자체가 캐싱되지만, 
`methodMonoString` 의 경우 `Mono.just("abc")` 객체가 캐싱되는 것을 확인 할 수 있다. 
즉 `Mono`, `Flux` 를 리턴하는 메소드의 결과값을 캐싱하기 위해서는 기존과 동일한 플로우를 사용하게 되면, 
결과 값을 캐싱하는 것이 아니라 `Mono`, `Flux` 객체 자체를 캐싱하기 때문에 실제 결과값 `abc` 를 캐싱하지 못한다.  

위와 같은 문제점을 해결할 수 있는 라이브러리가 바로 [reactor-extra](https://projectreactor.io/docs/extra/release/api/)
에 있는 [CacheMono](https://projectreactor.io/docs/extra/release/api/reactor/cache/CacheMono.html)
[CacheFlux](https://projectreactor.io/docs/extra/release/api/reactor/cache/CacheFlux.html)
이다. 
이를 활용하면 `Spring Cache`, `Caffeine` 등의 여타 캐시 라이브러리들과 함께 기존의 로직 그대로 캐싱 동작을 정상적으로 수행할 수 있다.  

먼자 `CacheMono`, `CacheFlux` 를 사용하는 일반적인 `Flow` 는 아래와 같이 3단계로 구성된다.  

```java
CacheMono                            // or CacheFlux
  .lookup(k -> {..}, key)            // 캐시 검색 : 캐시 저장소에서 key 에 해당하는 캐시가 존재하는지 찾는다. 
  .onCacheMissResume(...)            // 캐시 누락 처리 : 캐시 저장소에 존재하지 않는 경우 key 에 해당하는 값을 리턴한다. 
  .andWriteWith((k, signal) -> {..}) // 캐시 값 쓰기 : onCacheMissResume() 에서 전달된 key에 해당하는 값(signal) 을 캐시 저장소에 저장한다. 
```  

## CacheMono
먼저 `Mono` 타입을 캐싱하는 방법에 대해 알아본다. 
전체적인 흐름은 앞서 설명한 3단계로 구성된다. 

- `lookup()` : 캐시 검색
- `onCacheMissResume()` : 캐시 누락 처리
- `andWriteWith()` : 캐시 값 쓰기

### Manually Handle Lookup and Write
수동으로 캐시를 검색하고 누락된 값을 찾고, 값을 쓰는 방법이다. 

아래 `lookup()` 메소드를 사용해서 먼저 캐시 저장소에 키에 해당하는 값이 있는지 검색해야 한다.  

```java
public static <KEY, VALUE> MonoCacheBuilderCacheMiss<KEY, VALUE> lookup(
			Function<KEY, Mono<Signal<? extends VALUE>>> reader, KEY key)
```  

첫번째 인자로는 캐시 저장소에서 키에 해당하는 값을 찾아 `Mono` 타입으로 리턴하는 `Function` 타입을 전달해야 한다. 
그리고 두번째 인자로 캐시 키를 전달하게 되면, 해당 값은 첫번째 인자로 전달한 `Function` 의 파라미터로 전달된다.  

만약 캐시 저장소에서 키에 해당하는 값이 없는 경우(캐시 누락)에는 `onCacheMissResume()` 을 사용해서 누락된 값을 찾아 `Mono` 타입으로 리턴한다.  

```bash
MonoCacheBuilderCacheWriter<KEY, VALUE> onCacheMissResume(Mono<VALUE> other)
MonoCacheBuilderCacheWriter<KEY, VALUE> onCacheMissResume(Supplier<Mono<VALUE>> otherSupplier)
```  

`onCacheMissResume()` 은 2가지 타입의 인자를 사용할 수 있는데, 
캐시 값을 생성하는데 캐시 키 전달이 필요하지 않는 경우 `Mono` 타입을 직접 전달 할수 있다. 
그리고 캐시값 생성할때 캐시 키 전달이 필요한 경우에는 `Supplier` 타입을 사용한다.  

`onCacheMissResume()` 은 단순히 캐시 값을 찾는 역할만 수행하고, 실제로 캐시 저장소에 누락된 캐시는 저장하진 않는다. 
누락된 캐시 값을 저장하기 위해서는 `andWriteWith()` 를 사용해야 한다.  

```java
Mono<VALUE> andWriteWith(BiFunction<KEY, Signal<? extends VALUE>, Mono<Void>> writer)
```  

`BiFunction` 타입에 전달되는 `key` 와 `Signal`(`value) 을 사용해서 캐시 저장소에 캐시 키에 해당하는 캐시 값을 저장해 주면 된다.  


#### 예제
예제에서는 `Map` 으로 구현한 캐시 저장소를 사용한다.  

```java
final Map<String, String> stringCacheMap = new HashMap<>();
```  










---
## Reference
[reactor-extra](https://projectreactor.io/docs/extra/release/api/)  
[Project Reactor - CacheMono & CacheFlux with Caffeine Examples](https://www.woolha.com/tutorials/project-reactor-cachemono-cacheflux-with-caffeine-examples)  



