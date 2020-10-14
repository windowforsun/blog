--- 
layout: single
classes: wide
title: "[Spring 실습] Test Auto-configuration Annotations"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 목적에 맞는 테스트 환경을 구성해주는 어노테이션에 대해 알아보자'
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

`JUnit4` 를 사용하는 경우, 
`@RunWith(SpringRunner.class)` 가 선언되어야 한다.  

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
`@DataJdbcTest` 는 `JDBC` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 
해당 어노테이션이 설정된 테스트에 대해서는 전체 자동설정이 비활성화되고, 
`JDBC` 관련된 테스트 설정만 자동 설정으로 구성된다.  

`@DataJdbcTest` 어노테이션은 기본적으로 테스트 케이스가 끝날때마다, 
트랜잭션 및 롤백 처리를 수행한다. 
그리고 기본으로 인메모리 데이터베이스를 제공하기 때문에 별도의 데이터베이스 시스템을 구축하지 않아도 된다. 
필요에 따라 `@AutoConfigureTestDatabase` 어노테이션을 사용해서, 
별도로 구성한 데이터베이스를 사용한 테스트를 할 수 있다.  

만약 `@SpringBootTest` 어노테이션으로 통합테스트를 진행할때 인메모리 데이터베이스가 필요한 경우, 
`@AutoConfigureTestDatabase` 어노테이션을 통해 인메모리 데이터베이스를 사용할 수 있다.  

인메모리 데이터베이스를 사용하는 경우 테스트 코드의 예시는 아래와 같다.  

```java
@DataJdbcTest(
        properties = {
                "property.key=SampleDataJdbcTest"
        }
)
@RunWith(SpringRunner.class)
public class SpringBootDataJdbcTest {
    @Value("${property.key}")
    private String key;
    @Autowired
    private DataSource dataSource;
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    public void propertyTest() {
        assertThat(this.key, is("SampleDataJdbcTest"));
    }

    @Test
    public void dataSourceTest() throws Exception {
        // given
        String query = "select 111";
        Connection connection = this.dataSource.getConnection();
        PreparedStatement psmt = connection.prepareStatement(query);

        // when
        ResultSet rs = psmt.executeQuery();
        rs.next();
        int actual = rs.getInt(1);

        // then
        assertThat(actual, is(111));
    }

    @Test
    public void jdbcTemplateTest() {
        // given
        String query = "select 111";

        // when
        Map<String, Object> actual = jdbcTemplate.queryForMap(query);

        // then
        assertThat(actual, hasEntry("111", 111));
    }
}
```  

별도로 구성한 데이터베이스(`Docker MySQL`)를 사용하는 테스트 코드의 예시는 아래와 같다.  

```java
@DataJdbcTest
@RunWith(SpringRunner.class)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ContextConfiguration(initializers = {SpringBootRealDataJdbcTest.Initializer.class})
public class SpringBootRealDataJdbcTest {
    private static MySQLContainer mysql;

    static {
        mysql = new MySQLContainer("mysql:8.0.21");
        mysql.start();
    }

    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            TestPropertyValues.of(
                    "spring.datasource.url=" + mysql.getJdbcUrl(),
                    "spring.datasource.username=" + mysql.getUsername(),
                    "spring.datasource.password=" + mysql.getPassword()
            ).applyTo(applicationContext.getEnvironment());
        }
    }

    @Autowired
    private DataSource dataSource;

    @Test
    public void checkMySQLVersion() throws Exception{
        // given
        String query = "show variables like 'version'";
        Connection connection = this.dataSource.getConnection();
        PreparedStatement psmt = connection.prepareStatement(query);

        // when
        ResultSet rs = psmt.executeQuery();
        rs.next();
        String actual = rs.getString(2);

        // then
        assertThat(actual, is("8.0.21"));
    }
}
```  

### @DataJpaTest
`@DataJpaTest` 는 `JPA` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 
기능이나 설정의 대부분이 `@DataJdbcTest` 와 동일하고, 
차이점이 있다면 `JPA` 와 관련된 설정을 추가로 자동설정 한다는 부분이다.  

인메모리 데이터베이스를 사용하는 경우 테스트 코드의 예시는 아래와 같다.  

```java
@DataJpaTest(
        properties = {
                "property.key=SampleDataJpaTest"
        }
)
@RunWith(SpringRunner.class)
public class SpringBootDataJpaTest {
    @Value("${property.key}")
    private String key;
    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private TestEntityManager testEntityManager;

    @Test
    public void propertyTest() {
        assertThat(this.key, is("SampleDataJpaTest"));
    }

    @Test
    public void save() {
        // given
        Member member = new Member("test", "A");

        // when
        Member actual = this.memberRepository.save(member);

        // then
        assertThat(actual.getId(), greaterThan(0L));
    }

    @Test
    public void findById() {
        // given
        Member member = new Member("test", "A");
        member = this.memberRepository.save(member);

        // when
        Member actual = this.memberRepository.findById(member.getId()).orElse(null);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(member.getId()));
        assertThat(actual.getClassName(), is(member.getClassName()));
        assertThat(actual.getName(), is(member.getName()));
    }

    @Test
    public void testEntityManager() {
        // given
        Member member = this.testEntityManager.persist(new Member("test", "A"));

        // when
        Member actual = this.testEntityManager.find(Member.class, member.getId());

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(member.getId()));
        assertThat(actual.getClassName(), is(member.getClassName()));
        assertThat(actual.getName(), is(member.getName()));
    }
}
```  

별도로 구성한 데이터베이스(`Docker MySQL`)를 사용하는 테스트 코드의 예시는 아래와 같다.  

```java
@DataJpaTest
@RunWith(SpringRunner.class)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ContextConfiguration(initializers = {SpringBootRealDataJpaTest.Initializer.class})
public class SpringBootRealDataJpaTest {
    private static MySQLContainer mysql;

    static {
        mysql = new MySQLContainer("mysql:8.0.21");
        mysql.start();
    }

    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            TestPropertyValues.of(
                    "spring.datasource.url=" + mysql.getJdbcUrl(),
                    "spring.datasource.username=" + mysql.getUsername(),
                    "spring.datasource.password=" + mysql.getPassword()
            ).applyTo(applicationContext.getEnvironment());
        }
    }

    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private DataSource dataSource;

    @Test
    public void findById() {
        // given
        Member member = new Member("test", "A");
        member = this.memberRepository.save(member);

        // when
        Member actual = this.memberRepository.findById(member.getId()).orElse(null);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(member.getId()));
        assertThat(actual.getClassName(), is(member.getClassName()));
        assertThat(actual.getName(), is(member.getName()));
    }

    @Test
    public void checkMySQLVersion() throws Exception{
        // given
        String query = "show variables like 'version'";
        Connection connection = this.dataSource.getConnection();
        PreparedStatement psmt = connection.prepareStatement(query);

        // when
        ResultSet rs = psmt.executeQuery();
        rs.next();
        String actual = rs.getString(2);

        // then
        assertThat(actual, is("8.0.21"));
    }
}
```  

### @DataRedisTest
`@DataRedisTest` 는 `Redis` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 
해당 어노테이션이 설정된 테스트에 대해서는 전체 자동설정이 비활성화되고, 
`Redis` 관련된 테스트 설정만 자동 설정으로 구성된다.  

```java
@DataRedisTest(
        properties = {
                "property.key=SampleDataRedisTest"
        }
)
@RunWith(SpringRunner.class)
@ContextConfiguration(initializers = {SpringBootDataRedisTest.Initializer.class})
public class SpringBootDataRedisTest {
    private static GenericContainer redis;

    static {
        redis = new GenericContainer("redis:latest")
                .withExposedPorts(6379);
        redis.start();
    }

    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            TestPropertyValues.of(
                    "spring.redis.host=" + redis.getHost(),
                    "spring.redis.port=" + redis.getMappedPort(6379)
            ).applyTo(applicationContext.getEnvironment());
        }
    }

    @Value("${property.key}")
    private String key;
    @Autowired
    private RedisTemplate redisTemplate;
    @Autowired
    private MemberCountRepository memberCountRepository;

    @Test
    public void propertyTest() {
        assertThat(this.key, is("SampleDataRedisTest"));
    }

    @Test
    public void redisTemplate() {
        // given
        String key = "key1";
        String value = "value1";

        // when
        this.redisTemplate.opsForValue().set(key, value);

        // then
        String actual = this.redisTemplate.opsForValue().get(key) + "";
        assertThat(actual, is(value));
    }

    @Test
    public void memberCountRepository_findById() {
        // given
        MemberCount memberCount = new MemberCount("testClass", 1);
        this.memberCountRepository.save(memberCount);

        // when
        MemberCount actual = this.memberCountRepository.findById(memberCount.getClassName()).orElse(null);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getClassName(), is(memberCount.getClassName()));
        assertThat(actual.getCount(), is(memberCount.getCount()));
    }
}
```  

### @RestClientTest
`@DataRedisTest` 는 `RestTemplateBuilder` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 
즉 애플리케이션에서 `RestTemplate` 을 사용해 외부 `REST API` 호출하는 상황에 대한 테스트가 가능하다. 
해당 어노테이션이 설정된 테스트에 대해서는 전체 자동설정이 비활성화되고, 
`Rest Client` 관련된 테스트 설정만 자동 설정으로 구성된다. 
`Jaskcon`, `Gson` 과 같은 사용하는 `Json` 라이브러리들과 `@JsonComponent` 이 선언된 빈은 포함되지만, 
`@Component` 빈은 포함되지 않는다.  

`@RestClientTest` 어노테이션은 테스트를 위해 기본으로 `MockRestServiceServer` 빈을 자동설정 한다. 
`RestTemplate` 이 요청하는 가상 서버의 역할로, 
별도의 설정은 `@AutoConfigureMockRestServiceServer` 어노테이션을 사용해서 설정할 수 있다.  

만약 `REST API` 동작을 수행하는 부분에서 `RestTemplateBuilder` 를 사용하지 않고, 
`RestTemplate` 를 바로 사용할 경우에는 `@AutoConfigureWebClient(registerRestTemplate=true)` 으로 `RestTemplate` 빈을 설정할 수 있다. 

```java
@RestClientTest(
        value = ClassInfoService.class,
        properties = {
            "property.key=SampleRestClientTest"
        }
)
@RunWith(SpringRunner.class)
public class SpringBootRestClientTest {
    @Value("${property.key}")
    private String key;
    @Autowired
    private MockRestServiceServer server;
    @Autowired
    private ClassInfoService classInfoService;

    @Test
    public void propertyTest() {
        assertThat(this.key, is("SampleRestClientTest"));
    }

    @Test
    public void requestByJsonString() {
        // given
        String name = "testClass";
        String response = "{" +
                "\"name\":\"" + name + "\"," +
                "\"type\":1," +
                "\"detail\":\"this is test class\"" +
                "}";
        this.server.expect(requestToUriTemplate("http://localhost/class/{name}", name))
                .andRespond(withSuccess(response, MediaType.APPLICATION_JSON));

        // when
        ClassInfo actual = this.classInfoService.getByName(name);

        // then
        assertThat(actual.getName(), is(name));
        assertThat(actual.getType(), is(1));
        assertThat(actual.getDetail(), is("this is test class"));
    }

    @Test
    public void requestByJsonFile() {
        // given
        String name = "testClass";
        this.server.expect(requestToUriTemplate("http://localhost/class/{name}", name))
                .andRespond(withSuccess(new ClassPathResource("classinfo.json"), MediaType.APPLICATION_JSON));

        // when
        ClassInfo actual = this.classInfoService.getByName(name);

        // then
        assertThat(actual.getName(), is(name));
        assertThat(actual.getType(), is(1));
        assertThat(actual.getDetail(), is("this is test class"));
    }
}
```  

### @WebMvcTest
`@WebMvcTest` 는 `Spring MVC` 관련 테스트를 간편하게 할 수 있는 어노테이션이다. 
즉 애플리케이션에서 요청 처리를 수행하는 `Controller` 에 대한 테스트가 가능하다. 
해당 어노테이션이 설정된 테스트에 대해서는 전체 자동설정이 비활성화되고, 
`MVC` 관련된 테스트 설정만 자동 설정으로 구성된다.  

`@WebMvcTest` 는 기본으로 `MockMvc` 빈을 자동 설정한다. 
`MockMvc` 는 웹 애플리케이션을 서버에 배포하지 않고, `Spring MVC` 의 동작을 재현할 수 있는 클래스이다. 
`MockMvc` 의 간단한 흐름은 아래와 같다. 
1. 테스트 케이스에서 `MockMvc` 객체에 보낼 요청을 정의한다. 
1. `MockMvc` 는 설정된 요청을 `DispatcherServlet` 으로 보내고, 
`DispatcherServlet` 은 테스트용인 `TestDispatcherServlet` 으로 다시 전달한다. 
1. `TestDispatcherServlet` 은 요청을 받아 `DispatcherServlet` 과 동일한 흐름으로 요청을 처리하고, 
응답을 다시 `MockMvc` 에게 전달한다. 

`MockMvc` 에 대한 추가 설정은 `@AutoConfigureMockMvc` 어노테이션을 사용해서 가능하다. 
만약 통합 테스트인 `@SpringBootTest` 에서 `MockMvc` 를 사용하고 싶다면, 
`@AutoConfigureMockMvc` 어노테이션을 하는 방법으로 사용할 수 있다.  

또한 `MockMvc` 는 `@MockBean` 과 같은 모의 객체를 사용해서, 
컨트롤러에서 수행하는 복잡한 처리과정에 대한 결과를 모의로 정의해 테스트를 할 수 있다.  

```java
@WebMvcTest(
        value = MemberController.class,
        properties = {
                "property.key=SampleWebMvcTest"
        }
)
@RunWith(SpringRunner.class)
public class SpringBootWebMvcTest {
    @Value("${property.key}")
    private String key;
    @Autowired
    private MockMvc mockMvc;
    @MockBean
    private MemberService memberService;
    private ObjectMapper objectMapper = new ObjectMapper();

    @Test
    public void propertyTest() {
        assertThat(this.key, is("SampleWebMvcTest"));
    }

    @Test
    public void getById() throws Exception {
        // given
        Member member = new Member(111, "test", "testClass");
        given(this.memberService.getById(member.getId())).willReturn(member);

        // when
        this.mockMvc
                .perform(get("/member/{id}", member.getId())
                        .accept(MediaType.APPLICATION_JSON)
                        .contentType(MediaType.APPLICATION_JSON))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id", is((int) member.getId())))
                .andExpect(jsonPath("$.name", is(member.getName())))
                .andExpect(jsonPath("$.className", is(member.getClassName())))
                .andDo(print());
    }

    @Test
    public void create() throws Exception {
        // given
        Member before = new Member(0, "test", "testClass");
        Member after = new Member(111, "test", "testClass");
        given(this.memberService.add(any(Member.class))).willReturn(after);

        // when
        this.mockMvc
                .perform(post("/member")
                        .accept(MediaType.APPLICATION_JSON)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(this.objectMapper.writeValueAsString(before)))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id", is((int) after.getId())))
                .andExpect(jsonPath("$.name", is(after.getName())))
                .andExpect(jsonPath("$.className", is(after.getClassName())))
                .andDo(print());
    }
}
```  


### @SpringBootTest
`@SpringBootTest` 는 `Spring Boot` 기반 테스트를 간편하게 할 수 있는 어노테이션이다. 
지금까지 설명한 모든 테스트가 가능하고 간편하게 사용할 `ApplicationContext` 를 생성하고 조작할 수 있어서, 
주로 통합 테스트를 수행하는데 사용된다.  

먼저 `Bean` 과 관련된 설정은 `@SpringBootTest` 의 `classes` 필드가 있다. 
해당 필드에 테스트에서 사용할 빈들만 등록해서 테스트를 진행할 수 있다. 
`@Configuration` 이 선언된 클래스의 경우 내부에 선언된 `@Bean` 까지도 모두 등록된다. 
만약 `classes` 필드를 별도로 설정하지 않으면 애플리케이션 상에 정의된 모든 빈이 등록된다.  

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {ClassInfoService.class, RestTemplateConfig.class})
public class SpringBootTestClasses {
    @Autowired
    private ClassInfoService classInfoService;

    @Test
    public void testRun() {
        assertThat(this.classInfoService, notNullValue());
    }
}
```  

기존에 정의된 빈을 특정 테스트에서 재정의가 필요한 상황이 있다. 
이런 상황에서 `@TestConfiguration` 을 사용해서 기존에 정의된 빈을 재정의 할 수 있다. 
가장 간단한 방법은 아래와 같이 테스트 클래스안에 정의하는 방법이다. 

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class InnerTestConfigurationTest {
    @Autowired
    private ClassInfo testClassInfo;

    @Test
    public void Inner_TestConfiguration() {
        assertThat(this.testClassInfo.getName(), is("testConfig"));
        assertThat(this.testClassInfo.getType(), is(1));
        assertThat(this.testClassInfo.getDetail(), is("testConfigBean"));
    }

    @TestConfiguration
    public static class TestConfig {
        @Bean
        public ClassInfo testClassInfo() {
            return new ClassInfo("testConfig", 1, "testConfigBean");
        }
    }
}
```  

`@TestConfiguration` 또한 `ComponentScan` 을 통해 빈으로 설정된다. 
이러한 점으로 만약 `@SpringBootTest` 의 `classes` 필드에 설정할 특정 빈 클래스를 명시한 경우, 
`@TestConfiguration` 은 감지되지 않는다. 
위와 같은 상황에서는 `@TestConfiguration` 클래스를 `classes` 에 추가하거나, 
`@Import` 어노테이션을 사용해서 명시할 수 있다.  

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {ClassInfoService.class})
@Import(TestConfig2.class)
public class ImportTestConfigurationTest {
    @Autowired
    private ClassInfoService classInfoService;
    @Autowired
    private ClassInfo testClassInfo2;

    @Test
    public void Import_TestConfiguration() {
        assertThat(this.classInfoService, notNullValue());
        assertThat(this.testClassInfo2.getName(), is("testClassInfo2"));
        assertThat(this.testClassInfo2.getType(), is(2));
        assertThat(this.testClassInfo2.getDetail(), is("testClassInfoBean2"));
    }
}
```  

`@SpringBootTest` 또한 자동으로 `Mockito` 에 대한 설정을 수행하므로, 
`@MockBean` 어노테이션을 통해 모의 빈을 사용해 동작을 정의할 수 있다. 

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {MemberService.class})
public class MockBeanTest {
    @Autowired
    private MemberService memberService;
    @MockBean
    private MemberRepository memberRepository;
    @MockBean
    private MemberCountRepository memberCountRepository;

    @Test
    public void add() {
        // given
        Member member = new Member(111, "test", "testClass");
        MemberCount memberCount = new MemberCount("testClass", 1);
        given(this.memberCountRepository.findById(member.getClassName())).willReturn(Optional.of(memberCount));
        given(this.memberRepository.save(member)).willReturn(member);

        // when
        Member actual = this.memberService.add(member);

        // then
        assertThat(actual.getId(), is(111L));
        then(this.memberCountRepository).should().findById(member.getClassName());
        then(this.memberRepository).should().save(member);
    }
}
```  

`@SpringBootTest` 필드인 `webEnvironment` 를 사용하면 간편하게 웹 테스트 환경을 구성할 수 있고, 
아래와 같은 설정값을 제공한다. 
- `MOCK` : `WebApplicationContext` 로드하고, 내장된 `ServletContext` 가 아닌 `MockServletContainer` 를 제공한다. 
`@AutoConfigureMockMvc` 과 함께 사용하면 간편하게 `MockMvc` 를 통해 컨트롤러에 대한 테스트를 수행할 수 있다. 
- `RANDOM_PORT` : `EmbeddedWebApplicationContext` 를 로드하고, 실제 `ServletContext` 를 구성한다. 
생성된 `ServletContext` 는 `RANDOM_PORT` 에 해당하는 포트를 리슨한다. 
- `DEFINED_PORT` : 대부분의 설정이 `RANDOM_PORT` 와 비슷하지만, 
`application.yaml` 혹은 `application.properties` 에 정의한 포트를 리슨한다. 
- `NONE` : 일반적인 `ApplicationContext` 를 로드하고 `SerlvetContext` 와 같은 서블릿 환경은 구성하지 않는다. 


`@SpringBootTest` 의 `webEnvironment` 필드에 설정가능한 값 중, 
`ServletContext` 를 설정하는(`RANDOM_PORT`, `DEFINED_PORT`) 값을 사용한다면 
`TestRestTemplate` 을 사용할 수 있다. 
그리고 `TestRestTemplate` 은 구성된 테스트 환경에 `RestTemplate` 와 동일한 메소드를 사용해서 웹 통합 테스트를 진행할 수 있다. 
테스트시 요청을 하면 컨트롤러를 시작으로 애플리케이션 로직이 진행되는데, 
이와 비슷한 `MockMvc` 를 사용한 테스트와는 테스트 목적에서 부터 차이를 보인다. 
먼저 가장 큰 차이는 `ServletContext` 의 사용 여부이다. 
`TestRestTemplate` 를 사용한 테스트는 실제의 동작과 비슷한 `SerlvetContext` 를 사용하지만, 
`MockMvc` 를 사용한 테스트는 `Spring MVC` 처리가 모의 객체로 이뤄진다. 
그리고 `MockMvc` 는 서버가 처리하는 요청에 대한 비지니스 로직을 테스트 하는데 목적이 있다면, 
`TestRestTemplate` 는 클라이언트 입장에서 `RestTemplate` 을 사용해서 요청, 응답에 대한 테스라고 볼 수 있다. 

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class TestRestTemplateTest {
    @Autowired
    private TestRestTemplate testRestTemplate;
    @MockBean
    private MemberService memberService;

    @Test
    public void getById() {
        // given
        Member member = new Member(111, "test", "testClass");
        given(this.memberService.getById(member.getId())).willReturn(member);

        // when
        ResponseEntity<Member> actual = this.testRestTemplate.getForEntity("/member/{id}", Member.class, member.getId());

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.OK));
        assertThat(actual.getBody().getId(), is(member.getId()));
        then(this.memberService).should().getById(member.getId());
    }
}
```  

---
## Reference
[Spring Boot test annotation](http://wonwoo.ml/index.php/post/1926)  
[Spring Boot Test](https://cheese10yun.github.io/spring-boot-test/)  
[Test Auto-configuration Annotations](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html#test-auto-configuration)   
[Uses of Package org.springframework.boot.test.autoconfigure](https://docs.spring.io/spring-boot/docs/current/api/index.html?org/springframework/boot/test/autoconfigure/package-summary.html)   
