--- 
layout: single
classes: wide
title: "[Spring 실습] RestTemplate"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '손쉽게 REST API 요청/응답 처리를 할 수 있는 RestTemplate 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - RestTemplate
  - Restful
---  

## RestTemplate
- Spring 3.0 부터 지원하는 HTTP 통신 템플릿이다.
- HTTP 서버와의 통신을 단쉰화하고 RESTful 원칙을 지킨다.
- `JdbcTemplate` 와 같이 기계적이고 반복적인 코드를 깜끔하게 작성할 수 있도록 도와준다.
- 쉽게 `Json`, `XML`, `String` 등 응답을 받아 처리할 수 있다.
- 내부적으로는 `HttpClient` 를 사용한다.
- `AsyncRestTemplate` 은 Spring 4.0 부터 추가된 비동기 `RestTemplate` 이다.
	- Spring 5.0 에서는 Deprecated 되었다고 한다.
- Spring 5.0 부터는 `WebClient` 를 통해 Non-Blocking, Reactive, Async 방식을 지원한다.

## RestTemplate 에서 제공하는 메소드

Method|HTTP Method|Desc
---|---|---
getForObject|GET|URL 주소에 HTTP GET 메소드로 요청해 객체 결과를 리턴한다.
getForEntity|GET|URL 주소에 HTTP GET 메소드로 요청해 `ResponseEntity` 결과를 리턴한다.
postForObject|POST|URL 주소에 HTTP POST 메소드로 요청해 객체 결과를 리턴한다.
postForEntity|POST|URL 주소에 HTTP POST 메소드로 요청해 `ResponseEntity` 결과를 리턴한다.
postForLocation|POST|URL 주소에 HTTP POST 메소드로 요청해 헤더에 저장된 URL 결과를 리턴한다.
put|PUT|URL 주소에 HTTP PUT 메소드로 요청한다.
delete|DELETE|URL 주소에 HTTP DELETE 메소드로 요청한다.
patchForObject|PATCH|URL 주소에 HTTP PATCH 메소드로 요청해 객체 결과를 리턴한다.
exchange|any|URL 주소에 HTTP 헤더를 설정하고 어떤 HTTP 메소드(모두가능)로 요청해서 `ResponseEntity` 결과를 리턴한다.
optionsForAllow|OPTIONS|URL 주소에서 지원하는 HTTP 메소드를 조회해 `Set` 으로 리턴한다.
execute|any|RestTemplate 에서 기반이 되는 요청 메소드로 Request, Response 을 처리하는 Callback 메소드를 등록해 사용할 수 있다.

## RestTemplate Collection Pool
- `RestTemplate` 에서는 기본적으로 Connection Pool 을 지원하지 않아, 메소드 수행시마다 새로운 커넥션을 위해 `handshake` 를 수행한다.
- 반복적인 요청을 위해 Connection Pool 을 설정하기 위해서는 `Apache` 의 `HttpClient` 의 의존성이 필요하다.
	
	```xml
	<dependency>
       <groupId>org.apache.httpcomponents</groupId>
       <artifactId>httpclient</artifactId>
       <version>4.5.9</version>
    </dependency>
	```  
	
```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
        // 구현 요구사항에 따라 커스텀하게 설정할 수 있다.
        factory.setConnectTimeout(3000);
        HttpClient httpClient = HttpClientBuilder
                .create()
                .setMaxConnTotal(100)
                .setMaxConnPerRoute(10)
                .build();
        factory.setHttpClient(httpClient);

        return new RestTemplate(factory);
    }
}
```  

- `HttpComponentClientHttpRequestFactory` 를 통해 타임아웃, 버퍼링 등에 대한 설정을 할 수 있다.
- `HttpClient` 를 통해 Connection Pool 설정을 할 수 있다.
	- `setMaxConnTotal` 은 Connection Pool 의 총 개수
	- `setMaxConnPerRoute` 는 요청하는 HOST 당 연결 제한 수
	
		
## RestTemplate Logging
- `RestTemplate` 는 HTTP 통신시 `Interceptor` 를 사용해서 요청과 응답 로그를 남길 수 있다.
- `RestTemplate` 의 `Body` 는 `Stream` 이기 때문에 `Interceptor` 에서 `Body` 를 읽게되면 실제로 응답을 받아야하는 코드에서 응답을 받지 못하게 된다.
- 위 상황을 위해 `RestTemplate` 빈 생성시에 `BufferingClientHttpRequestFactory` 를 설정해 준다.

```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplateBuilder()
                .setConnectTimeout(Duration.ofSeconds(2))
                .setReadTimeout(Duration.ofSeconds(2))
                // BufferingClientHttpRequestFactory 등록
                .requestFactory(() -> new BufferingClientHttpRequestFactory(new SimpleClientHttpRequestFactory()))
                // Interceptor 등록
                .additionalInterceptors(new RestTemplateLoggingInterceptor())
                .build();

        return restTemplate;
    }
}
```  

```java
@Slf4j
public class RestTemplateLoggingInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes, ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {

        URI uri = httpRequest.getURI();
        log.info("request uri: {}, method: {}, body: {}", httpRequest.getURI(), httpRequest.getMethod(), new String(bytes, StandardCharsets.UTF_8));

        ClientHttpResponse response = clientHttpRequestExecution.execute(httpRequest,bytes);

        log.info("response uri: {}, status: {}, body: {}", uri, response.getStatusCode(), StreamUtils.copyToString(response.getBody(), StandardCharsets.UTF_8));

        return response;
    }
}
```  

	
## 예제
### pom.xml

```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.9</version>
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
```  

## RestTemplateConfig

```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpClient httpClient = HttpClientBuilder
            .create()
            .setMaxConnTotal(100)
            .setMaxConnPerRoute(10)
            .build();
        HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
        factory.setHttpClient(httpClient);
        BufferingClientHttpRequestFactory bufferingFactory = new BufferingClientHttpRequestFactory(factory);

        RestTemplate restTemplate = new RestTemplateBuilder()
                .setConnectTimeout(Duration.ofSeconds(2))
                .setReadTimeout(Duration.ofSeconds(2))
                .requestFactory(() -> bufferingFactory)
                .additionalInterceptors(new RestTemplateLoggingInterceptor())
//                .additionalMessageConverters(new StringHttpMessageConverter(Charset.forName("UTF-8")))
                .build();

        return restTemplate;
    }
}
```  

### ExamModel

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
public class ExamModel {
    private String id;
    private int num;
    @Builder.Default
    private List<String> strList = new LinkedList<>();

    public ExamModel addStrList(String str) {
        this.strList.add(str);

        return this;
    }
}
```  

### ExamController

```java
@RestController
@RequestMapping("/exam")
public class ExamController {
    // 데이터 저장소
    public static Map<String, ExamModel> map = new HashMap<>();

    // 모든 데이터 읽기
    @GetMapping
    public ResponseEntity<List<ExamModel>> findAll() {
        List<ExamModel> list = new LinkedList<>();

        for (Map.Entry<String, ExamModel> entry : this.map.entrySet()) {
            list.add(entry.getValue());
        }

        return ResponseEntity.ok(list);
    }

    @GetMapping("/range/{equals}")
    public ResponseEntity<List<ExamModel>> findByList(@RequestParam int start, @RequestParam int end, @PathVariable int equals) {
        List<ExamModel> list = new LinkedList<>();
        ExamModel examModel;

        for (Map.Entry<String, ExamModel> entry : map.entrySet()) {
            examModel = entry.getValue();
            if (examModel.getNum() >= start && examModel.getNum() <= end || examModel.getNum() == equals) {
                list.add(examModel);
            }
        }

        return ResponseEntity.ok(list);
    }

    // 특정 데이터 읽기
    @GetMapping("/{id}")
    public ResponseEntity<ExamModel> findById(@PathVariable String id) {
        ExamModel examModel = map.get(id);
        ResponseEntity responseEntity;

        if (examModel == null) {
            responseEntity = ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
        } else {
            responseEntity = ResponseEntity.ok(examModel);
        }

        return responseEntity;
    }

    // 새로운 데이터 생성
    @PostMapping
    public ResponseEntity<ExamModel> create(@RequestBody ExamModel examModel) throws URISyntaxException {
        ResponseEntity responseEntity;

        if(examModel == null || examModel.getId() == null) {
            responseEntity = ResponseEntity.badRequest().build();
        } else if (map.containsKey(examModel.getId())) {
            responseEntity = ResponseEntity.status(HttpStatus.CONFLICT).build();
        } else {
            map.put(examModel.getId(), examModel);
            responseEntity = ResponseEntity.created(new URI("/exam/" + examModel.getId())).body(examModel);
        }

        return responseEntity;
    }

    // 새로운 데이터 생성할 때 Header 값에 있는 인증정보를 검사 후 진행
    @PostMapping("/headers")
    public ResponseEntity<ExamModel> createAndCheckHeader(@RequestBody ExamModel examModel,
                                                          @RequestHeader(value = "Authentication", required = true) String authentication) throws URISyntaxException {
        ResponseEntity responseEntity;

        if(!authentication.equals("this is key")) {
            responseEntity = ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }else if(examModel == null) {
            responseEntity = ResponseEntity.badRequest().build();
        } else if (map.containsKey(examModel.getId())) {
            responseEntity = ResponseEntity.status(HttpStatus.CONFLICT).build();
        } else {
            map.put(examModel.getId(), examModel);
            responseEntity = ResponseEntity.created(new URI("/exam/" + examModel.getId())).body(examModel);
        }

        return responseEntity;
    }

    // 특정 데이터 갱신
    @PutMapping("/{id}")
    public ResponseEntity<Void> update(@RequestBody ExamModel examModel, @PathVariable String id) {
        ResponseEntity responseEntity;

        if (id == null || id.isEmpty() || examModel == null) {
            responseEntity = ResponseEntity.badRequest().build();
        } else if (map.containsKey(id)) {
            examModel.setId(id);
            map.put(id, examModel);
            responseEntity = ResponseEntity.ok().build();
        } else {
            responseEntity = ResponseEntity.notFound().build();
        }

        return responseEntity;
    }

    // 특정 데이터의 특정 필드 갱신
    @PatchMapping("/{id}")
    public ResponseEntity<Void> updateNum(@RequestBody int num, @PathVariable String id) {
        ResponseEntity responseEntity;

        if (id == null || id.isEmpty()) {
            responseEntity = ResponseEntity.badRequest().build();
        } else if (map.containsKey(id)) {
            ExamModel examModel = map.get(id);
            examModel.setNum(num);
            map.put(id, examModel);
            responseEntity = ResponseEntity.ok().build();
        } else {
            responseEntity = ResponseEntity.notFound().build();
        }

        return responseEntity;
    }

    // 특정 데이터 삭제
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteById(@PathVariable String id) {
        ResponseEntity responseEntity;

        if (map.containsKey(id)) {
            map.remove(id);
            responseEntity = ResponseEntity.ok().build();
        } else {
            responseEntity = ResponseEntity.notFound().build();
        }

        return responseEntity;
    }

    // 모든 데이터 삭제
    @DeleteMapping
    public ResponseEntity<Void> delete() {
        map.clear();

        return ResponseEntity.ok().build();
    }

    // 응답이 오래걸리는 동작
    @GetMapping("/sleep")
    public ResponseEntity<ExamModel> sleep() throws Exception {
        Thread.sleep(3000);

        return ResponseEntity.ok(null);
    }
}
```  

### 테스트

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class RestTemplateTest {
    @Autowired
    private RestTemplate restTemplate;
    private String url;
    @LocalServerPort
    private int port;

    @Before
    public void setUp() {
        this.url = "http://localhost:" + this.port + "/exam";
        ExamController.map.clear();

        ExamController.map.put(
                "a",
                ExamModel.builder()
                        .id("a")
                        .num(1)
                        .build()
                        .addStrList("a")
                        .addStrList("b")
        );
        ExamController.map.put(
                "b",
                ExamModel.builder()
                        .id("b")
                        .num(2)
                        .build()
                        .addStrList("aa")
                        .addStrList("bb")
        );
        ExamController.map.put(
                "c",
                ExamModel.builder()
                        .id("c")
                        .num(3)
                        .build()
                        .addStrList("aaa")
                        .addStrList("bbb")
        );
    }

    @Test
    public void getForObject() {
        // given
        String path = "/{id}";
        this.url += path;
        String id = "a";

        // when
        ExamModel actual = this.restTemplate.getForObject(this.url, ExamModel.class, id);

        // then
        assertThat(actual, is(notNullValue()));
        assertThat(actual.getId(), is(id));
    }

    @Test
    public void getForEntity() {
        // when
        ResponseEntity<ExamModel[]> actual = this.restTemplate.getForEntity(this.url, ExamModel[].class);

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.OK));
        assertThat(actual.getBody(), arrayWithSize(3));
    }

    @Test
    public void getForEntity_WithParameters() {
        // given
        String path = "/range/{equals}?start={start}&end={end}";
        this.url += path;
        Map<String, Integer> params = new HashMap<>();
        params.put("start", 1);
        params.put("end", 2);
        params.put("equals", 3);

        // when
        ResponseEntity<ExamModel[]> actual = this.restTemplate.getForEntity(this.url, ExamModel[].class, params);

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.OK));
        assertThat(actual.getBody(), arrayWithSize(3));
    }

    @Test
    public void postForObject() {
        // given
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");

        // when
        ExamModel actual = this.restTemplate.postForObject(this.url, examModel, ExamModel.class);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(examModel.getId()));
        assertThat(actual.getNum(), is(examModel.getNum()));
        assertThat(actual.getStrList(), is(examModel.getStrList()));
    }

    @Test
    public void postForObject_WithHeaders() {
        // given
        String path = "/headers";
        this.url += path;
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");
        String authenication = "this is key";
        HttpHeaders headers = new HttpHeaders();
        headers.set("authentication", authenication);
        HttpEntity<ExamModel> request = new HttpEntity<>(examModel, headers);

        // when
        ExamModel actual = this.restTemplate.postForObject(this.url, request, ExamModel.class);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(examModel.getId()));
        assertThat(actual.getNum(), is(examModel.getNum()));
        assertThat(actual.getStrList(), is(examModel.getStrList()));
    }

    @Test
    public void postForEntity() {
        // given
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");

        // when
        ResponseEntity<ExamModel> actual = this.restTemplate.postForEntity(this.url, examModel, ExamModel.class);

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.CREATED));
        assertThat(actual.getBody(), notNullValue());
        assertThat(actual.getBody().getId(), is(examModel.getId()));
        assertThat(actual.getBody().getNum(), is(examModel.getNum()));
        assertThat(actual.getBody().getStrList(), is(examModel.getStrList()));
    }

    @Test
    public void postForLocation() {
        // given
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");
        HttpEntity<ExamModel> request = new HttpEntity<>(examModel);

        // when
        URI actual = this.restTemplate.postForLocation(this.url, request);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getPath(), startsWith("/exam/"));
    }

    @Test
    public void put() {
        // given
        String path = "/{id}";
        this.url += path;
        String id = "a";
        Map<String, String> params = new HashMap<>();
        params.put("id", id);
        ExamModel examModel = ExamModel.builder()
                .num(11)
                .build()
                .addStrList("str11")
                .addStrList("str22");

        // when
        this.restTemplate.put(this.url, examModel, params);

        // then
        ExamModel actual = ExamController.map.get(id);
        assertThat(actual, notNullValue());
        assertThat(actual.getNum(), is(11));
        assertThat(actual.getStrList(), hasItems("str11", "str22"));
    }

    @Test
    public void delete() {
        // given
        String path = "/{id}";
        this.url += path;
        String id = "a";
        Map<String, String> params = new HashMap<>();
        params.put("id", id);

        // when
        this.restTemplate.delete(this.url, params);

        // then
        ExamModel actual = ExamController.map.get(id);
        assertThat(actual, nullValue());
    }

    @Test
    public void patchForObject() {
        // given
        String path = "/{id}";
        this.url += path;
        String id = "a";
        int num = 111;

        // when
        this.restTemplate.patchForObject(this.url, num, Void.class, id);

        // then
        ExamModel actual = ExamController.map.get(id);
        assertThat(actual.getNum(), is(num));
    }

    @Test
    public void exchange_GET() {
        // given
        String path = "/{id}";
        this.url += path;

        // when
        ResponseEntity<ExamModel> actual = this.restTemplate.exchange(this.url, HttpMethod.GET, null, ExamModel.class, "a");

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.OK));
        assertThat(actual.getBody().getId(), is("a"));
    }

    @Test
    public void exchange_GET_Collections() {
        // when
        ResponseEntity<List<ExamModel>> actual = this.restTemplate.exchange(this.url, HttpMethod.GET, null, new ParameterizedTypeReference<List<ExamModel>>(){});

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.OK));
        assertThat(actual.getBody(), hasSize(3));
    }

    @Test
    public void exchange_POST() {
        // given
        String path = "/headers";
        this.url += path;
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");
        String authentication = "this is key";
        HttpHeaders headers = new HttpHeaders();
        headers.set("authentication", authentication);
        HttpEntity<ExamModel> request = new HttpEntity<>(examModel, headers);

        // when
        ResponseEntity<ExamModel> actual = this.restTemplate.exchange(this.url, HttpMethod.POST, request, ExamModel.class);

        // then
        assertThat(actual.getStatusCode(), is(HttpStatus.CREATED));
        assertThat(actual.getBody(), notNullValue());
        assertThat(actual.getBody().getId(), is(examModel.getId()));
        assertThat(actual.getBody().getNum(), is(examModel.getNum()));
        assertThat(actual.getBody().getStrList(), is(examModel.getStrList()));
    }

    @Test
    public void optionsForAllow() {
        // when
        Set<HttpMethod> actual = this.restTemplate.optionsForAllow(this.url);

        // then
        assertThat(actual, not(empty()));
    }

    @Test(expected = ResourceAccessException.class)
    public void timeout() {
        // given
        String path = "/sleep";
        this.url += path;

        // when
        this.restTemplate.getForObject(this.url, ExamModel.class);
    }


    @Test
    public void execute_GET() {
        // given
        String path = "/{id}";
        this.url += path;
        String id = "a";

        // when
        ExamModel actual = this.restTemplate.execute(this.url, HttpMethod.GET, this.getRequestCallback(), this.getResponseExtractor(), id);

        // then
        assertThat(actual, is(notNullValue()));
        assertThat(actual.getId(), is(id));

    }

    public RequestCallback getRequestCallback() {
        return clientHttpRequest -> {
            clientHttpRequest.getHeaders().add(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_UTF8_VALUE);
        };
    }

    public ResponseExtractor<ExamModel> getResponseExtractor() {
        return clientHttpResponse -> {
            ObjectMapper objectMapper = new ObjectMapper();
            return objectMapper.readValue(clientHttpResponse.getBody(), ExamModel.class);
        };
    }

    @Test
    public void execute_POST() {
        // given
        ExamModel examModel = ExamModel.builder()
                .id("d")
                .num(4)
                .build()
                .addStrList("str1")
                .addStrList("str2");

        // when
        ExamModel actual = this.restTemplate.execute(this.url, HttpMethod.POST, this.postRequestCallback(examModel), this.postResponseExtractor());

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(examModel.getId()));
        assertThat(actual.getNum(), is(examModel.getNum()));
        assertThat(actual.getStrList(), is(examModel.getStrList()));
    }

    public RequestCallback postRequestCallback(final ExamModel examModel) {
        return clientHttpRequest -> {
            ObjectMapper objectMapper = new ObjectMapper();
            objectMapper.writeValue(clientHttpRequest.getBody(), examModel);
            clientHttpRequest.getHeaders().add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_UTF8_VALUE);
            clientHttpRequest.getHeaders().add(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_UTF8_VALUE);
        };
    }

    public ResponseExtractor<ExamModel> postResponseExtractor() {
        return clientHttpResponse -> {
          ObjectMapper objectMapper = new ObjectMapper();
          return objectMapper.readValue(clientHttpResponse.getBody(), ExamModel.class);
        };
    }
}
```  
	

---
## Reference
[Spring boot TestRestTemplate POST with headers example](https://howtodoinjava.com/spring-boot2/testing/testresttemplate-post-example/)  
[The Guide to RestTemplate](https://www.baeldung.com/rest-template)  
[RestTemplate (정의, 특징, URLConnection, HttpClient, 동작원리, 사용법, connection pool 적용)](https://sjh836.tistory.com/141)  
[마로의 Spring Framework 공부 - RestTemplate](https://hoonmaro.tistory.com/46)  
