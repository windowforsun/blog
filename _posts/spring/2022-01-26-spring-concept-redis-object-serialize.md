--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Redis Object Serializer"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Redis 를 사용할 때 Object 를 Serializer 로 저장하는 종류와 차이에 대해서 알아본다. '
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - RestTemplate
    - JdkSerializationRedisSerializer
    - StringRedisSerializer
    - GenericJackson2JsonRedisSerializer
    - Jackson2JsonRedisSerializer
toc: true
use_math: true
---  

## Spring Redis Object Serializers
`Redis` 는 `key-value` 구조라는 강력한 성능으로 `DB` 캐시와 같은 중간 저장소 역할로 많이 사용한다. 
`Spring` 혹은 `Spring Boot` 프로젝트에서 이와 같은 동작을 사용해서 `Object` 의 데이터를 캐싱하기 위해서는 적절한 `Serializer` 등록이 필요하다. 
이번 포스트에서는 각 제공되는 `RedisSerializer` 들의 종류와 그 차이에 대해서 알아 본다.  


### 테스트 클래스
`Object` 를 저장하고 실제로 어떤 형식으로 저장되는지 확인을 위해 이후 테스트에서 아래 클래스를 사용한다.  

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ValueClass {
    private String str;
    private int num;
    private List<String> strList;
    private InnerClass innerClass;
    private List<InnerClass> innerClassList;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class InnerClass {
        private String innerStr;
    }
}
```  

### JdkSerializationRedisSerializer
`Spring Redis` 를 사용할 때 별다른 `Serializer` 설정을 하지 않는 경우 기본으로 사용된다. 
`Java` 의 기본 직렬화 방식이므로 `byte` 배열로 저장이 되기 때문에 실제 저장된 값을 사람이 읽어 파악하기에는 어려움이 있다. 
그리고 해당 `Serializer` 를 사용하기 위해서는 직렬화 대상이 되는 모든 `Class` 에 `java.io.Serializable` 를 `implementation` 해줘야 한다.  

```java
@Data
@Builder
public class ValueClass implements Serializable {
    private String str;
    private int num;
    private List<String> strList;
    private InnerClass innerClass;
    private List<InnerClass> innerClassList;

    @Data
    @Builder
    public static class InnerClass implements Serializable {
        private String innerStr;
    }
}
```  

아래는 테스트에서 사용한 코드 이부분이다.  

```java
@Autowired
private RedisTemplate redisTemplate;

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();

this.redisTemplate.opsForValue().set("testKey", valueClass);

ValueClass actual = (ValueClass) this.redisTemplate.opsForValue().get("testKey");
assertThat(actual, notNullValue());
assertThat(actual.getStr(), is("str"));
assertThat(actual.getNum(), is(1));
assertThat(actual.getStrList(), contains("a", "b", "c"));
assertThat(actual.getInnerClass().getInnerStr(), is("innerStr"));
assertThat(actual.getInnerClassList(), hasSize(2));
assertThat(actual.getInnerClassList().get(0).getInnerStr(), is("innerA"));
assertThat(actual.getInnerClassList().get(1).getInnerStr(), is("innerB"));
```  

실제로 저장된 값을 `redis-cli` 를 통해 확인하면 아래와 같다. 

```bash
127.0.0.1:6379> keys *
1) "\xac\xed\x00\x05t\x00\atestKey"
127.0.0.1:6379> get "\xac\xed\x00\x05t\x00\atestKey"
"\xac\xed\x00\x05sr\x00Bcom.windowforsun.springredis.objetserialize.ValueSerializableClass[\x83\xd8\x1d\xbe$\x7f\xb6\x02\x00\x05I\x00\x03numL\x00\x16innerSerializableClasst\x00[Lcom/windowforsun/springredis/objetserialize/ValueSerializableClass$InnerSerializableClas
s;L\x00\x1ainnerSerializableClassListt\x00\x10Ljava/util/List;L\x00\x03strt\x00\x12Ljava/lang/String;L\x00\astrListq\x00~\x00\x02xp\x00\x00\x00\x01sr\x00Ycom.windowforsun.springredis.objetserialize.ValueSerializableClass$InnerSerializableClass\xdd\xdaf\xa6\xd0\x9e\x
80\xae\x02\x00\x01L\x00\binnerStrq\x00~\x00\x03xpt\x00\binnerStrsr\x00\x1ajava.util.Arrays$ArrayList\xd9\xa4<\xbe\xcd\x88\x06\xd2\x02\x00\x01[\x00\x01at\x00\x13[Ljava/lang/Object;xpur\x00\\[Lcom.windowforsun.springredis.objetserialize.ValueSerializableClass$InnerSer
ializableClass;R{H'\x15\xb9\x9al\x02\x00\x00xp\x00\x00\x00\x02sq\x00~\x00\x05t\x00\x06innerAsq\x00~\x00\x05t\x00\x06innerBt\x00\x03strsq\x00~\x00\bur\x00\x13[Ljava.lang.String;\xad\xd2V\xe7\xe9\x1d{G\x02\x00\x00xp\x00\x00\x00\x03t\x00\x01at\x00\x01bt\x00\x01c"
```  

`Serializable` 만 구현해주면 `RedisTemplate` 하나로 여러 `Object` 를 직렬화 해서 `Redis` 에 저장할 수 있다는 장점이 있지만, 
매번 `Serializable` 을 구현해 줘야 한다는 점과 실제 저장된 `key`, `value` 를 파악하기가 어렵다는 단점이 있다.  

### StringRedisSerializer
`SringRedisSerializer` 는 문자열 데이터를 `Redis` 에 직렬화 해서 저장하는 용도로 사용된다. 
`Charset` 은 `UTF-8`, `ASCII` 등 으로 지정이 가능하고 기본으로 `UTF-8` 이 사용된다. 
그리고 주로 아래 처럼 `key` 데이터를 직렬화 하는데 많이 사용 된다.  

```java
@Bean
public RedisTemplate<String, Object> restTemplate(RedisConnectionFactory redisConnectionFactory) {
    RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory);
    // 주로 key serializer 로 지정해서 사용 됨
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setKeySerializer(.. some serializer ..);

    return redisTemplate;
}
```  

`JdkSerializationRedisSerializer` 과 마찬가지로 `StringRedisSerializer` 도 기본으로 `StringRestTemplate` 이라는 빈이 제공된다. 
`StringRedisTemplate` 으로 사용해도 되고 `RedisTemplate<String, String>` 처럼 사용 해도 된다. 
아래는 테스트 코드의 일부분 이다.  

```java

@Autowired
private RedisTemplate<String, String> redisTemplate;

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();
ObjectMapper objectMapper = new ObjectMapper();

this.redisTemplate.opsForValue().set("testKey", objectMapper.writeValueAsString(valueClass));

ValueClass actual = objectMapper.readValue(this.redisTemplate.opsForValue().get("testKey"), ValueClass.class);
assertThat(actual, notNullValue());
assertThat(actual.getStr(), is("str"));
assertThat(actual.getNum(), is(1));
assertThat(actual.getStrList(), contains("a", "b", "c"));
assertThat(actual.getInnerClass().getInnerStr(), is("innerStr"));
assertThat(actual.getInnerClassList(), hasSize(2));
assertThat(actual.getInnerClassList().get(0).getInnerStr(), is("innerA"));
assertThat(actual.getInnerClassList().get(1).getInnerStr(), is("innerB"));
```  

`Object` 를 문자열로 저장해야 되기 때문에 `value` 값은 `ObjectMapper` 를 사용해서 `Json` 으로 직렬화 한 후 저장 했다. 

```bash
127.0.0.1:6379> keys *
1) "testKey"
127.0.0.1:6379> get "testKey"
"{\"str\":\"str\",\"num\":1,\"strList\":[\"a\",\"b\",\"c\"],\"innerClass\":{\"innerStr\":\"innerStr\"},\"innerClassList\":[{\"innerStr\":\"innerA\"},{\"innerStr\":\"innerB\"}]}"
```  

```json
{
    "str": "str",
    "num": 1,
    "strList": [
        "a",
        "b",
        "c"
    ],
    "innerClass": {
        "innerStr": "innerStr"
    },
    "innerClassList": [
        {
            "innerStr": "innerA"
        },
        {
            "innerStr": "innerB"
        }
    ]
}
```

`key`, `value` 모두 사람이 봤을 떄 어느정도 인지가 가능한 형태를 보여주고 있다. 
그리고 `value` 의 경우 일반적인 `Json` 구조이기 때문에 굳이 `Spring` 프레임워크 애플리케이션이 아니더라도 해당 데이터를 읽어 사용할 수 있다. 
하지만 `Spring` 입장에서는 `Object` 를 `Json` 으로 인코딩하고, 다시 `Json` 을 `Object` 로 디코딩해서 사용할 수 있는 중간 처리가 필요하다.  


### GenericJackson2JsonRedisSerializer
`GenericJackson2JsonRedisSerializer` 은 특정 `Object` 타입을 지정하지 않더라도 `Object` 를 `Json` 으로 변환해주고, 
다시 `Json` 을 `Object` 로 생성해주는 `Serializer` 이다. 
`StringRedisSerializer` 에서 `Object` 직렬화를 위해 직접 구현해줘야 하는 동작이 함께 구현된 `Serializer` 라고 할 수 있다. 
그리고 `JdkSerializationRedisSerializer` 처럼 모든 클래스에 `Serializable` 를 구현해 줄 필요도 없다. 
아래는 테스트 코드 일부이다.  

```java
@TestConfiguration
public static class TestConfig {
    @Bean
    public RedisTemplate<String, Object> restTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        return redisTemplate;
    }
}

@Autowired
private RedisTemplate<String, Object> redisTemplate;

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();

this.redisTemplate.opsForValue().set("testKey", valueClass);

ValueClass actual = (ValueClass) this.redisTemplate.opsForValue().get("testKey");
assertThat(actual, notNullValue());
assertThat(actual.getStr(), is("str"));
assertThat(actual.getNum(), is(1));
assertThat(actual.getStrList(), contains("a", "b", "c"));
assertThat(actual.getInnerClass().getInnerStr(), is("innerStr"));
assertThat(actual.getInnerClassList(), hasSize(2));
assertThat(actual.getInnerClassList().get(0).getInnerStr(), is("innerA"));
assertThat(actual.getInnerClassList().get(1).getInnerStr(), is("innerB"));
```  

별도의 변환 없이 `Object` 자체를 `value` 값으로 전달해 주면 된다.  

```bash
127.0.0.1:6379> keys *
1) "testKey"
127.0.0.1:6379> get "testKey"
"{\"@class\":\"com.windowforsun.springredis.objetserialize.ValueClass\",\"str\":\"str\",\"num\":1,\"strList\":[\"java.util.Arrays$ArrayList\",[\"a\",\"b\",\"c\"]],\"innerClass\":{\"@class\":\"com.windowforsun.springredis.objetserialize.ValueClass$InnerClass\",\"innerStr\":\"innerStr\"},\"innerClassList\":[\"java.util.Arrays$ArrayList\",[{\"@class\":\"com.windowforsun.springredis.objetserialize.ValueClass$InnerClass\",\"innerStr\":\"innerA\"},{\"@class\":\"com.windowforsun.springredis.objetserialize.ValueClass$InnerClass\",\"innerStr\":\"innerB\"}]]}"
```  

```json
{
    "@class": "com.windowforsun.springredis.objetserialize.ValueClass",
    "str": "str",
    "num": 1,
    "strList": [
        "java.util.Arrays$ArrayList",
        [
            "a",
            "b",
            "c"
        ]
    ],
    "innerClass": {
        "@class": "com.windowforsun.springredis.objetserialize.ValueClass$InnerClass",
        "innerStr": "innerStr"
    },
    "innerClassList": [
        "java.util.Arrays$ArrayList",
        [
            {
                "@class": "com.windowforsun.springredis.objetserialize.ValueClass$InnerClass",
                "innerStr": "innerA"
            },
            {
                "@class": "com.windowforsun.springredis.objetserialize.ValueClass$InnerClass",
                "innerStr": "innerB"
            }
        ]
    ]
}
```  

`Spring` 에서 단일 애플리케이션을 사용하거나 `Gradle`, `Maven` 멀티 모듈을 통해 여러 애플리케이션이 구성된 경우라면 사용하기에는 아주 간편한 방법이다. 
하지만 `Json` 을 보면 알 수 있듯이, `@class` 라는 필드에 `Class Type`이 들어간다는 점이 있다. 
해당 `Json` 을 다시 `Object` 로 변환하기 위해서는 `@class` 에 저장된 동일한 패키지에 동일한 클래스가 존재해야 정상적으로 사용이 가능하다는 단점이 있다.  


### Jackson2JsonRedisSerializer
`Jackson2JsonRedisSerializer` 는 `GenericJackson2JsonRedisSerializer` 의 단점인 `@class` 필드에 `Class Type` 정보를 저장하지 않는다. 
하지만 `Serializer` 에 `Class Type` 을 명시해 줘야 한다.  

만약 아래처럼 `Object` 타입을 사용하는 경우 `Json` 으로 저장은 가능하지만 값을 로드할때 예외가 발생하게 된다.  

```java
@TestConfiguration
public static class TestConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));

        redisTemplate.afterPropertiesSet();

        return redisTemplate;
    }
}

@Autowired
private RedisTemplate<String, Object> redisTemplate;

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();

this.redisTemplate.opsForValue().set("testKey", valueClass);
        
assertThat(this.redisTemplate.opsForValue().get("testKey"), instanceOf(Object.class));
assertThrows(ClassCastException.class, () -> {
    ValueClass valueClass1 = (ValueClass) this.redisTemplate.opsForValue().get("testKey");
});
```  

정상적으로 저정하고 값을 `Object` 로 로드하기 위해서는 아래와 같이 사용해야 한다.  

```java
@TestConfiguration
public static class TestConfig {
    @Bean
    public RedisTemplate<String, ValueClass> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, ValueClass> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(ValueClass.class));

        redisTemplate.afterPropertiesSet();

        return redisTemplate;
    }
}

@Autowired
private RedisTemplate<String, ValueClass> redisTemplate;

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();

this.redisTemplate.opsForValue().set("testKey", valueClass);

ValueClass actual = this.redisTemplate.opsForValue().get("testKey");
assertThat(actual, notNullValue());
assertThat(actual.getStr(), is("str"));
assertThat(actual.getNum(), is(1));
assertThat(actual.getStrList(), contains("a", "b", "c"));
assertThat(actual.getInnerClass().getInnerStr(), is("innerStr"));
assertThat(actual.getInnerClassList(), hasSize(2));
```  

```
127.0.0.1:6379> keys *
1) "testKey"
127.0.0.1:6379> get "testKey"
"{\"str\":\"str\",\"num\":1,\"strList\":[\"a\",\"b\",\"c\"],\"innerClass\":{\"innerStr\":\"innerStr\"},\"innerClassList\":[{\"innerStr\":\"innerA\"},{\"innerStr\":\"innerB\"}]}"
```  

```json
{
    "str": "str",
    "num": 1,
    "strList": [
        "a",
        "b",
        "c"
    ],
    "innerClass": {
        "innerStr": "innerStr"
    },
    "innerClassList": [
        {
            "innerStr": "innerA"
        },
        {
            "innerStr": "innerB"
        }
    ]
}
```  

위 결과처럼 실제 저장된 `Json` 문자열을 확인해보면, 가장 일반적인 `Json` 이 저장된 것을 확인할 수 있다. 
하지만 `Jackson2JsonRedisSerializer` 은 저장하고자 하는 모든 `Class Type` 과 매핑되는 `RestTemplate` 을 만들어서 사용해야 한다는 단점이 있다.  


### StringRedisSerializer 활용하기
`StringRedisSerializer` 를 사용해서 좀더 커스텀한 기능을 추가하면, 
일반적인 `Json` 형식으로 저장되면서 여러 `Class Type` 을 하나의 `RestTemplate` 을 공유하며 사용할 수 있다. 

```java

@Autowired
private RedisTemplate<String, String> redisTemplate;
private ObjectMapper objectMapper = new ObjectMapper();

public <T> T getData(String key, Class<T> classType) throws Exception {
    String jsonResult = this.redisTemplate.opsForValue().get(key);

    if(StringUtils.hasText(jsonResult)) {
        return this.objectMapper.readValue(jsonResult, classType);
    } else {
        return null;
    }
}

public void setData(String key, Object data) throws Exception {
    this.redisTemplate.opsForValue().set(key, this.objectMapper.writeValueAsString(data));
}

ValueClass valueClass = ValueClass.builder()
    .str("str")
    .num(1)
    .strList(Arrays.asList("a", "b", "c"))
    .innerClass(ValueClass.InnerClass.builder().innerStr("innerStr").build())
    .innerClassList(Arrays.asList(
        ValueClass.InnerClass.builder().innerStr("innerA").build(),
        ValueClass.InnerClass.builder().innerStr("innerB").build()
    ))
    .build();

this.setData("testKey", valueClass);

ValueClass actual = this.getData("testKey", ValueClass.class);
assertThat(actual, notNullValue());
assertThat(actual.getStr(), is("str"));
assertThat(actual.getNum(), is(1));
assertThat(actual.getStrList(), contains("a", "b", "c"));
assertThat(actual.getInnerClass().getInnerStr(), is("innerStr"));
assertThat(actual.getInnerClassList(), hasSize(2));
assertThat(actual.getInnerClassList().get(0).getInnerStr(), is("innerA"));
assertThat(actual.getInnerClassList().get(1).getInnerStr(), is("innerB"));
```  

하지만 이 또한 `Redis` 에서 다른 `Operation`(`HashOperations`, `ListOperations`, ..) 을 사용 한다면 각 메소드 마다 다시 `Wrapping` 해주는 작업이 필요할 것이다.  

어떠한 방법을 사용할지는 구현하고자 하는 애플리케이션과 그 구성에 맞게 선택해야 할 것이다.  

---
## Reference
[Spring boot integrates the JSON serialization folder operation of reids](https://www.fatalerrors.org/a/spring-boot-integrates-the-json-serialization-folder-operation-of-reids.html)  
[Spring Redis Serializers](https://docs.spring.io/spring-data/redis/docs/current/reference/html/#redis:serializer)  