--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Bean 생성과 등록 과정"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Bean 생성과 등록 과정에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Bean
    - Spring AutoConfiguration
    - ComponentScan
    - EnableAutoConfiguration
toc: true
use_math: true
---  

## Spring Boot Bean 초기화 과정

`Spring Framework` 는 `IoC Container` 에 생성된 수 많은 `Bean` 들의 
유기적인 상호 과정으로 하나의 애플리케이션이 구동된다. 
그 만큼 `Spring Framework` 에서 가장 핵심 중 하나는 `Bean` 이고, 
우리는 이 `Bean` 을 사용해서 애플리케이션을 개발하는 것과 맞찬가지이다.  

그 만큼 우리는 간단하게 정의만 한 `Bean` 이 어떤 과정으로 생성되고 등록되는지 알고 있을 필요가 있다. 
이와 관련된 내용으로 `Bean` 을 생성하는 대표적인 방법과 그 과정에 대해서 간단하게 알아보고자 한다.  

### Bean 생성
`Spring Boot` 에서 `Bean` 을 생성하는 방법은 대표적으로 아래 2가지가 있다.  

- `@Component`, `@Service` 등과 같은 어노테이션 등록하고자 하는 클래스 레벨에 선언

```java
@Component
public class MyComponent {
    // stub ..
}
```  

- `@Bean` 어노테이션을 `@Configuration` 클래스의 내부에 빈생성 메소드 레벨에 선언

```java
@Configuration
public class MyConfig {
    @Bean
    public MyComponent myComponent() {
        return new MyComponent();
    }
}
```  

### Bean 등록
우리는 위 2가지 방법으로 `Bean` 만 생성에 대한 정의만 하고 따로 `IoC Container` 에 등록하는 과정은 거치지 않는다. 
이러한게 가능한 이유는 우리가 생성하고자 하는 `Bean` 들을 자동으로 `IoC Container` 에 등록해주는 2가지 방식과 어노테이션을 
`Spring Boot` 가 기본적으로 제공해 주기 때문이다.  


#### Component Scan
`Component Scan` 은 `@Component`, `@Bean` 과 같은 어노테이션이 정의된 것들을 `Bean` 으로 등록하는 것을 의미한다. 
위와 동작은 실제로 `@ComponentScan` 어노테이션을 통해 수행된다. 아래는 `@ComponentScan` 어노테이션의 내용이다.  

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {

	@AliasFor("basePackages")
	String[] value() default {};

	@AliasFor("value")
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

	Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

	Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;

	ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;

	String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;

	boolean useDefaultFilters() default true;

	Filter[] includeFilters() default {};

	Filter[] excludeFilters() default {};

	boolean lazyInit() default false;
	
	// ..
}
```  

기본적으로 `basePackages` 에 설정된 패키지 및 하위 패키지에 포함된 모든 경로를 탐색해 `Bean` 으로 등록하게 된다.  

하지만 우리는 `Spring Boot Application` 을 구현할때 별도의 `@ComponentScan` 을 선언해주지 않았지만, 
아무 문제없이 우리가 선언한 `Bean` 들을 사용할 수 있었다. 
그 이유는 `@SpringBootApplication` 어노테이션에 이미 `@ComponentScan` 이 선언돼 있기 때문이다.  

```java
// ..
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
	// stub ..
}
```  

`@SpringBootApplication` 에 선언된 `@ComponentScan` 을 보면 별도의 `basePackages` 가 정의 되지 않은 것을 확인 할 수 있다.  
하지만 우리는 항상 `@SpringBootApplication` 어노테이션이 선언된 클래스 패키지와 그 하위 패키지에 대해서 `Bean` 을 등록하고 사용할 수 있었다. 

그 이유는 [@ComponentScan](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/ComponentScan.html)
문서에 아래와 같이 나와있다. 

> Either basePackageClasses() or basePackages() (or its alias value()) may be specified to define specific packages to scan. If specific packages are not defined, scanning will occur from the package of the class that declares this annotation.

즉 별도로 `basePackages` 가 정의되지 않은 경우, 
`@ComponentScan` 은 기본적으로 해당 어노테이션이 선언된 패키지를 포함한 하위 패키지를 대상으로 동작을 수행하기 때문에 
우리는 문제 없이 `Bean` 을 사용할 수 있었던 것이다.  

`Component Scan` 의 `Bean` 초기화 순서는 아래와 같다. 

1. `@Component`, `@Service` 등 어노테이션
   1. 선언된 클래스 알파펫 순서(A ~ Z)
1. `@Configuration` 클래스에 정의된 `@Bean` 어노테이션
   1. `@Configuration` 이 선언된 클래스 알파펫 순서(A ~ Z)
   1. `@Configuration` 클래스에 `@Bean` 이 정의된 순서(먼저 정의 될 수록 먼저 초기화)


#### Auto Configuration
`Auto Configuration` 은 많이 알려진 `Spring Boot` 의 자동설정 기능으로 `Spring Boot` 가 제공하는 클래스나, 
각 종 라이브러리들의 자동 설정을 가능하도록 하는 방식이다. 
`Auto Configuration` 은 `@EnableAutoConfiguration` 어노테이션을 통해 동작이 수행 되는데, 
이 또한 `@SpringBootApplication` 어노테이션에 기본으로 정의돼 있다.  

```java
// ..
@EnableAutoConfiguration
public @interface SpringBootApplication {
    // ..
}
```  

아래는 `@EnableAutoConfiguration` 의 내용이다.  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```  

`@EnableAutoConfiguration` 은 `src/main/resources/META-INF/spring.factories` 
파일 내부에 정의된 클래스의 풀 패키지 경로를 바탕으로 `Bean` 초기화를 수행하게 된다.  

예시로 `spring-boot-autoconfigure` 패키지에 정의된 내용을 확인하면, 
아래와 같이 자동 설정에 대한 클래스 풀 패키지들이 정의 된 것을 확인 할 수 있다.  


```properties
spring-boot-autoconfigure
└── META-INF
  └── spring.factories



# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

// ..

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\

// ..
```  

### Condition 과 순서
`Bean` 등록을 수행하는 2가지 방법에도 실행 순서는 아래와 같이 `Component Scan` 이후 `Auto Configuration` 이 수행되는 것으로 정해져 있다. 

1. `Component Scan`
1. `Auto Configuration`


`Bean` 은 기본적으로 `Sigleton scope` 로 생성되기 때문에 하나의 애플리케이션에 하나의 `Bean` 인스턴스만 존재 가능하다. 
그로 인해 `Component Scan` 으로 생성된 `Bean` 이 의도치 않게 `Auto Configuration` 에 의해 `Override` 되거나(설정 필요), 
초기화 과정에서 예외가 발생하게 될 것이다.  

이러한 문제는 `Spring 4.0` 부터 사용가능한 [Condition Annotations](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.condition-annotations)
어느정도 해결할 수 있다.  

그리고 자동설정간의 순서도 중요할 수 있는데 이는 [@AutoConfigurerAfter, @AutoConfigurerBefore, @AutoConfigurerOrder](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.locating-auto-configuration-candidates)  
를 사용해서 순서나 우선순위를 조정할 수 있다.  


### 테스트
간단한 테스트 코드로 `Bean` 초기화 순서에 대한 확인을 해본다.  

프로젝트 구성은 아래와 같다.  

```bash
.
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           ├── springbootinitbean
    │   │           │   ├── ABeanConfig.java
    │   │           │   ├── AComponentClass.java
    │   │           │   ├── BBeanConfig.java
    │   │           │   ├── BComponentClass.java
    │   │           │   ├── BeanClass.java
    │   │           │   ├── CDBeanConfig.java
    │   │           │   └── DemoApplication.java
    │   │           ├── myautoconfig
    │   │           │   ├── AutoConfigClass.java
    │   │           │   ├── EnableMyAutoConfigBean.java
    │   │           │   └── MyAutoConfig.java
    │   │           └── myautoconfigu2
    │   │               ├── AutoConfigClass2.java
    │   │               ├── EnableMyAutoConfigBean2.java
    │   │               └── MyAutoConfig2.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
    └── test
        └── java
            └── com
                └── windowforsun
                    └── beaninitializeprocess
                        ├── ComponentScanAndAutoConfigTest.java
                        └── ComponentScanTest.java
```  

프로젝트는 3개의 패키지와 `spring.factories` 로 구성돼 있다. 

- `springbootinitbean` : `@SpringBootApplication` 이 선언된 애플리케이션 패키지
- `myautoconfig` : 커스텀 자동설정 패키지 1
- `myautoconfig2` : 커스텀 자동설정 패키지 2
- `spring.factories` : `myautoconfig`, `myautoconfig2` 자동 설정 등록


`springbootinitbean` 패키지에 정의된 `Bean` 만 `@SpringBootApplication` 을 통해 초기화 되고,
`myautoconfig`, `myautoconfig2` 는 자동 설정을 통해 초기화 된다.

#### springbootinitbean 패키지 클래스

- `@Configuration` 클래스에서 `@Bean` 을 통해 생성할 클래스 

```java
@Slf4j
public class BeanClass {
    private String value = "";

    public BeanClass() {
    }

    public BeanClass(String value) {
        this.value = value;
    }

    @PostConstruct
    public void init() {
        log.info("BeanClass.PostConstruct : {}", this.value);
    }
}
```  

```java
@Configuration
public class ABeanConfig {
    @Bean
    public BeanClass aBeanClass() {
        return new BeanClass("A");
    }
}
```  

```java
@Configuration
public class BBeanConfig {
    @Bean
    public BeanClass bBeanClass() {
        return new BeanClass("B");
    }
}
```  

```java
@Configuration
public class CDBeanConfig {
    @Bean
    public BeanClass dBeanClass() {
        return new BeanClass("D");
    }

    @Bean
    public BeanClass cBeanClass() {
        return new BeanClass("C");
    }
}
```  


- `@Component` 어노테이션을 사용해서 생성할 클래스

```java
@Slf4j
@Component
public class AComponentClass {
    @PostConstruct
    public void init() {
        log.info("ACommonClass.PostConstruct");
    }
}
```  

```java
@Slf4j
@Component
public class BComponentClass {
    @PostConstruct
    public void init() {
        log.info("BCommonClass.PostConstruct");
    }
}
```  

- `SpringBootApplication` 클래스

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```  

#### myautoconfig

```java
@Slf4j
public class AutoConfigClass {
    @PostConstruct
    public void init() {
        log.info("AutoConfigClass.PostConstruct");
    }
}
```  

```java
public class EnableMyAutoConfigBean {

}
```  

```java
@Configuration
@ConditionalOnBean(EnableMyAutoConfigBean.class)
public class MyAutoConfig {
    @Bean
    public AutoConfigClass autoConfigClass() {
        return new AutoConfigClass();
    }
}
```  

`MyAutoConfig` 자동 설정은 `EnableMyAutoConfigBean` 이 정의된 경우에만 설정 된다.  

#### myautoconfig2

```java
@Slf4j
public class AutoConfigClass2 {
    @PostConstruct
    public void init() {
        log.info("AutoConfigClass2.PostConstruct");
    }
}
```  

```java
public class EnableMyAutoConfigBean2 {

}
```  

```java
@Configuration
@ConditionalOnBean(EnableMyAutoConfigBean2.class)
public class MyAutoConfig2 {
    @Bean
    public AutoConfigClass2 autoConfigClass2() {
        return new AutoConfigClass2();
    }
}
```  

`MyAutoConfig2` 자동 설정은 `EnableMyAutoConfigBean2` 가 정의된 경우에만 수행된다.  

#### spring.factories
`myautoconfig`, `myautoconfig2` 를 자동 설정을 위해 `src/main/resources/META-INF/spring.factories` 
파일에 아래 내용을 작성해 준다.  

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.windowforsun.myautoconfig.MyAutoConfig,\
com.windowforsun.myautoconfigu2.MyAutoConfig2
```  

#### 테스트
- `myatoconfig`, `myautoconfig2` 자동 설정은 초기화하지 않고, 
  `springbootinitbean` 패키지 `Bean` 만 초기화한 경우 테스트이다. 
  - `@Component` 어노테이션 클래스가 알파벳 순으로 초기화된 후, 
	`@Configuration` 클래스의 알파벳 순서에서 각 클래스에 `@Bean` 이 정의된 순서대로 초기화되는 것을 확인 할 수 있다.   
  
  
```java
@Slf4j
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class ComponentScanTest {
    @Test
    public void runTest() {
        log.info("test done");
    }
}
```  

```
INFO 39356 --- [           main] c.w.springbootinitbean.AComponentClass   : ACommonClass.PostConstruct
INFO 39356 --- [           main] c.w.springbootinitbean.BComponentClass   : BCommonClass.PostConstruct
INFO 39356 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : A
INFO 39356 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : B
INFO 39356 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : D
INFO 39356 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : C
INFO 39356 --- [           main] c.w.s.ComponentScanTest                  : Started ComponentScanTest in 1.937 seconds (JVM running for 3.129)
INFO 39356 --- [           main] c.w.s.ComponentScanTest                  : test done
```  

- `myautoconfig`, `myautoconfig2` 자동설정을 활성화 한 경우에 대한 테스트이다. 
  - `Component Scan` 의 `Bean` 이 먼저 초기화 된 후, `Auto Configuration` 의 `Bean` 이 초기화 됐다. 
  - `Auto Coniguration` 이 초기화하는 `Bean` 의 경우 알파벳 순으로 초기화 된 거을 확인 할 수 있다.  

```java
@Slf4j
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class ComponentScanAndAutoConfigTest {
    @TestConfiguration
    public static class TestConfig {

        @Bean
        public EnableMyAutoConfigBean enableMyAutoConfigBean() {
            return new EnableMyAutoConfigBean();
        }
        @Bean
        public EnableMyAutoConfigBean2 enableMyAutoConfigBean2() {
            return new EnableMyAutoConfigBean2();
        }
    }

    @Test
    public void runTest() {
        log.info("test done");
    }
}
```  

```
INFO 44080 --- [           main] c.w.springbootinitbean.AComponentClass   : ACommonClass.PostConstruct
INFO 44080 --- [           main] c.w.springbootinitbean.BComponentClass   : BCommonClass.PostConstruct
INFO 44080 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : A
INFO 44080 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : B
INFO 44080 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : D
INFO 44080 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : C
INFO 44080 --- [           main] c.w.myautoconfig.AutoConfigClass         : AutoConfigClass.PostConstruct
INFO 44080 --- [           main] c.w.myautoconfigu2.AutoConfigClass2      : AutoConfigClass2.PostConstruct
INFO 44080 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : Started ComponentScanAndAutoConfigTest in 2.028 seconds (JVM running for 3.402)
INFO 44080 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : test done
```  


- `myautoconfig`, `myautoconfig2` 자동설정을 활성화 하면서 `@AutoConfigurerBefore` 사용(`ComponentScanAndAutoConfigTest` 로 테스트 수행)
  - `MyAutoConfig2` 에 `@AutoConfigureBefore(MyAutoConfig.class)` 이 선언 됐기 때문에, 
	`MyAutoConfig2` 가 초기화 된 후 `MyAutoConfig` 가 초기화 된것을 확인 할 수 있다.

```java
@Configuration
@ConditionalOnBean(EnableMyAutoConfigBean2.class)
@AutoConfigureBefore(MyAutoConfig.class)
public class MyAutoConfig2 {
    @Bean
    public AutoConfigClass2 autoConfigClass2() {
        return new AutoConfigClass2();
    }
}
```  

```java
@Slf4j
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class ComponentScanAndAutoConfigTest {
    @TestConfiguration
    public static class TestConfig {

        @Bean
        public EnableMyAutoConfigBean enableMyAutoConfigBean() {
            return new EnableMyAutoConfigBean();
        }
        @Bean
        public EnableMyAutoConfigBean2 enableMyAutoConfigBean2() {
            return new EnableMyAutoConfigBean2();
        }
    }

    @Test
    public void runTest() {
        log.info("test done");
    }
}
```  

```
INFO 54412 --- [           main] c.w.springbootinitbean.AComponentClass   : ACommonClass.PostConstruct
INFO 54412 --- [           main] c.w.springbootinitbean.BComponentClass   : BCommonClass.PostConstruct
INFO 54412 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : A
INFO 54412 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : B
INFO 54412 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : D
INFO 54412 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : C
INFO 54412 --- [           main] c.w.myautoconfigu2.AutoConfigClass2      : AutoConfigClass2.PostConstruct
INFO 54412 --- [           main] c.w.myautoconfig.AutoConfigClass         : AutoConfigClass.PostConstruct
INFO 54412 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : Started ComponentScanAndAutoConfigTest in 2.149 seconds (JVM running for 3.457)
INFO 54412 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : test done
```  


- `myautoconfig`, `myautoconfig2` 자동설정을 활성화 하면서 `@AutoConfigurerAfter` 사용 (`ComponentScanAndAutoConfigTest` 로 테스트 수행)
	- `MyAutoConfig` 에 `@AutoConfigureAfter(MyAutoConfig2.class)` 이 선언 됐기 때문에,
	  `MyAutoConfig2` 가 초기화 된 후 `MyAutoConfig` 가 초기화 된것을 확인 할 수 있다.

```java
@Configuration
@ConditionalOnBean(EnableMyAutoConfigBean.class)
@AutoConfigureAfter(MyAutoConfig2.class)
public class MyAutoConfig {
    @Bean
    public AutoConfigClass autoConfigClass() {
        return new AutoConfigClass();
    }
}
```

```
INFO 52140 --- [           main] c.w.springbootinitbean.AComponentClass   : ACommonClass.PostConstruct
INFO 52140 --- [           main] c.w.springbootinitbean.BComponentClass   : BCommonClass.PostConstruct
INFO 52140 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : A
INFO 52140 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : B
INFO 52140 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : D
INFO 52140 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : C
INFO 52140 --- [           main] c.w.myautoconfigu2.AutoConfigClass2      : AutoConfigClass2.PostConstruct
INFO 52140 --- [           main] c.w.myautoconfig.AutoConfigClass         : AutoConfigClass.PostConstruct
INFO 52140 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : Started ComponentScanAndAutoConfigTest in 2.056 seconds (JVM running for 3.272)
INFO 52140 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : test done
```  

- `myautoconfig`, `myautoconfig2` 자동설정을 활성화 하면서 `@AutuConfigurerOrder` 사용
	- `MyAutoConfig2` 에 `@AutoConfigureOrder(-100)` 이 선언 됐기 때문에,
	  `MyAutoConfig2` 가 초기화 된 후 `MyAutoConfig` 가 초기화 된것을 확인 할 수 있다.

```java
@Configuration
@ConditionalOnBean(EnableMyAutoConfigBean2.class)
@AutoConfigureOrder(-100)
public class MyAutoConfig2 {
    @Bean
    public AutoConfigClass2 autoConfigClass2() {
        return new AutoConfigClass2();
    }
}
```  

```
INFO 24564 --- [           main] c.w.springbootinitbean.AComponentClass   : ACommonClass.PostConstruct
INFO 24564 --- [           main] c.w.springbootinitbean.BComponentClass   : BCommonClass.PostConstruct
INFO 24564 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : A
INFO 24564 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : B
INFO 24564 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : D
INFO 24564 --- [           main] c.w.springbootinitbean.BeanClass         : BeanClass.PostConstruct : C
INFO 24564 --- [           main] c.w.myautoconfigu2.AutoConfigClass2      : AutoConfigClass2.PostConstruct
INFO 24564 --- [           main] c.w.myautoconfig.AutoConfigClass         : AutoConfigClass.PostConstruct
INFO 24564 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : Started ComponentScanAndAutoConfigTest in 2.241 seconds (JVM running for 3.614)
INFO 24564 --- [           main] c.w.s.ComponentScanAndAutoConfigTest     : test done
```  

---
## Reference