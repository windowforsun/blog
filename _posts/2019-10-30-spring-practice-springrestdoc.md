--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Rest Docs"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Rest Docs 로 API 문서 자동화를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - SpringBoot
    - RestDocs
    - Asciidoc
---  

# 목표
- Spring Rest Docs 에 대해 알아본다.
- Spring Rest Docs 을 사용해서 API 문서 자동화를 수행한다.

# 방법
## Spring Rest Docs 이란
- Spring Rest Docs 은 테스트 코드 기반으로 RESTful 문서생성을 돕는 도구이다.
- 기본적으로 AsciiDoc 을 사용해서 HTML 를 생성한다.
- Spring Test Framework 로 생성된 Snippet 을 사용해서 문서와 API 의 정확성을 보장한다.
- Junit, TestNG 등 다양한 테스트 프레임워크를 지원한다.
- MockMvc, WebTestClient, RestAssured 등 다양한 방식을 사용해서 Controller 에 대한 테스트를 진행 할 수 있다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springrestdoc-1.png)

## pom.xml

```xml
<!-- 생략 -->
	<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
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
        <!-- Spring Rest Docs -->
        <dependency>
            <groupId>org.springframework.restdocs</groupId>
            <artifactId>spring-restdocs-mockmvc</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Embedded Redis -->
        <dependency>
            <groupId>it.ozimov</groupId>
            <artifactId>embedded-redis</artifactId>
            <version>0.7.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <!-- AsciiDocs 플러그인 -->
                <groupId>org.asciidoctor</groupId>
                <artifactId>asciidoctor-maven-plugin</artifactId>
                <version>1.5.3</version>
                <executions>
                    <execution>
                        <id>generate-docs</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>process-asciidoc</goal>
                        </goals>
                        <configuration>
                            <backend>html</backend>
                            <doctype>book</doctype>
                        </configuration>
                    </execution>
                </executions>
                <dependencies>
                    <!-- Spring Rest Docs AsciiDoctor 의존성 추가 (Snippets 구성) -->
                    <dependency>
                        <groupId>org.springframework.restdocs</groupId>
                        <artifactId>spring-restdocs-asciidoctor</artifactId>
                        <version>2.0.2.RELEASE</version>
                    </dependency>
                </dependencies>
            </plugin>
            <plugin>
                <!-- AsciiDoctor 플러그인 설정 -->
                <artifactId>maven-resources-plugin</artifactId>
                <version>2.7</version>
                <executions>
                    <execution>
                        <id>copy-resources</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <!-- 문서가 출력되는 경로 설정 -->
                            <outputDirectory>
                                ${project.build.outputDirectory}/static/docs
                            </outputDirectory>
                            <resources>
                                <resource>
                                    <directory>
                                        ${project.build.directory}/generated-docs
                                    </directory>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
    <!-- 생략 -->
    </build>
<!-- 생략 -->
```  

## Config
- RedisConfig 설정 클래스에 Embedded Redis 설정을 작성한다.

	```java
	@Configuration
    public class RedisConfig {
        private RedisServer redisServer;
        @Value("${spring.redis.port}")
        private int port;
    
        @PostConstruct
        public void startRedisServer() throws IOException {
            this.redisServer = new RedisServer(this.port);
            this.redisServer.start();
        }
    
        @PreDestroy
        public void stopRedisServer() {
            if(this.redisServer != null) {
                this.redisServer.stop();
            }
        }
    
        @Bean
        public RedisConnectionFactory redisConnectionFactory() {
            RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
            redisStandaloneConfiguration.setPort(this.port);
    
            LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);
            return lettuceConnectionFactory;
        }
    
        @Bean
        public RedisTemplate<String, Object> redisTemplate() {
            RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
            redisTemplate.setConnectionFactory(this.redisConnectionFactory());
            redisTemplate.setDefaultSerializer(new StringRedisSerializer());
            redisTemplate.setKeySerializer(new StringRedisSerializer());
            redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
    
            return redisTemplate;
        }
    }
	```  
	
## application.yml

```yaml
spring:
  redis:
    port: 61133 # Embedded Redis 에서 사용할 포트
```  

## 구현내용
- Member 클래스에 Redis에 저장할 도메인을 정의한다.

	```java
	@Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @EqualsAndHashCode
    @RedisHash("Member")
    public class Member {
        @Id
        private long id;
        private String name;
        private String email;
        private int age;
    }
	```  
	
- MemberRepository 클래스를 통해 Redis 에 저장할 수있도록 인터페이스를 구현해준다.

	```java
	@Repository
    public interface MemberRepository extends CrudRepository<Member, Long> {
    }
	```  
	
- MemberController 클래스에 Member 도메인에 대한 RESTful API 를 구현한다.

	```java
	@RestController
    @RequestMapping("/member")
    public class MemberController {
        private MemberRepository memberRepository;
    
        @GetMapping
        public Map readAll() {
            List<Member> memberList = new LinkedList<>();
            Iterable<Member> iter = this.memberRepository.findAll();
            iter.forEach(memberList::add);
    
            Map<String, Object> map = new HashMap<>();
            map.put("members", memberList);
    
            return map;
        }
    
        @GetMapping("/{id}")
        public Member readById(@PathVariable long id) {
            return this.memberRepository.findById(id).orElse(null);
        }
    
        @PostMapping
        public Member create(@RequestBody Member member) {
            return this.memberRepository.save(member);
        }
    
        @PutMapping("/{id}")
        public Member update(@PathVariable long id, @RequestBody Member member) {
            member.setId(id);
            return this.memberRepository.save(member);
        }
    
        @DeleteMapping("/{id}")
        public Member deleteMember(@PathVariable long id) {
            Member member = this.memberRepository.findById(id).orElse(null);
    
            if(member instanceof Member) {
                this.memberRepository.deleteById(id);
            }
    
            return member;
        }
    
        @Autowired
        public void setMemberRepository(MemberRepository memberRepository) {
            this.memberRepository = memberRepository;
        }
    }
	```  
	
## 테스트 코드

```java
import static org.hamcrest.Matchers.*;
import static org.hamcrest.core.IsNull.notNullValue;
import static org.hamcrest.core.IsNull.nullValue;
import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.document;
import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.documentationConfiguration;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessRequest;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessResponse;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.prettyPrint;
import static org.springframework.restdocs.payload.PayloadDocumentation.*;
import static org.springframework.restdocs.request.RequestDocumentation.parameterWithName;
import static org.springframework.restdocs.request.RequestDocumentation.pathParameters;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.springframework.restdocs.mockmvc.RestDocumentationRequestBuilders.*;

@SpringBootTest
@RunWith(SpringRunner.class)
public class MemberControllerTest {
    /**
     * Junit 테스트 프레임워크를 사용할 경우 JUnitRestDocumentation 객체가 필요하다.
     */
    @Rule
    public JUnitRestDocumentation restDocumentation = new JUnitRestDocumentation();

    /**
     * 구현된 Application 의 Context 객체이다.
     */
    @Autowired
    private WebApplicationContext context;
    private RestDocumentationResultHandler document;
    private MockMvc mockMvc;

    private ObjectMapper objectMapper;
    @Autowired
    private MemberRepository memberRepository;

        @Before
    public void setUp() {
        this.objectMapper = new ObjectMapper();


        // <TestClassName>/<TestMethodName> 의 디렉토리 경로로 snippets 을 생성한다.
        this.document = document(
                "{class-name}/{method-name}",
                // 요청 문서 출력이 이쁘게 출력된다.
                preprocessRequest(prettyPrint()),
                // 응답 문서 출력이 이쁘게 출력된다.
                preprocessResponse(prettyPrint())
        );
        this.mockMvc = MockMvcBuilders.webAppContextSetup(this.context)
                .apply(documentationConfiguration(this.restDocumentation))
                // 모든 테스트에 대해서 snippets 을 생성한다.
                .alwaysDo(this.document)
                .build();
    }

    @Test
    public void readById() throws Exception {
        Member member = Member.builder()
                .id(1)
                .name("name")
                .email("abc@abc.com")
                .age(13)
                .build();
        this.memberRepository.save(member);

        this.mockMvc
                .perform(get("/member/{id}", 1l)
                        .accept(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isOk())
                .andDo(this.document.document(
                        // Path Parameter 에 대한 정의를 작성한다.
                        pathParameters(
                                parameterWithName("id").description("Member's id")
                        ),
                        // Response Body 에 대한 정의를 작성한다.
                        responseFields(
                                fieldWithPath("id").description("Member's id"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        )
                ))
                // 응답 값을 검증한다.
                .andExpect(jsonPath("$.name", is(notNullValue())))
                .andExpect(jsonPath("$.email", is(notNullValue())))
                .andExpect(jsonPath("$.age", is(notNullValue())))
                .andExpect(jsonPath("$.id", is(notNullValue())))
        ;
    }

    @Test
    public void create() throws Exception {
        Member member = Member.builder()
                .id(1)
                .name("name")
                .email("wwww@abc.com")
                .age(20)
                .build();

        this.mockMvc
                .perform(post("/member")
                        .accept(MediaType.APPLICATION_JSON)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(this.objectMapper.writeValueAsString(member)))
                .andDo(print())
                .andExpect(status().isOk())
                .andDo(this.document.document(
                        requestFields(
                                fieldWithPath("id").description("Member's id"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        ),
                        responseFields(
                                fieldWithPath("id").description("Member's id"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        )
                ))
                .andExpect(jsonPath("$.id", is(notNullValue())))
                .andExpect(jsonPath("$.name", is(notNullValue())))
                .andExpect(jsonPath("$.email", is(notNullValue())))
                .andExpect(jsonPath("age", is(notNullValue())))
        ;
    }

    @Test
    public void readAll() throws Exception {
        Member member = Member.builder()
                .id(1)
                .name("name")
                .email("wwww@abc.com")
                .age(20)
                .build();
        this.memberRepository.save(member);

        this.mockMvc
                .perform(get("/member")
                        .accept(MediaType.APPLICATION_JSON))
                .andDo(this.document.document(
                        responseFields(
                                fieldWithPath("members").description("Member's array"),
                                fieldWithPath("members[].id").description("Member's id"),
                                fieldWithPath("members[].name").description("Member's name"),
                                fieldWithPath("members[].email").description("Member's email"),
                                fieldWithPath("members[].age").description("Member's age")
                        )
                ))
                .andExpect(jsonPath("$.members", not(emptyArray())))
                .andExpect(jsonPath("$.members[0].name", is(notNullValue())))
                .andExpect(jsonPath("$.members[0].id", is(notNullValue())))
                .andExpect(jsonPath("$.members[0].age", is(notNullValue())))
                .andExpect(jsonPath("$.members[0].email", is(notNullValue())))
        ;
    }

    @Test
    public void update() throws Exception {
        Member member = Member.builder()
                .id(1)
                .name("name")
                .email("wwww@abc.com")
                .age(20)
                .build();
        this.memberRepository.save(member);
        member.setAge(22);
        member.setEmail("aaaa@addddd.com");

        this.mockMvc
                .perform(put("/member/{id}", 1l)
                        .accept(MediaType.APPLICATION_JSON)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(this.objectMapper.writeValueAsString(member)))
                .andDo(print())
                .andExpect(status().isOk())
                .andDo(this.document.document(
                        pathParameters(
                                parameterWithName("id").description("Member's id")
                        ),
                        // Request Body 에 대한 정의를 작성한다.
                        requestFields(
                                fieldWithPath("id").description("This field not use"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        ),
                        responseFields(
                                fieldWithPath("id").description("Member's id"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        )
                ))
                .andExpect(jsonPath("$.id", is(notNullValue())))
                .andExpect(jsonPath("$.name", is(notNullValue())))
                .andExpect(jsonPath("$.email", is(notNullValue())))
                .andExpect(jsonPath("$.age", is(notNullValue())))
        ;
    }

    @Test
    public void deleteMember() throws Exception {
        Member member = Member.builder()
                .id(1)
                .name("name")
                .email("wwww@abc.com")
                .age(20)
                .build();
        this.memberRepository.save(member);

        this.mockMvc
                .perform(delete("/member/{id}", 1l))
                .andDo(print())
                .andExpect(status().isOk())
                .andDo(this.document.document(
                        pathParameters(
                                parameterWithName("id").description("Member's id")
                        ),
                        responseFields(
                                fieldWithPath("id").description("Member's id"),
                                fieldWithPath("name").description("Member's name"),
                                fieldWithPath("email").description("Member's email"),
                                fieldWithPath("age").description("Member's age")
                        )

                ))
                .andExpect(jsonPath("$.id", is(notNullValue())))
                .andExpect(jsonPath("$.name", is(notNullValue())))
                .andExpect(jsonPath("$.email", is(notNullValue())))
                .andExpect(jsonPath("$.age", is(notNullValue())))
        ;
    }
}
```  

## API 문서 작성하기
- 작성된 테스트가 성공하면 `<build-path>/genterated-snippets/<test-classname>/<test-methodname>` 의 하위에 관련 `adoc` 문서가 생성된다.

	![그림 2]({{site.baseurl}}/img/spring/practice-springrestdoc-2.png)
	
- 생성된 `adoc` 파일을 통해 문서를 작성하기 위해서는 뼈대를 구성해야 한다.
- `src/main/asciidoc/` 경로에 `api-guide.adoc` 혹은 `index.adoc` 파일만들고 아래와 같이 틀을 구성해준다.

	```adoc
	= RESTful Notes API Guide
    windowforsun;
    :doctype: book
    :icons: font
    :source-highlighter: highlightjs
    :toc: left
    :toclevels: 4
    :sectlinks:
    :site-url: /build/asciidoc/html5/
    
    [[overview]]
    = Overview
    
    [[overview-http-verbs]]
    == HTTP verbs
    
    RESTful notes tries to adhere as closely as possible to standard HTTP and REST conventions in its
    use of HTTP verbs.
    
    |===
    | Verb | Usage
    
    | `GET`
    | Used to retrieve a resource
    
    | `POST`
    | Used to create a new resource
    
    | `PUT`
    | Used to update an existing resource, including partial updates
    
    | `DELETE`
    | Used to delete an existing resource
    |===
    
    [[overview-http-status-codes]]
    == HTTP status codes
    
    RESTful notes tries to adhere as closely as possible to standard HTTP and REST conventions in its
    use of HTTP status codes.
    
    |===
    | Status code | Usage
    
    | `200 OK`
    | The request completed successfully
    
    | `201 Created`
    | A new resource has been created successfully. The resource's URI is available from the response's
    `Location` header
    
    | `204 No Content`
    | An update to an existing resource has been applied successfully
    
    | `400 Bad Request`
    | The request was malformed. The response body will include an error providing further information
    
    | `404 Not Found`
    | The requested resource did not exist
    |===
    
    = Member
    == Member 전체 조회
    === Request
    include::{snippets}/member-controller-test/read-all/path-parameters.adoc[]
    include::{snippets}/member-controller-test/read-all/http-request.adoc[]
    === Response
    include::{snippets}/member-controller-test/read-all/http-response.adoc[]
    include::{snippets}/member-controller-test/read-all/response-fields.adoc[]
    include::{snippets}/member-controller-test/read-all/response-body.adoc[]
    
    
    == Member 조회
    === Request
    include::{snippets}/member-controller-test/read-by-id/path-parameters.adoc[]
    include::{snippets}/member-controller-test/read-by-id/http-request.adoc[]
    === Response
    include::{snippets}/member-controller-test/read-by-id/http-response.adoc[]
    include::{snippets}/member-controller-test/read-by-id/response-fields.adoc[]
    
    == Member 생성
    === Request
    include::{snippets}/member-controller-test/create/http-request.adoc[]
    include::{snippets}/member-controller-test/create/request-fields.adoc[]
    === Response
    include::{snippets}/member-controller-test/create/http-response.adoc[]
    include::{snippets}/member-controller-test/create/response-fields.adoc[]
    include::{snippets}/member-controller-test/create/response-body.adoc[]
    
    == Member 갱신
    === Request
    include::{snippets}/member-controller-test/update/path-parameters.adoc[]
    include::{snippets}/member-controller-test/update/http-request.adoc[]
    include::{snippets}/member-controller-test/update/request-fields.adoc[]
    === Response
    include::{snippets}/member-controller-test/update/http-response.adoc[]
    include::{snippets}/member-controller-test/update/response-fields.adoc[]
    include::{snippets}/member-controller-test/update/response-body.adoc[]
    
    == Member 삭제
    === Request
    include::{snippets}/member-controller-test/update/path-parameters.adoc[]
    include::{snippets}/member-controller-test/update/http-request.adoc[]
    include::{snippets}/member-controller-test/update/request-fields.adoc[]
    === Response
    include::{snippets}/member-controller-test/update/http-response.adoc[]
    include::{snippets}/member-controller-test/update/response-fields.adoc[]
    include::{snippets}/member-controller-test/update/response-body.adoc[]
	```  
	
- `mvn install`, `mvn spring-boot:run` 을 통해 빌드 및 애플리케이션을 실한다.
- `http://<server-ip>:<server-port>/docs/index.html` 에 접속하면 아래와 같은 결과 화면을 확인 할 수 있다.
	
	![그림 3]({{site.baseurl}}/img/spring/practice-springrestdoc-3.png)


## Reference
[Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/1.1.2.RELEASE/reference/html5/)  
[Asciidoctor User Manual](https://asciidoctor.org/docs/user-manual/)  
[AsciiDoc Syntax Quick Reference](https://asciidoctor.org/docs/asciidoc-syntax-quick-reference/)  
[Spring REST Docs v1.0.1 레퍼런스](https://springboot.tistory.com/26)  
[Spring Rest Docs 적용](http://woowabros.github.io/experience/2018/12/28/spring-rest-docs.html)  
[Spring REST Docs를 사용한 API 문서 자동화](https://velog.io/@kingcjy/Spring-REST-Docs%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-API-%EB%AC%B8%EC%84%9C-%EC%9E%90%EB%8F%99%ED%99%94)  
[Spring REST Docs API 문서를 자동화 해보자](https://www.popit.kr/spring-rest-docs/)  
[Generating documentation for your REST API with Spring REST Docs](https://dimitr.im/spring-rest-docs)  
[Introduction to Spring REST Docs](https://www.baeldung.com/spring-rest-docs)  
[[spring] REST DOcs 사용중 'urlTemplate not found. If you are using MockMvc did you use RestDocumentationRequestBuilders to build the request?' 발생시](https://java.ihoney.pe.kr/517)  
[refer1](https://jojoldu.tistory.com/294)  
[refer2](http://wonwoo.ml/index.php/post/476)  
[refer3](https://blog.naver.com/PostView.nhn?blogId=varkiry05&logNo=221388209806&categoryNo=107&parentCategoryNo=0&viewDate=&currentPage=1&postListTopCurrentPage=1&from=search&userTopListOpen=true&userTopListCount=5&userTopListManageOpen=false&userTopListCurrentPage=1)  
