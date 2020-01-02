--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cache 와 EhCache 적용"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot Cache 에 대해 알아보고, EhCache 를 적용시켜 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - EhCache
  - Spring Cache
---  

## Spring Cache Abstraction
- Spring 3.1 부터 캐시를 보다 쉽게 애플리케이션에서 사용 할 수 있는 캐시 추상화 메서드를 지원한다.
- 트랜잭션을 제공하고 기존 비지니스 로직에 영향을 최소화 하면서 적용 할 수 있다.
- 이는 메서드의 실행 시점에서 파라미터에 대한 캐시가 존재하지 않으면 등록하고, 존재한다면 캐시를 리턴한다.
- 이러한 캐시 로직으로 개발자는 별도의 캐시관련 코드를 작성하지 않아도 된다.
- 아래의 의존성을 통해 Spring Cache Abstraction 의 의존성을 추가할 수 있다.

	```xml
	<dependency>
	  <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter-cache</artifactId>
	</dependency>
	```  
	
- `CacheManager` 를 구성해서 캐시 저장소 및 관련 설정이 필요하다.
- 추가적인 서드파티 모듈(EhCache, Memcached, Redis) 이 없을 경우 로컬 메모리에 캐시를 저장 할 수 있는 `ConcurrentMap` 기반 `ConcurrentMapCacheManager` 의 Bean 이 자동 생성 된다.
- 서드파티 모듈인 EhCache 는 `EhCacheCacheManager`, Redis 는 RedisCacheManager 를 Bean 으로 등록해서 사용 가능 하다.
- 위와 같은 서드파티 모듈을 사용하게 될경우 단순 로컬 메모리 캐싱이 아닌 CacheServer 를 바탕으로 캐싱을 수행 할 수 있다.


## Spring Cache Annotation
- Spring 에서는 Annotation 을 사용해서 캐시 관련 동작을 쉽게 제어 가능하다.

### @EnableCaching
- Annotation 기반으로 캐싱 설정을 사용하도록 한다.
	- 내부적으론 Spring AOP 를 이용
	
속성|설명|기본값
---|---|---
proxyTargetClass|클래스 기반 Proxy 생성 여부<br/>false 인 경우 JDK Dynamic Proxy 사용(Interface 기반)<br/>true 인 경우 CGLIB Proxy 사용(Class 기반)|false
mode|위빙(Weaving) 모드 설정<br/>PROXY : 기존 Spring AOP 방식을 이용한 RTW 방식<br/>ASPECTJ : aspectj 라이브러리를 이용한 CTW, LTW 방식 지원|PROXY
order|AOP order 설정|Integer.MAX_VALUE

### @Cacheable
- 캐싱할 수 있는 메서드를 지정
- 메서드를 기준으로 수행되고, 2가지 기능이 동작한다.
	1. 해당 메서드 인자, 혹은 정의된 캐시키가 존재할 경우 메서드는 수행되지 않고, 캐시 값을 바로 리턴한다.
	1. 캐시키가 존재하지 않아 메서드가 수행 되었을 때, 메서드의 리턴값을 캐시키의 값으로 저장한다.
	
속성|설명|기본값
---|---|---
value, cacheName|캐시 이름|{}
key|같은 캐시 이름에서 구분되는 키 값(KeyGenerator 사용 불가)|""
keyGenerator|특정 로직에 의해 캐시 키를 만들고자 할때<br/>SimpleKeyGenerator, CustomKeyGenerator 를 사용 할 수 있다.|""
cacheManager|사용할 CacheManager 를 지정|""
cacheResolver|캐시 키에 대한 결과값을 돌려주는 Resolver(Interceptor역할)<br/>CacheResolver 를 구현해서 커스텀 처리 가능|""
condition|SpEL 표현식을 통해 특정 조건에 부합하는 경우에만 캐시 사용<br/>and, or 표현식 등으로 복수 조건 사용 가능<br/>연산 조건이 true 일 경우 캐시 동작|""
unless|캐시 동작이 수행되지 않는 조건 설정<br/>연산 조건이 true 이면 캐싱이 되지 않음<br/>ex) (unless = "#key == null") null 인경우 캐시가 동작하지 않음|""
sync|캐시 구현체가 Thread safe 하지 않는 경우, 자체적으로 캐시에 동기화를 거는 속성|false

### @CacheEvict
- 저장된 캐시를 삭제한다.
- 동작시 영향을 주는 하나 이상의 캐시를 지정해야 한다.

속성|설명|기본값
---|---|---
value, cacheName|캐시 이름|{}
key|같은 캐시 이름에서 구분되는 키 값(KeyGenerator 사용 불가)|""
keyGenerator|특정 로직에 의해 캐시 키를 만들고자 할때<br/>SimpleKeyGenerator, CustomKeyGenerator 를 사용 할 수 있다.|""
cacheManager|사용할 CacheManager 를 지정|""
cacheResolver|캐시 키에 대한 결과값을 돌려주는 Resolver(Interceptor역할)<br/>CacheResolver 를 구현해서 커스텀 처리 가능|""
condition|SpEL 표현식을 통해 특정 조건에 부합하는 경우에만 캐시 사용<br/>and, or 표현식 등으로 복수 조건 사용 가능<br/>연산 조건이 true 일 경우 캐시 동작|""
allEntries|캐시 키에 대한 전체 데이터 삭제 여부|false
beforeInvocation|true 일 경우 메서드 실행 이전에 캐시 삭제, false 일 경우 메서드 실행 후 삭제|false

### @CachePut
- 메서드 실행에 영향을 주지 않고 캐시를 갱신해야 하는 경우 사용
- `@CachePut` 은 캐시를 생성하는 용도로만 사용된다.

속성|설명|기본값
---|---|---
value, cacheName|캐시 이름|{}
key|같은 캐시 이름에서 구분되는 키 값(KeyGenerator 사용 불가)|""
keyGenerator|특정 로직에 의해 캐시 키를 만들고자 할때<br/>SimpleKeyGenerator, CustomKeyGenerator 를 사용 할 수 있다.|""
cacheManager|사용할 CacheManager 를 지정|""
cacheResolver|캐시 키에 대한 결과값을 돌려주는 Resolver(Interceptor역할)<br/>CacheResolver 를 구현해서 커스텀 처리 가능|""
condition|SpEL 표현식을 통해 특정 조건에 부합하는 경우에만 캐시 사용<br/>and, or 표현식 등으로 복수 조건 사용 가능<br/>연산 조건이 true 일 경우 캐시 동작|""
unless|캐시 동작이 수행되지 않는 조건 설정<br/>연산 조건이 true 이면 캐싱이 되지 않음<br/>ex) (unless = "#key == null") null 인경우 캐시가 동작하지 않음|""

### @Caching
- `@CacheEvict`, `@CachePut`, `@Cacheable` 을 여러개 지정해서 사용해야 하는 경우 사용 한다.
- 조건식이나 표현식이 다른 경우 사용 한다.
- 여러가지 key에 대한 캐시를 중첩적으로 삭제해야 할 때 사용 한다.

속성|설명|기본값
---|---|---
cacheable|@Cacheable Annotation 을 등록한다.|{}
put|@CachePut Annotation 을 등록한다.|{}
evict|@CacheEvict Annotation 을 등록한다.|{}

### @CacheConfig
- 클래스 레벨에서 캐시에 대한 설정을 할 때 사용한다.
- `CacheManager` 여러개 인 경우 사용 한다.

속성|설명|기본값
---|---|---
cacheNames|캐시 이름|{}
keyGenerator|특정 로직에 의해 캐시 키를 만들고자 할때<br/>SimpleKeyGenerator, CustomKeyGenerator 를 사용 할 수 있다.|""
cacheManager|사용할 CacheManager 를 지정|""
cacheResolver|캐시 키에 대한 결과값을 돌려주는 Resolver(Interceptor역할)<br/>CacheResolver 를 구현해서 커스텀 처리 가능|""



## EhCache
- 오픈소스 기반 Local Cache 다.
- 속도가 빠르고 경량 Cache 이다.
- Disk, Memory 에 저장이 가능하다.
	- `EhCache` 에서 Memory 저장은 기본으로 설정된다.
	- Disk 저장의 경우 `diskPersistent` 기능을 설정해야 저장된다.
- 서버 간 분산 캐시를 지원한다(동기/비동기 복제)
- JSR107 JCache 표준을 지원한다. JCache 에서 제공하는 Annotation 을 사용해서 보다 쉽게 캐시 기능을 적용 할 수 있다.
- `EhCache` 사용을 위해서는 별도의 `xml` 설정을 해주어야 한다.
	- [예시](http://www.ehcache.org/ehcache.xml)
- `EhCache` 를 사용하기 위해서는 아래 의존성을 추가해야 한다.

	```xml
    <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache</artifactId>
        <version>2.10.6</version>
    </dependency>	
	```  


## EhCache 예제
### 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springcacheehcache-1.png)


- `resources/ehcache.xml`

	```xml
	<?xml version="1.0" encoding="UTF-8"?>
	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	         xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
	         updateCheck="true"
	         monitoring="autodetect"
	         dynamicConfig="true">
	
	    <!-- Disk 저장 경로 -->
	    <diskStore path="java.io.tmpdir" />
	
	    <!-- default Cache 설정 (반드시 선언해야 하는 Cache) -->
	    <defaultCache
	            eternal="false"
	            timeToIdleSeconds="0"
	            timeToLiveSeconds="1200"
	            overflowToDisk="false"
	            diskPersistent="false"
	            diskExpiryThreadIntervalSeconds="120"
	            memoryStoreEvictionPolicy="LRU">
	    </defaultCache>
	
	    <cache name="OnlyMemory"
	           maxElementsInMemory="10000"
	           eternal="true"
	           overflowToDisk="false"
	           timeToLiveSeconds="30000"
	           timeToIdleSeconds="0"
	           memoryStoreEvictionPolicy="LFU"
	           transactionalMode="off">
	    </cache>
	
	    <cache name="DiskMemory"
	           diskPersistent="true"
	           maxElementsInMemory="10000"
	           eternal="true"
	           overflowToDisk="false"
	           timeToLiveSeconds="30000"
	           timeToIdleSeconds="0"
	           memoryStoreEvictionPolicy="LFU"
	           transactionalMode="off">
	    </cache>
	</ehcache>
	```  

	- `name` : 캐시 이름 지정
	- `maxEntriesLocalHeap` : 메모리에 생성될 Entry Max 값(0=제한없음)
	- `maxEntriesLocalDisk` : DiskStore에 저장될 Entry Max 값(0=제한없음)
	- `eternal` : 영구 Cache 사용 여부(true 인 경우, timeToIdleSeconds, timeToLiveSeconds 설정은 무시된다.)
	- `timeToIdleSeconds` : 해당 시간 동안 캐시가 사용되지 않으면 삭제 된다.(0=삭제되지 않는다.)
	- `timeToLiveSeconds` : 해당 시간이 지나면 캐시는 삭제된다.(0=삭제되지 않는다)
	- `diskExpiryThreadIntervalSeconds` : DiskStore 캐시 정리 작업 실행 간격 (default = 120)
	- `diskSpoolBufferSizeMB` : 스풀버퍼에 대한 DiskStore 크기 설정
	- `clearOnFlush` : flush() 메서드 호출 시점에 MemoryStore 삭제 여부 (default = true)
	- `memoryStoreEvictionPolicy` : `maxEntriesLocalHeap 설정 값에 도달 했을 때 설정된 정책에 따라 캐시가 제거되고 새로 추가된다.
	- `logging` : 로싱 사용 여부를 설정
	- `maxEntriesInCache` : Terracotta 의 분산캐시에만 사용하능하다. 클러스터에 저장 할 수 있는 최대 Entry 수를 설정한다. 캐시가 동작하는 동은 설정 변경 불가(0=제한없음)
	- `overflowToDisk` : 오버플로우 된 항목에 대해 DiskStore 에 저장할지 여부 (default=false)
	- `diskPersistent` : 캐시를 DiskStore 에 저장하여, 서버 로드시 캐시를 영속화 한다(default=false)

- EhCacheConfig

	```java
	@Configuration
	@EnableCaching
	public class EhCacheConfig {
		// EhCache 설정 로드
	    @Bean
	    public EhCacheManagerFactoryBean ehCacheFactoryBean() {
	        EhCacheManagerFactoryBean ehCacheManagerFactoryBean = new EhCacheManagerFactoryBean();
	        ehCacheManagerFactoryBean.setConfigLocation(new ClassPathResource("ehcache.xml"));
	        ehCacheManagerFactoryBean.setShared(true);
	
	        return ehCacheManagerFactoryBean;
	    }
	
	    // EhCache 설정에 따라 생성된 CacheManager
	    @Bean
	    public EhCacheCacheManager ehCacheManager(EhCacheManagerFactoryBean ehCacheFactoryBean) {
	        EhCacheCacheManager ehCacheCacheManager = new EhCacheCacheManager();
	        ehCacheCacheManager.setCacheManager(ehCacheFactoryBean.getObject());
	
	        return ehCacheCacheManager;
	    }
	}
	```  

	- 위와 같이 CacheManager 가 한개만 존재할 경우 간단하게 `application.properties`, `application.yml` 파일에서 설정파일을 자동이 설정 가능하다.
	
		```yml
		spring:
		  cache:
		    ehcache:
			  config: ehcache.xml
		```  
	
### Spring Cache Annotation 으로 Ehcache 사용하기
- `OnlyDiskMemoryCacheRepository`

	```java
	@Repository
	@CacheConfig(cacheManager = "ehCacheManager", cacheNames = "OnlyMemory")
	public class OnlyMemoryCacheRepository {
	    private static Map<String, String> map = new HashMap<>();
	
	    @PostConstruct
	    public void setUp() {
	        this.init();
	    }
	
	    public void init() {
	        map.put("a", "a1");
	        map.put("b", "b1");
	    }
	
    	@Cacheable(key = "#id")
	    public String findById(String id){
	        try {
	            Thread.sleep(1000);
	        } catch (Exception e) {
	
	        }
	
	        return map.get(id);
	    }
	
	    @CacheEvict(allEntries = true)
	    public void clearCache() {
	
	    }
	
	    public void clearData() {
	        this.clearCache();
	        map.clear();
	    }
	
	    @CachePut(key = "'OnlyMemoryKey'")
	    public List<String> setAll() {
	        List<String> values = new LinkedList<>();
	
	        for(Map.Entry<String, String> entry : map.entrySet()) {
	            values.add(entry.getValue());
	        }
	
	        return values;
	    }
	
	    @Cacheable(key = "'OnlyMemoryKey'")
	    public List<String> findAll() {
	        try {
	            Thread.sleep(1000);
	        } catch(Exception e) {
	
	        }
	        return this.setAll();
	    }
	
	    @Caching(
	            evict = {
	                    @CacheEvict(key = "'OnlyMemoryKey'")
	            },
	            put = {
	                    @CachePut(key = "#key?:'OnlyMemoryKey'")
	            }
	    )
	    public String add(String key, String value) {
	        map.put(key, value);
	
	        return value;
	    }
	}
	```  
	
	- `Repository` 클래스에서 데이터를 가져오는 메서드들은 모두 1초의 딜레이 시간을 주었다.
		- Cache 가 적용 되지 않으면 1초 이상의 시간이 소요된다.
		- Cache 가 적용 되면 1초 미만의 시간이 소요된다.
	
- `OnlyMemoryRepositoryTest`

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest
	public class OnlyMemoryCacheRepositoryTest {
	    @Autowired
	    private OnlyMemoryCacheRepository onlyMemoryCacheRepository;
	
	    @Before
	    public void setUp() {
	        this.onlyMemoryCacheRepository.clearData();
	        this.onlyMemoryCacheRepository.init();
	    }
	
	    @Test
	    public void findById_Once() {
	        // given
	        long start = System.currentTimeMillis();
	
	        // when
	        String result = this.onlyMemoryCacheRepository.findById("a");
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, is("a1"));
	        assertThat(actual, greaterThanOrEqualTo(1000l));
	    }
	
	    @Test
	    public void findById_Twice_UseCache() {
	        // given
	        long start = System.currentTimeMillis();
	        this.onlyMemoryCacheRepository.findById("a");
	
	        // when
	        String result = this.onlyMemoryCacheRepository.findById("a");
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, is("a1"));
	        assertThat(actual, allOf(greaterThanOrEqualTo(1000l), lessThan(1500l)));
	    }
	
	    @Test
	    public void setAll_FindAll_UseCache() {
	        // given
	        long start = System.currentTimeMillis();
	        this.onlyMemoryCacheRepository.setAll();
	
	        // when
	        List<String> result = this.onlyMemoryCacheRepository.findAll();
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, contains("a1", "b1"));
	        assertThat(actual, is(lessThan(1000l)));
	    }
	
	    @Test
	    public void add_FindById_UseCache() {
	        // given
	        String key = "c";
	        String value = "c1";
	        long start = System.currentTimeMillis();
	
	        // when
	        this.onlyMemoryCacheRepository.add(key, value);
	        String result = this.onlyMemoryCacheRepository.findById("c");
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, is("c1"));
	        assertThat(actual, lessThan(1000l));
	    }
	
	    @Test
	    public void setAll_add_FindAll_NotUseCache() {
	        // given
	        String key = "c";
	        String value = "c1";
	        long start = System.currentTimeMillis();
	
	        // when
	        this.onlyMemoryCacheRepository.setAll();
	        this.onlyMemoryCacheRepository.add(key, value);
	        List<String> result = this.onlyMemoryCacheRepository.findAll();
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, contains("a1", "b1", "c1"));
	        assertThat(actual, greaterThanOrEqualTo(1000l));
	    }
	
	    @Test
	    public void setAll_clearCache_findAll_NotUseCache() {
	        // given
	        this.onlyMemoryCacheRepository.setAll();
	        long start = System.currentTimeMillis();
	
	        // when
	        this.onlyMemoryCacheRepository.clearCache();
	        List<String> result = this.onlyMemoryCacheRepository.findAll();
	        long actual = System.currentTimeMillis() - start;
	
	        // then
	        assertThat(result, contains("a1", "b1"));
	        assertThat(actual, greaterThanOrEqualTo(1000l));
	    }
	}
	```  
	
### CacheManager 를 통해 직접 캐시 조작하기
- `EhCacheManagerService`

	```java
	@Service
	@RequiredArgsConstructor
	public class EhCacheManagerService {
	    private final EhCacheCacheManager ehCacheManager;
	
	    public void removeAll() {
	        this.ehCacheManager.getCacheManager().clearAll();
	    }
	
	    public void removeAllOnlyMemory() {
	        this.ehCacheManager.getCacheManager().getCache("OnlyMemory").removeAll();
	    }
	
	    public void removeAllDiskMemory() {
	        this.ehCacheManager.getCacheManager().getCache("DiskMemory").removeAll();
	    }
	
	    public Map<Object, Object> getAll() {
	        Map<Object, Object> caches = new HashMap<>();
	        String[] cacheNames = this.ehCacheManager.getCacheManager().getCacheNames();
	
	        for(String cacheName : cacheNames) {
	            List keys = this.ehCacheManager.getCacheManager().getCache(cacheName).getKeys();
	            caches.putAll(this.ehCacheManager.getCacheManager().getCache(cacheName).getAll(keys));
	        }
	
	        return caches;
	    }
	
	    public Map<Object, Object> getAllByCacheName(String cacheName) {
	        Map<Object, Object> caches = new HashMap<>();
	
	        List keys = this.ehCacheManager.getCacheManager().getCache(cacheName).getKeys();
	        caches.putAll(this.ehCacheManager.getCacheManager().getCache(cacheName).getAll(keys));
	
	        return caches;
	    }
	
	    public Map<Object, Object> getAllOnlyMemory() {
	        return this.getAllByCacheName("OnlyMemory");
	    }

	    public Map<Object, Object> getAllDiskMemory() {
	        return this.getAllByCacheName("DiskMemory");
	    }

	    public void addCacheName(String cacheName, String key, String value) {
	        this.ehCacheManager.getCacheManager().getCache(cacheName).put(new Element(key, value));
	    }
	
	    public void addOnlyMemory(String key, String value) {
	        this.addCacheName("OnlyMemory", key, value);
	    }
	
	    public void addDiskMemory(String key, String value) {
	        this.addCacheName("DiskMemory", key, value);
	    }
	}
	```  

- `EhCacheManagerServiceTest`

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest
	public class EhCacheManagerServiceTest {
	    @Autowired
	    private EhCacheManagerService ehCacheManagerService;
	    @Autowired
	    private OnlyMemoryCacheRepository onlyMemoryCacheRepository;
	
	    @Before
	    public void setUp() {
	        this.ehCacheManagerService.removeAll();
	    }
	
	    @Test
	    public void getAllOnlyMemory() {
	        this.onlyMemoryCacheRepository.findAll();
	
	        Map<Object, Object> actual = this.ehCacheManagerService.getAllOnlyMemory();
	
	        assertThat(actual.size(), is(1));
	        assertThat(actual, hasKey("OnlyMemoryKey"));
	    }
	
	    @Test
	    public void getAllOnlyMemory_2() {
	        this.onlyMemoryCacheRepository.findAll();
	        this.onlyMemoryCacheRepository.findById("a");
	        this.onlyMemoryCacheRepository.findById("b");
	
	        Map<Object, Object> actual = this.ehCacheManagerService.getAllOnlyMemory();
	
	        System.out.println(actual);
	
	        assertThat(actual.size(), is(3));
	        assertThat(actual, hasKey("OnlyMemoryKey"));
	        assertThat(actual, hasKey("a"));
	        assertThat(actual, hasKey("b"));
	    }
	}
	```  
	
	

---
## Reference
