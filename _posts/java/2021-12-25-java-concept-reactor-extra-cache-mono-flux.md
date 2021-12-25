--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Extra 의 CacheMono, CacheFlux"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor Mono, Flux 를 캐시할 수 있게 하는 reactor-extra 라이브러리의 CacheMono 와 CacheFlux 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Reactor Extra
  - CacheMono
  - CacheFlux
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

위 내용을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept-reactor-extra-cache-mono-flux-1.png)  


## CacheMono
먼저 `Mono` 타입을 캐싱하는 방법에 대해 알아본다. 
전체적인 흐름은 앞서 설명한 3단계로 구성된다. 

- `lookup()` : 캐시 검색
- `onCacheMissResume()` : 캐시 누락 처리
- `andWriteWith()` : 캐시 값 쓰기

### Manually Handle Lookup and Write(MonoCacheBuilderCacheMiss)
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
`Mono` 객체를 바로 전달하거나 `Supplier` 의 객체를 전달 할 수 있다.  
캐시 값을 바로 생성한다면 `Mono` 를 사용하고, 캐시 키에 따라 캐시 값이 생성된다면 `Supplier` 를 사용할 수 있다.  

`onCacheMissResume()` 은 단순히 캐시 값을 찾는 역할만 수행하고, 실제로 캐시 저장소에 누락된 캐시는 저장하진 않는다. 
누락된 캐시 값을 저장하기 위해서는 `andWriteWith()` 를 사용해야 한다.  

```java
Mono<VALUE> andWriteWith(BiFunction<KEY, Signal<? extends VALUE>, Mono<Void>> writer)
```  

`BiFunction` 타입에 전달되는 `key` 와 `Signal`(`value) 을 사용해서 캐시 저장소에 캐시 키에 해당하는 캐시 값을 저장해 주면 된다.  


#### 예제
예제에서는 `Map` 으로 구현한 `String` 타입의 값을 캐싱하는 저장소를 사용한다.  

```java
final Map<String, String> stringCacheMap = new HashMap<>();
```  

캐시 키가 존재하지 않는 경우에 대한 테스트는 아래와 같다.  

```java
@Test
public void monoCacheBuilderCacheMiss_cacheMono_key_not_exists() {
	// given
	String key = "testKey";

	// when
	Mono<String> cachedMono = CacheMono
			.lookup(k -> {
				log.info("lookup:{}", k);

				return Mono.justOrEmpty(stringCacheMap.get(k)).map(Signal::next);
			}, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				return Mono.just("abc");
			})
			.andWriteWith((k, signal) -> {
				String value = signal.get();
				log.info("andWriteWith key:{}, value:{}", k, value);
				return Mono.fromRunnable(() -> stringCacheMap.put(k, value));
			});

	// then
	StepVerifier
			.create(cachedMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(stringCacheMap.size(), is(1));
	assertThat(stringCacheMap.get(key), is("abc"));
}
```  

```
INFO 63736 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : lookup:testKey
INFO 63736 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : onCacheMissResume
INFO 63736 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : andWriteWith:testKey,abc
```  

로그 출력 결과를 보면 앞서 설명한 것처럼 캐시 키가 존재하지 않는 경우 `lookup -> onCacheMissResume -> andWriteWith` 의 순으로 수행되는 것을 확인할 수 있다.  
검증에서 알 수 있듯이 `cachedMono` 가 방출하는 값이 `abc` 인것을 확인 할 수 있고, 
캐시 저장소인 `stringCacheMap` 에도 `testKey` 의 값으로 `abc` 가 저장된 것을 확인 할 수 있다.  

캐시 키가 이미 존자해는 경우는 아래와 같다.  

```java
@Test
public void monoCacheBuilderCacheMiss_cacheMono_key_exists() {
	// given
	String key = "testKey";
	stringCacheMap.put(key, "abc");

	// when
	Mono<String> cachedMono = CacheMono
			.lookup(k -> {
				log.info("lookup:{}", k);

				return Mono.justOrEmpty(stringCacheMap.get(key)).map(Signal::next);
			}, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				// 정확한 테스트를 위해 비어있는 시퀀스 리턴
				return Mono.empty();
			})
			.andWriteWith((k, signal) -> {
				String value = signal.get();
				log.info("andWriteWith key:{}, value:{}", k, value);
				return Mono.fromRunnable(() -> stringCacheMap.put(k, value));
			});

	// then
	StepVerifier
			.create(cachedMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(stringCacheMap.size(), is(1));
	assertThat(stringCacheMap.get(key), is("abc"));
}
```  

```
INFO 77380 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : lookup:testKey
```  

`stringCacheMap` 저장소에 먼저 `testKey` 에 해당하는 `abc` 값을 저장해 주고, 
동일한 코드를 실행시킨 로그 결과를 보면 `lookup` 만 수행된 것을 확인할 수 있다.  


### Provide Map(MonoCacheBuilderMapMiss)
`CacheMono` 를 사용하는 다른 방법은 캐시 저장소에 해당하는 `Map` 을 인자로 전달하는 것이다. 
`Function` 를 사용해서 캐시 저장소에서 `lookup` 을 구현하는 것이 아니라, 
`Map` 타입의 캐시 저장소만 전달해주면 미리 정의된 방법에 따라 `lookup` 을 수행한다.  

```java
public static <KEY, VALUE> MonoCacheBuilderMapMiss<VALUE> lookup(Map<KEY, ? super Signal<? extends VALUE>> cacheMap, KEY key)
public static <KEY, VALUE> MonoCacheBuilderMapMiss<VALUE> lookup(Map<KEY, ? super Signal<? extends VALUE>> cacheMap, KEY key, Class<VALUE> valueClass)
```  

사용할 수 있는 `lookup` 메소드는 위와 같은데 차이점으로는 캐시 값의 타입을 전달 하느냐 마느냐이다.  

다음으로 `lookup` 을 수행했을 때 전달된 캐시 저장소에 키에 해당하는 값이 없을 때에 누락 값을 처리하는 `onCacheMissReume` 의 메소드는 아래와 같다.  


```java
Mono<VALUE> onCacheMissResume(Supplier<Mono<VALUE>> otherSupplier)
Mono<VALUE> onCacheMissResume(Mono<VALUE> other)
```  

두 메소드의 차이는 캐시로 저장할 값에 값을 생성하는 작업을 직접 `Mono` 객체를 전달하느냐 `Supplier` 객체를 전달하느냐의 차이이다.  
캐시 값을 바로 생성한다면 `Mono` 를 사용하고, 캐시 키에 따라 캐시 값이 생성된다면 `Supplier` 를 사용할 수 있다.

`lookup` 에서 `Map` 캐시 저장소를 전달하는 경우 `onCacheMissResume` 에서 존재하지 않는 캐시 값을 가져온 이후, 
`andWriteWith` 를 사용해서 직접 캐시 저장소에 저장하는 작업을 해주지 않아도 된다.  

#### 예제
`MonoCacheBuilderMapMiss` 을 리턴하는 `lookup` 을 사용할때 캐시 저장소인 `Map` 에서 `value` 타입이 `String` 이라면 `Signal<? extends String>` 임을 기억해야 한다. 
실제로 테스트에서 사용할 캐시 저장소는 아래와 같다.  

```java
final Map<String, Signal<? extends String>> stringSignalCacheMap = new HashMap<>();
```  

캐시 값 타입을 지정하지 않는 `lookup` 을 사용하면서 캐시 키가 존재하지 않는 경우에 대한 테스트는 아래와 같다.  

```java
@Test
public void monoCacheBuilderMapMiss_cacheMono_key_not_exists() {
	// given
	String key = "testKey";

	// when
	Mono<String> cachedMono = CacheMono
			.lookup(stringSignalCacheMap, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				return Mono.just("abc");
			});

	// then
	StepVerifier
			.create(cachedMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(stringSignalCacheMap.size(), is(1));
	assertThat(stringSignalCacheMap.get(key).get(), is("abc"));
}
```  

```
INFO 85936 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : onCacheMissResume
```  

캐시 저장소에 캐시 키에 해당하는 값이 존재하지 않는 경우 직접 구현해줘야 할 부분과 실제로 호출 되는 부분은 `onCacheMissResume` 만 있다. 
검즘문을 보면 `cachedMono` 가 `abc` 문자열을 방출하는 것과 `stringSignalCacheMap` 저장소의 `testKey` 값으로 `abc` 값이 저장된 것을 확인할 수 있다.  

다음은 캐시 값 타입을 지정하지 않는 `lookup` 을 사용하면서 캐시 키의 값이 이미 존재하는 경우에 대한 테스트 이다.  

```java
@Test
public void monoCacheBuilderMapMiss_cacheMono_key_exists() {
	// given
	String key = "testKey";
	stringSignalCacheMap.put(key, Signal.next("abc"));

	// when
	Mono<String> cachedMono = CacheMono
			.lookup(stringSignalCacheMap, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				// 정확한 테스트를 위해 비어있는 시퀀스 리턴
				return Mono.empty();
			});
			
	// then
	StepVerifier
			.create(cachedMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(stringSignalCacheMap.size(), is(1));
	assertThat(stringSignalCacheMap.get(key).get(), is("abc"));
}
```  

캐시 키에 해당하는 값이 존재하기 때문에 아무 로그도 출력되지 않고, 캐시 키가 존재하기 때문에 `onCacheMissResume` 메소드도 호출되지 않는다.  


다음으로는 캐시 키 타입을 지정하는 `lookup` 메소드를 사용할떄의 캐시 저장소는 아래와 같다. 
아래와 같이 사용할 경우 캐시 저장소인 `objectSignalMap` 에는 어느 값이든 저장할 수 있다.  

```java
final Map<String, Signal<? extends Object>> objectSignalMap = new HashMap<>();
```  

캐시 값 타입을 지정하는 `lookup` 을 사용하면서 캐시 키의 값이 존재하지 않는 테스트는 아래와 같다.  

```java
@Test
public void monoCacheBuilderMapMiss_2_cacheMono_key_not_exists() {
	// given
	String key = "testKey";

	// when
	Mono<String> cachedMono = CacheMono
			.lookup(objectSignalMap, key, String.class)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				return Mono.just("abc");
			});

	// then
	StepVerifier
			.create(cachedMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(objectSignalMap.size(), is(1));
	assertThat(objectSignalMap.get(key).get(), is("abc"));
}
```  

```
INFO 75652 --- [           main] c.w.r.cachemonoflux.CacheMonoTest        : onCacheMissResume
```  

`lookup` 을 사용할때 캐시 값에 타입을 마지막 인자로 넘겨주면된다. 
그러면 검증 문과 같이 `cachedMono` 가 `abc` 값을 방출하는 것과, `objectSignalMap` 의 `testKey` 에 `abc` 값이 저장된 것을 확인할 수 있다.  

아래는 캐시 값 타입을 지정하는 `lookup` 을 사용하면서 캐시 키가 존재하는 테스트이다.  

```java
@Test
public void monoCacheBuilderMapMiss_2_cacheMono_key_exists() {
	// given
	String key = "testKey";
	objectSignalMap.put(key, Signal.next("abc"));

	// when
	Mono<String> cacheMono = CacheMono
			.lookup(objectSignalMap, key, String.class)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				// 정확한 테스트를 위해 비어있는 시퀀스 리턴
				return Mono.empty();
			});

	// then
	StepVerifier
			.create(cacheMono)
			.expectNext("abc")
			.verifyComplete();
	assertThat(objectSignalMap.size(), is(1));
	assertThat(objectSignalMap.get(key).get(), is("abc"));
}
```  

캐시 키에 해당하는 값이 존재하기 때문에 아무 로그도 출력되지 않고, 캐시 키가 존재하기 때문에 `onCacheMissResume` 메소드도 호출되지 않는다.  


## CacheFlux
`CacheFlux` 또한 `CacheMono` 와 사용방법은 동일하다. 
`CacheFlux` 도 전체적인 흐름은 아래와 같이 3단계로 구성된다.  

- `lookup()` : 캐시 검색
- `onCacheMissResume()` : 캐시 누락 처리
- `andWriteWith()` : 캐시 값 쓰기


### Manually Handle Lookup and Write(FluxCacheBuilderCacheMiss)
수동으로 캐시를 검색하고 누락된 값을 찾고, 해당 값을 직접 캐시 저장소에 쓰는 방법이다.  

아래 `lookup()` 메소드를 사용해서 캐시 저장소에 키에 해당하는 값이 있는지 검색한다.  

```java
public static <KEY, VALUE> FluxCacheBuilderCacheMiss<KEY, VALUE> lookup(
			Function<KEY, Mono<List<Signal<VALUE>>>> reader, KEY key)
```  

첫번째 인자에는 캐시 저장소에서 키에 해당하는 값을 찾아 `Mono` 타입으로 리턴하는 `Function` 객체를 전달한다. 
캐시 저장소에 키가 존재하는 경우 `Mono<List<Signal<VALUE>>` 를 리턴하고, 키가 존재하지 않는 경우 빈 `Mono` 를 리턴해야 한다.  

다음은 캐시 키가 존재하지 않는 경우 값을 생성하는 `onCacheMissResume()` 메소드이다. 
캐시의 값은 `Flux<VALUE>` 타입으로 리턴해 주면된다.  

```java
FluxCacheBuilderCacheWriter<KEY, VALUE> onCacheMissResume(Supplier<Flux<VALUE>> otherSupplier);
FluxCacheBuilderCacheWriter<KEY, VALUE> onCacheMissResume(Flux<VALUE> other);
```  

`Mono` 와 동일하게 직접 `Flux` 객체를 전달하는 방법과 `Supplier` 객체를 전달하는 2가지 방법을 제공한다.  
캐시 값을 바로 생성한다면 `Flux` 를 사용하고, 캐시 키에 따라 캐시 값이 생성된다면 `Supplier` 를 사용할 수 있다.

마지막으로 `andWriteWith()` 를 사용해서 `onCacheMissResume()` 에서 찾은 누락된 캐시 값을 캐시 저장소에 저장하는 부분을 직접 작성해 주면된다.  

```java
Flux<VALUE> andWriteWith(BiFunction<KEY, List<Signal<VALUE>>, Mono<Void>> writer);
```  

#### 예제
예제에서 사용할 캐시 저장소는 `List<String>` 타입을 값으로 저장한다.  

```java
final Map<String, List<String>> stringListCacheMap = new HashMap<>();
```  

캐시 키가 존재하지 않는 경우에 대한 테스트는 아래와 같다.  

```java
@Test
public void fluxCacheBuilderCacheMiss_cacheFlux_key_not_exists() {
	// given
	String key = "testKey";

	// when
	Flux<String> cachedFlux = CacheFlux
			.lookup(k -> {
				if (stringListCacheMap.get(k) != null) {
					log.info("lookup key exists:{}", k);
					Mono<List<Signal<String>>> res = Flux.fromIterable(stringListCacheMap.get(k))
							.map(Signal::next)
							.collectList();

					return res;
				} else {
					log.info("lookup key not exists:{}", k);
					return Mono.empty();
				}
			}, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				return Flux.fromIterable(Arrays.asList("a", "b", "c"));
			})
			.andWriteWith((k, signals) -> {
				log.info("andWriteWith key:{}, value:{}", k, Arrays.toString(signals.toArray()));

				return Mono.fromRunnable(
						() -> stringListCacheMap.put(
								k,
								signals.stream()
										.filter(stringSignal -> stringSignal.getType() == SignalType.ON_NEXT)
										.map(Signal::get)
										.collect(Collectors.toList())

						)
				);
			});

	// then
	StepVerifier
			.create(cachedFlux)
			.expectNext("a")
			.expectNext("b")
			.expectNext("c")
			.verifyComplete();
	assertThat(stringListCacheMap.size(), is(1));
	assertThat(stringListCacheMap.get(key), contains("a", "b", "c"));
}
```  

```
INFO 77768 --- [           main] c.w.r.cachemonoflux.CacheFluxTest        : lookup key not exists:testKey
INFO 77768 --- [           main] c.w.r.cachemonoflux.CacheFluxTest        : onCacheMissResume
INFO 77768 --- [           main] c.w.r.cachemonoflux.CacheFluxTest        : andWriteWith key:testKey, value:[onNext(a), onNext(b), onNext(c), onComplete()]
```  

로그 출력결과를 보면 캐시 키가 존재하지 않기 때문에 `lookup -> onCacheMissResume -> andWriteWith` 순으로 수행된 것을 확인 할 수 있다.  
그리고 `cachedFlux` 가 `a, b, c` 를 차례로 방출하는 것과 `stringListCacheMap` 의 `testKey` 에 `a, b, c` 이 저장된 것도 확인할 수 있다.  

캐시 키가 이미 존재하는 경우는 아래와 같다.  

```java
@Test
public void fluxCacheBuilderCacheMiss_cacheFlux_key_exists() {
	// given
	String key = "testKey";
	stringListCacheMap.put(key, Arrays.asList("a", "b", "c"));

	// when
	Flux<String> cacheFlux = CacheFlux
			.lookup(k -> {
				if (stringListCacheMap.get(k) != null) {
					log.info("lookup key exists:{}", k);
					Mono<List<Signal<String>>> res = Flux.fromIterable(stringListCacheMap.get(k))
							.map(Signal::next)
							.collectList();

					return res;
				} else {
					log.info("lookup key not exists:{}", k);
					return Mono.empty();
				}
			}, key)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				// 정확한 테스트를 위해 비어있는 시퀀스 리턴
				return Flux.empty();
			})
			.andWriteWith((k, signals) -> {
				log.info("andWriteWith key:{}, value:{}", k, Arrays.toString(signals.toArray()));

				return Mono.fromRunnable(
						() -> stringListCacheMap.put(
								k,
								signals.stream()
										.filter(stringSignal -> stringSignal.getType() == SignalType.ON_NEXT)
										.map(Signal::get)
										.collect(Collectors.toList())

						)
				);
			});

	// then
	StepVerifier
			.create(cacheFlux)
			.expectNext("a", "b", "c")
			.verifyComplete();
	assertThat(stringListCacheMap.size(), is(1));
	assertThat(stringListCacheMap.get(key), contains("a", "b", "c"));
}
```  

```java
INFO 79528 --- [           main] c.w.r.cachemonoflux.CacheFluxTest        : lookup key exists:testKey
```  


> reactor-extra 3.4.2 버전을 사용했을 때 `CacheFlux` 를 사용하면서 `lookup` 에서 정상적으로 캐시 키에 해당하는 값을 리턴하더라도, 
> `onCacheMissResume` 이 호출되는 버그가 있었다. 
> 관련해서 이미 [issue](https://github.com/reactor/reactor-addons/issues/234) 
> 가 올려져 있고 3.4.6 버전으로 올려 해결 했다. 


### Provide Map(FluxCacheBuilderMapMiss)
다음은 `CacheMono` 와 동일하게 `Map` 타입의 캐시 저장소를 `lookup()` 메소드에 함께 전달하는 방법이다.  

```java
public static <KEY, VALUE> FluxCacheBuilderMapMiss<VALUE> lookup(
			Map<KEY, ? super List> cacheMap, KEY key, Class<VALUE> valueClass)
```  

`CacheMono` 와 차이점이 있다면, `Map` 에서 값 타입에 일반 타입을 지정할 수 없고, `List` 객체 만을 사용해야 한다. 
그리고 `List` 에는 `Signal` 타입 값만 저장할 수 있다.  

다음으로 `lookup` 을 수행했을 때 전달된 캐시 저장소에 키에 해당하는 값이 없을 때에 누락 값을 처리하는 `onCacheMissReume` 의 메소드는 아래와 같다.

```java
Flux<VALUE> onCacheMissResume(Supplier<Flux<VALUE>> otherSupplier);
Flux<VALUE> onCacheMissResume(Flux<VALUE> other);
```  

두 메소드의 차이는 캐시로 저장할 값에 값을 생성하는 작업을 직접 `Mono` 객체를 전달하느냐 `Supplier` 객체를 전달하느냐의 차이이다.  
캐시 값을 바로 생성한다면 `Flux` 를 사용하고, 캐시 키에 따라 캐시 값이 생성된다면 `Supplier` 를 사용할 수 있다.  


`lookup` 에서 `Map` 캐시 저장소를 전달하는 경우 `onCacheMissResume` 에서 존재하지 않는 캐시 값을 가져온 이후,
`andWriteWith` 를 사용해서 직접 캐시 저장소에 저장하는 작업을 해주지 않아도 된다는 점은 `CacheMono` 와 동일하다.  

#### 예제
`FluxCacheBuilderMapMiss` 을 리턴하는 `lookup` 을 사용할때 캐시 저장소인 `Map` 에서 `value` 타입이 `List` 임을 기억해야 한다.
실제로 테스트에서 사용할 캐시 저장소는 아래와 같다.  

```java
final Map<String, List> listCacheMap = new HashMap<>();
```  

캐시 키가 존재하지 않는 경우에 대한 테스트는 아래와 같다.

```java
@Test
public void fluxCacheBuilderMapMiss_cacheFlux_key_not_exists() {
	// given
	String key = "testKey";

	// when
	Flux<String> cachedFlux = CacheFlux
			.lookup(listCacheMap, key, String.class)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				return Flux.fromIterable(Arrays.asList("a", "b", "c"));
			});

	// then
	StepVerifier
			.create(cachedFlux)
			.expectNext("a", "b", "c")
			.verifyComplete();
	assertThat(listCacheMap.size(), is(1));
	assertThat((List<Signal<String>>) listCacheMap.get(key), contains(Signal.next("a"), Signal.next("b"), Signal.next("c"), Signal.complete()));
}
```  

```
INFO 22720 --- [           main] c.w.r.cachemonoflux.CacheFluxTest        : onCacheMissResume
```  


캐시 키의 값이 이미 존재하는 경우에 대한 테스트이다.  

```java
@Test
public void fluxCacheBuilderMapMiss_cacheFlux_key_exists() {
	BlockHound.install();
	// given
	String key = "testKey";
	listCacheMap.put(key, Arrays.asList(Signal.next("a"), Signal.next("b"), Signal.next("c"), Signal.complete()));

	// when
	Flux<String> cachedFlux = CacheFlux
			.lookup(listCacheMap, key, String.class)
			.onCacheMissResume(() -> {
				log.info("onCacheMissResume");
				// 정확한 테스트를 위해 비어있는 시퀀스 리턴
				return Flux.empty();
			});

	// then
	StepVerifier
			.create(cachedFlux)
			.expectNext("a", "b", "c")
			.verifyComplete();

	assertThat(listCacheMap.size(), is(1));
	assertThat((List<Signal<String>>) listCacheMap.get(key), contains(Signal.next("a"), Signal.next("b"), Signal.next("c"), Signal.complete()));
}
```  

캐시 키가 존재하기 때문에 `onCacheMissResume` 도 수행되지 않아 로그는 출력되지 않는다.  



---
## Reference
[reactor-extra](https://projectreactor.io/docs/extra/release/api/)  
[Project Reactor - CacheMono & CacheFlux with Caffeine Examples](https://www.woolha.com/tutorials/project-reactor-cachemono-cacheflux-with-caffeine-examples)  



