--- 
layout: single
classes: wide
title: "[Spring 실습] Test Auto-configuration Annotations"
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
    - Junit
    - Auto-configuration
    - Test Annotation
toc: true
use_math: true
---  

## Test Auto-configuration Annotations
`Spring Boot` 에서 `@...Test` 와 같이 `Test` 로 끝나는 어노테이션은 테스트에 해당하는 설정을 자동 설정해주는 역할을 한다. 
어노테이션의 종류와 자동 대상이되는 리스트는 [여기](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html#test-auto-configuration)
에서 확인할 수 있다.  

본 포스트에서 모든 종류를 다루지 않으므로 설명할 어노테이션은 아래와 같다. 

Annotation|Desc|Bean
---|---|---
@SpringBootTest|통합 테스트, 전체|Bean 전체
@WebMvcTest|단위 테스트, MVC 테스트|MVC 관련 Bean
@DataJpaTest|단위 테스트, JPA 테스트|JPA 관련 Bean
@DataJdbcTest|단위 테스트, JDBC 테스트|JDBC 관련 Bean
@DataRedisTest|단위 테스트, Redis 테스트|Redis 관련 Bean
@RestClientTest|단위 테스트, Rest API 테스트|Rest API 관련 Bean
@JsonTest|단위 테스트, Json 테스트|Json 관련 Bean

테스트하고자 하는 목적에 맞는 적절한 어노테이션을 사용하면, 
목적에 필요한 `ApplicationContext` 만 자동 설정을 수행하므로 좀더 가볍고 빠른 테스트를 수행할 수 있다. 
그리고 몇개의 어노테이션에서는 테스트를 보다 간편하게 할 수 있는 빈을 제공한다. 
기본적으로 테스트관련 어노테이션에서는 테스트에 고유하게 사용할 수 있는 `Properties` 설정이 가능하다. 

### @JsonTest
`@JsonTest` 어노테이션을 사용하면 `Json` 관련 `Serialize`, `Deserialize` 결과를 쉽게 테스트 할 수 있다. 
`@JsonComponent`, `Jackson` 모듈 등을 자동 설정한다. 
기본적으로 `BasicJsonTester` 가 빈으로 등록되어 사용할 수 있는데, `Json` 문자열에 대한 검증만 가능하다. 
그리고 `Jackson` 혹은 `Gson` 라이브러리를 사용한다면 해당하는 빈인 `..Tester` 가 자동으로 등록돼 보다 다양한 테스트를 수행할 수 있다. 

```java
@JsonTest(
        properties = {
                "property.key=SampleJsonTest"
        }
)
@RunWith(SpringRunner.class)
public class SpringBootJsonTest {
    @Value("${property.key}")
    private String key;
    @Autowired
    private BasicJsonTester basicJsonTester;
    @Autowired
    private JacksonTester<ClassInfo> json;

    @Test
    public void propertyTest() {
        Assert.assertEquals("SampleJsonTest", this.key);
    }

    @Test
    public void basicJsonTester_Serialize() throws Exception {
        // given
        String jsonString = "{" +
                "\"name\":\"testClass\"," +
                "\"type\":1," +
                "\"detail\":\"this is test class\"" +
                "}";
        // when
        JsonContent<Object> actual = this.basicJsonTester.from(jsonString);

        // then
        assertThat(actual)
                .extractingJsonPathValue("$.name")
                .isEqualTo("testClass");
        assertThat(actual)
                .extractingJsonPathValue("$.type")
                .isEqualTo(1);
        assertThat(actual)
                .extractingJsonPathValue("$.detail")
                .isEqualTo("this is test class");

    }

    @Test
    public void jacksonTester_Serialize() throws Exception {
        // given
        ClassInfo classInfo = new ClassInfo("testClass", 1, "this is test class");

        // when
        JsonContent<ClassInfo> actual = this.json.write(classInfo);

        // then
        assertThat(actual)
                .extractingJsonPathValue("$.name")
                .isEqualTo("testClass");
        assertThat(actual)
                .extractingJsonPathValue("$.type")
                .isEqualTo(1);
        assertThat(actual)
                .extractingJsonPathValue("$.detail")
                .isEqualTo("this is test class");
    }

    @Test
    public void jacksonTester_Deserialize() throws Exception {
        // given
        String jsonString = "{" +
                "\"name\":\"testClass\"," +
                "\"type\":1," +
                "\"detail\":\"this is test class\"" +
                "}";

        // when
        ObjectContent<ClassInfo> actual = this.json.parse(jsonString);

        // then
        assertThat(actual)
                .hasFieldOrPropertyWithValue("name", "testClass")
                .hasFieldOrPropertyWithValue("type", 1)
                .hasFieldOrPropertyWithValue("detail", "this is test class");
    }
}
```  

### @DataJdbcTest
`@DataJdbcTest` 는 `Jdbc` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 









































































### @SpringBootTest
















































---
## Reference
[Spring Boot test annotation](http://wonwoo.ml/index.php/post/1926)  
[Spring Boot Test](https://cheese10yun.github.io/spring-boot-test/)  
[Test Auto-configuration Annotations](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html#test-auto-configuration)   
[Uses of Package org.springframework.boot.test.autoconfigure](https://docs.spring.io/spring-boot/docs/current/api/index.html?org/springframework/boot/test/autoconfigure/package-summary.html)   
