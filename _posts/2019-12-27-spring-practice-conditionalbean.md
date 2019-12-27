--- 
layout: single
classes: wide
title: "[Spring 실습] 조건부 빈"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '특정 조건에 따라 빈을 생성해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - Bean
  - Conditional
---  

## Conditional Annotation
- 애플리케이션이 구동될 때 모듈별 혹은 환경별 다른 빈을 등록해야 하는 경우가 있다.
- 상황에 따라 다른 빈을 등록을 해야할 때 `if else` 문을 사용할 수도 있지만 이는 가독성 등 다양한 측면에서 좋지 않을 수 있다.
- Spring 에서는 `Conditional Annotation` 을 사용해서 각 상황에 맞는 빈을 등록할 수 있도록 제공한다.
- `Conditional Annotation` 의 종류는 아래와 같다.
	- ConditionalOnBean
	- ConditionalOnClass
	- ConditionalOnCloudPlatform
	- ConditionalOnExpression
	- ConditionalOnJava
	- ConditionalOnJndi
	- ConditionalOnMissingBean
	- ConditionalOnMissingClass
	- ConditionalOnNotWebApplication
	- ConditionalOnProperty
	- ConditionalOnResource
	- ConditionalOnSingleCandidate
	- ConditionalOnWebApplication
- `Conditional Annotation` 은 Class 레벨, Method 레벨에 모두 사용가능 하다
	
## 테스트 프로젝트 구성

![그림 1]({{site.baseurl}}/img/spring/practice-conditionalbean-1.png)

- 프로젝트는 크게 parent 모듈과 web 모듈로 구성되어 있다.
- 모든 설정 클래스는 parent 모듈에 위치한다.
- 조건에 따라 parent 모듈, web 모듈에 테스트 코드가 위치한다.

### parent/pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <packaging>pom</packaging>
    <modules>
        <module>web</module>
    </modules>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.windowfosun</groupId>
    <artifactId>conditional-bean</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>conditional-bean</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```  

### parent/application.yml

```yaml
property:
  module: parent
  test1: true
```  

### web/pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>conditional-bean</artifactId>
        <groupId>com.windowfosun</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>web</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.windowfosun</groupId>
            <artifactId>conditional-bean</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```  

### web/application.yml

```yaml
property:
  module: web
  test1: false
```  


## ConditionalOnBean
- 명시한 빈의 이름이나 리턴 타입의 클래스가 BeanFactory 에 존재할 경우 빈이 생성된다.

```java
@Configuration
public class ConditionalOnBeanConfig {
    @Bean
    public A beanA() {
        return new A();
    }

    @Bean
    @ConditionalOnBean(name = "beanA")
    public B beanB() {
        // 이름이 beanA 인 빈이 존재하면 이 빈은 생성된다.
        return new B();
    }

    @Bean
    @ConditionalOnBean
    public C beanC() {
        // C 타입의 빈이 존재할 경우 이 빈은 생성된다.
        return new C();
    }

    @Bean
    public Object beanOtherA() {
        return new OtherA();
    }

    @Bean
    @ConditionalOnBean
    public Object beanOtherB() {
        // Object 타입의 빈이 존재할 경우 이 빈은 생성된다.
        return new OtherB();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnBeanConfig.class})
	public class ConditionalOnBeanConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test
	    public void beanB_Exists() {
	        B actual = this.applicationContext.getBean(B.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanC_NotExists() {
	        C actual = this.applicationContext.getBean(C.class);
	    }
	
	    @Test
	    public void beanOtherA_Exists() {
	        OtherA actual = this.applicationContext.getBean(OtherA.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test
	    public void beanOtherB_NotExists() {
	        OtherB actual = this.applicationContext.getBean(OtherB.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
## ConditionalOnClass
- 명시된 `value`, `name` 클래스가 classpath 에 존재할 경우 빈이 생성된다.
- `name` 은 classpath 에서 명시된 클래스의 사용 가능 여부가 확실치 않을 때 사용한다.

```java
@Configuration
public class ConditionalOnClassConfig {
    @Bean
    @ConditionalOnClass(value = {java.util.List.class})
    public A beanA() {
        // classpath에 java.util.List 클래스가 존재하면 이 빈은 생성된다.
        return new A();
    }

    @Bean
    @ConditionalOnClass(name = "com.notexists.Dummy")
    public B beanB() {
        // classpath에 com.noteixts.Dummy 클래스가 존재하면 이 빈은 생성된다.
        return new B();
    }

    @Bean
    @ConditionalOnClass(name = "com.windowfosun.conditionalbean.OtherA")
    public C beanC() {
        // classpath에 com.windowforsun.conditionalbean.OtherA 가 존재하면 이 빈은 생성된다.
        return new C();
    }
}
```  
	
- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnClassConfig.class})
	public class ConditionalOnClassConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	
	    @Test
	    public void beanC_Exists() {
	        C actual = this.applicationContext.getBean(C.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
## ConditionalOnMissingBean
- 명시된 빈의 이름이 `BeanFactory` 에 없는 경우 빈이 생성된다.
- Application Context 에서 빈의 정의가 처리 될때만 판별이 가능 하기 때문에 Auto-Configuration 클래스에서만 사용하는 것이 좋다.

```java
@Configuration
public class ConditionalOnMissingBeanConfig {
    @Bean
    public A beanA() {
        return new A();
    }

    @Bean
    @ConditionalOnMissingBean(name = "beanA")
    public B beanB() {
        // 이름이 beanA 인 빈이 없으면 이 빈은 생성된다.
        return new B();
    }

    @Bean
    @ConditionalOnMissingBean(name = "beanZ")
    public C beanC() {
        // 이름이 beanZ 인 빈이 없으면 이 빈은 생성된다.
        return new C();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnMissingBeanConfig.class})
	public class ConditionalOnMissingBeanConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	
	    @Test
	    public void beanC_Exists() {
	        C actual = this.applicationContext.getBean(C.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
## ConditionalOnMissingClass
- 명시된 클래스가 classpath 에 존재하지 않을 경우 빈이 생성된다.

```java
@Configuration
public class ConditionalOnMissingClassConfig {
    @Bean
    @ConditionalOnMissingClass(value = {"com.notexists.Dummy"})
    public A beanA() {
        // classpath 에 com.notexisxts.Dummy 클래스가 존재하지 않으면 이 빈은 생성된다.
        return new A();
    }

    @Bean
    @ConditionalOnMissingClass(value = {"com.windowfosun.conditionalbean.C"})
    public B beanB() {
        // classpath 에 com.windowforsun.conditionalbean.C 클래스가 존재하지 않으면 이 빈은 생성된다.
        return new B();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnMissingClassConfig.class})
	public class ConditionalOnMissingClassConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	}
	```  
	
## ConditionalOnWebApplication / ConditionalOnNotWebApplication
- 첫번째는 Application Context 가 WebApplication 일 경우에 빈이 등록된다.
- 두번째는 Application Context 가 WebApplication 이 아닐 경우 빈이 등록된다.

```java
@Configuration
public class ConditionalOnOrNotWebApplicationConfig {
    @Bean
    @ConditionalOnWebApplication
    public A beanA() {
        // WebApplication 일 경우 이 빈은 생성된다.
        return new A();
    }

    @Bean
    @ConditionalOnNotWebApplication
    public B beanB() {
        // WebApplication 이 아닐 경우 이 빈은 생성된다.
        return new B();
    }
}
```   

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnOrNotWebApplicationConfig.class})
	public class ConditionalOnOrNotWebApplicationConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanA_NotExists() {
	        A actual = this.applicationContext.getBean(A.class);
	    }
	
	    @Test
	    public void beanB_Exists() {
	        B actual = this.applicationContext.getBean(B.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
- web 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnOrNotWebApplicationConfig.class})
	public class ConditionalOnOrNotWebApplicationConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	}
	```  
	
## ConditionalOnResource
- 명시된 리소스가 classpath 나 경로에서 사용 가능 할때 빈이 생성된다.

```java
@Configuration
public class ConditionalOnResourceConfig {
    @Bean
    @ConditionalOnResource(resources = {"classpath:application.properties"})
    public A beanA() {
        // classpath:application.properties 리소스가 있을 경우 이 빈은 생성된다.
        return new A();
    }

    @Bean
    @ConditionalOnResource(resources = {"file:///c:/res/my-resource.txt"})
    public B beanB() {
        // c:/res/my-resource.txt 리소스가 있을 경우 이 빈은 생성된다.
        return new B();
    }

    @Bean
    @ConditionalOnResource(resources = {"file:///c:/res/not-exists.txt"})
    public C beanC() {
        // c:/res/not-exists.txt 리소스가 있을 경우 이 빈은 생성된다.
        return new C();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnResourceConfig.class})
	public class ConditionalOnResourceConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test
	    public void beanB_Exists() {
	        B actual = this.applicationContext.getBean(B.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanC_NotExists() {
	        C actual = this.applicationContext.getBean(C.class);
	    }
	}
	```  
	
## ConditionalOnJava
- 명시된 Java 버전 정보와 일치 할때 빈이 생성된다.
- `value` 와 `range` 를 통해 설정 할 수 있다.

```java
@Configuration
public class ConditionalOnJavaConfig {
    @Bean
    @ConditionalOnJava(value = JavaVersion.EIGHT)
    public A beanA() {
        // Java 버전이 8일 경우 이 빈은 생성된다.
        return new A();
    }

    @Bean
    @ConditionalOnJava(value = JavaVersion.ELEVEN)
    public B beanB() {
        // Java 버전이 11일 경우 이 빈은 생성된다.
        return new B();
    }

    @Bean
    @ConditionalOnJava(value = JavaVersion.TWELVE, range = ConditionalOnJava.Range.OLDER_THAN)
    public C beanC() {
        // Java 버전이 12 보다 낮을 경우 이 빈은 생성된다.
        return new C();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnJavaConfig.class})
	public class ConditionalOnJavaConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	
	    @Test
	    public void beanC_Exists() {
	        C actual = this.applicationContext.getBean(C.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
## ConditionalOnProperty
- 명시된 프로퍼티 의 정보와 값이 일치 할 경우 빈이 생성된다.
- `name`, `havingValue` 로 프로퍼티를 명시하고, `matchIfMissing` 는 일치하지 않을 경우 `true` 이면 빈 생성, `false` 이면 빈을 생성하지 않는다.

```java
@Configuration
public class ConditionalOnPropertyConfig {
    @Bean
    @ConditionalOnProperty(name = "property.module", havingValue = "parent")
    public A beanA() {
        return new A();
    }

    @Bean
    @ConditionalOnProperty(name = "property.module", havingValue = "web")
    public B beanB() {
        return new B();
    }

    @Bean
    @ConditionalOnProperty(name ="property.test1", havingValue = "true")
    public OtherA beanOtherA() {
        return new OtherA();
    }

    @Bean
    @ConditionalOnProperty(name = "property.test2", havingValue = "true", matchIfMissing = true)
    public OtherB beanOtherB() {
        return new OtherB();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = ConditionalOnPropertyConfig.class)
	public class ConditionalOnPropertyConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	
	    @Test
	    public void beanOtherA_Exists() {
	        OtherA actual = this.applicationContext.getBean(OtherA.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test
	    public void beanOtherB_Exists() {
	        OtherB actual = this.applicationContext.getBean(OtherB.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
- web 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = ConditionalOnPropertyConfig.class)
	public class ConditionalOnPropertyConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanA_NotExists() {
	        A actual = this.applicationContext.getBean(A.class);
	    }
	
	    @Test
	    public void beanB_Exists() {
	        B actual = this.applicationContext.getBean(B.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanOtherA_NotExists() {
	        OtherA actual = this.applicationContext.getBean(OtherA.class);
	    }
	
	    @Test
	    public void beanOtherB_Exists() {
	        OtherB actual = this.applicationContext.getBean(OtherB.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
## ConditionalOnExpression
- `ConditionalOnProperty` 보다 프로퍼티의 조건이 복잡한 경우 사용한다.

```java
@Configuration
public class ConditionalOnExpressionConfig {
    @Bean
    @ConditionalOnExpression("'${property.module}' == 'parent' and ${property.test1:true}")
    public A beanA() {
        return new A();
    }

    @Bean
    @ConditionalOnExpression("'${property.module}' == 'web' and ${property.test1:false}")
    public B beanB() {
        return new B();
    }

    @Bean
    @ConditionalOnExpression("'${property.module}' == 'parent' or ${property.test1:false}")
    public C beanC() {
        return new C();
    }
}
```  

- parent 모듈에서 테스트

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnExpressionConfig.class})
	public class ConditionalOnExpressionConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test
	    public void beanA_Exists() {
	        A actual = this.applicationContext.getBean(A.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanB_NotExists() {
	        B actual = this.applicationContext.getBean(B.class);
	    }
	
	    @Test
	    public void beanC_Exists() {
	        C actual = this.applicationContext.getBean(C.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  
	
- web 모듈에서 테스트
	- 실패 확인 필요
	
	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = {ConditionalOnExpressionConfig.class})
	public class ConditionalOnExpressionConfigTest {
	    @Autowired
	    private ApplicationContext applicationContext;
	
	    @Test(expected = NoSuchBeanDefinitionException.class)
	    public void beanA_NotExists() {
	        A actual = this.applicationContext.getBean(A.class);
	    }
	
	    @Test
	    public void beanB_Exists() {
	        B actual = this.applicationContext.getBean(B.class);
	
	        assertThat(actual, notNullValue());
	    }
	
	    @Test
	    public void beanC_Exists() {
	        C actual = this.applicationContext.getBean(C.class);
	
	        assertThat(actual, notNullValue());
	    }
	}
	```  

---
## Reference
