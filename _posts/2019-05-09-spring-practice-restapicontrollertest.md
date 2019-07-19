--- 
layout: single
classes: wide
title: "[Spring 실습] Simple Rest API Unit Test"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '간단한 Rest API 의 Unit Test 를 진행해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - REST
    - Unit Test
    - Junit
---  

# 목표
- Spring 에서 Rest API 의 Unit Test 를 수행한다.
- 간단하고 빠른 Test 구현으로 반복적은 테스트를 수행한다.
- 단위 테스트를 수행하여 API 단위 로직에 대한 안정성을 보장한다.

# 방법
- Rest API Unit Test 를 위한 의존성을 추가한다.
- Rest API 에서 주고 받을 Model 객체를 만든다.
- Rest API 로 사용할 Controller 를 만든다.
- 필요에 따라 Service 클래스도 구현해 준다.
- 테스트에 필요한 설정 클래스를 작성한다.
- Junit 을 통해 Test 클래스에 Test Case 를 작성한다.

# 예제
## 의존성 추가
- pom.xml 에 아래와 같은 의존성이 필요하다.

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${org.springframework-version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>${org.springframework-version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>${org.springframework-version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>${org.springframework-version}</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>RELEASE</version>
</dependency>
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path-assert</artifactId>
    <version>2.2.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>2.23.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.0.1</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.3</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.20</version>
</dependency>
```  

- jackjson 의존성을 추가해서 Rest API 에서 json 관련 연산을 자동적으로 하도록 한다.
- lombok 의존성을 추가해서 구현 클래스에서 기본적인 메서드를 Annotation 으로 구현하도록 한다.

## Model 클래스 만들기
- Test 용 Rest API 에서 사용할 Model 객체를 만든다.
- Serialize/UnSerialize 를 위해 기본 생성자는 필수로 있어야 한다.

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class TestModel {
    private String key;
    private int num;
    private String str;
    private ArrayList<String> list;
    private HashMap<String, String> map;
}
```  

## Service 클래스 만들기
- Service 클래스는 간단한 구조로 되어 있다.
- 클래스 변수인 Map 을 통해 Model 객체를 관리하며 CURD 관련 연산이 구현되어 있다.

```java
@Service
public class TestService {
    private static HashMap<String, TestModel> modelMap = new HashMap<>();

    public TestModel create(TestModel model) {
        TestModel result = null;

        if (!modelMap.containsKey(model.getKey())) {
            modelMap.put(model.getKey(), model);

            result = model;
        }

        return result;
    }

    public TestModel update(TestModel model) {
        TestModel result = null;

        if(modelMap.containsKey(model.getKey())) {
            modelMap.put(model.getKey(), model);

            result = model;
        }

        return result;
    }

    public TestModel indate(TestModel model) {
        modelMap.put(model.getKey(), model);

        return model;
    }

    public TestModel remove(String key) {
        return modelMap.remove(key);
    }

    public TestModel read(String key) {
        return modelMap.get(key);
    }

    public List<TestModel> readList(List<String> keyList) {
        LinkedList<TestModel> result = new LinkedList<>();

        for (String key : keyList) {
            if(modelMap.containsKey(key)) {
                result.addLast(this.read(key));
            }
        }

        return result;
    }

    public List<TestModel> readAll() {
        return modelMap.values().stream().collect(Collectors.toList());
    }

    public void removeAll() {
        modelMap.clear();
    }
```  

## Controller 클래스 만들기
- 아래는 HTTP 의 POST 메서드로만 만든 API Controller 클래스 이다.
- HTTP 의 POST 메서드의 Body 에 json 형식으로 여러 데이터를 넣으면 POST 메소드로만 다양한 API 구현이 가능하다.
	- 이는 Restful 하지 못하지만, 개발 스펙에 따라 사용하기도 한다.

```java
@RestController
@RequestMapping("/test-post")
public class TestPostController {

    @Autowired
    private TestService service;

    @RequestMapping(value = "/echo", method = RequestMethod.POST)
    public String echo(@RequestBody String str) {
        return str;
    }

    @RequestMapping(value = "/echoList", method = RequestMethod.POST)
    public List<Object> echoList(@RequestBody List<Object> list) {
        return list;
    }

    @RequestMapping(value = "/create", method = RequestMethod.POST)
    public TestModel create(@RequestBody TestModel model) {
        return this.service.create(model);
    }

    @RequestMapping(value = "/read", method = RequestMethod.POST)
    public TestModel read(@RequestBody String key) {
        return this.service.read(key);
    }

    @RequestMapping(value = "/update", method = RequestMethod.POST)
    public TestModel update(@RequestBody TestModel model) {
        return this.service.update(model);
    }

    @RequestMapping(value = "/remove", method = RequestMethod.POST)
    public TestModel remove(@RequestBody String key) {
        return this.service.remove(key);
    }

    @RequestMapping(value = "/readList", method = RequestMethod.POST)
    public List<TestModel> readList(@RequestBody List<String> keyList) {
        return this.service.readList(keyList);
    }

    @RequestMapping(value = "/readAll", method = RequestMethod.POST)
    public List<TestModel> readAll() {
        return this.service.readAll();
    }
}
```  

- 아래는 Rest 형식으로 작성한 API Controller 클래스 이다.

```java
@RestController
@RequestMapping("/test-rest")
public class TestRestController {

    @Autowired
    private TestService testService;

    @GetMapping
    public List<TestModel> findAll() {
        return this.testService.readAll();
    }

    @GetMapping("/{key}")
    public TestModel findOne(@PathVariable String key) {
        return this.testService.read(key);
    }

    @GetMapping("/[{keyList}]")
    public List<TestModel> findList(@PathVariable String[] keyList) {
        return this.testService.readList(Arrays.asList(keyList));
    }

    @PostMapping
    public TestModel create(@RequestBody TestModel model) {
        return this.testService.create(model);
    }

    @PatchMapping
    public TestModel update(@RequestBody TestModel model) {
        return this.testService.update(model);
    }

    @PutMapping
    public TestModel indate(@RequestBody TestModel model) {
        return this.testService.indate(model);
    }

    @DeleteMapping("/{key}")
    public TestModel delete(@PathVariable String key) {
        return this.testService.remove(key);
    }
}

```  

## 설정 클래스 작성하기

```java
@Configuration
@ComponentScan("research.rest.exmy")
public class TestControllerConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}
```  

## Test 클래스에 Test Case 작성하기
### POST API Controller Test

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.hamcrest.CoreMatchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
import static org.springframework.test.web.servlet.setup.MockMvcBuilders.standaloneSetup;

@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration(classes = TestControllerConfig.class)
public class TestPostControllerTest {
    public MockMvc mockMvc;
    @Autowired
    private ObjectMapper objectMapper;
    @Autowired
    private TestPostController testPostController;
    @Autowired
    private TestService testService;
    private TestModel defaultModel;

    @Before
    public void setUp() throws Exception {
        this.mockMvc = standaloneSetup(this.testPostController).build();


        ArrayList<String> strList = new ArrayList<>();
        strList.add("a");
        strList.add("b");

        HashMap<String, String> map = new HashMap<>();
        map.put("a", "a");
        map.put("b", "b");

        this.defaultModel = new TestModel("key1", 1, "a", strList, map);
    }

    @After
    public void tearDown() throws Exception{
        this.testService.removeAll();
    }

    @Test
    public void echo() throws Exception {
        String content = "Hello World!";

        this.mockMvc.perform(post("/test-post/echo")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void test() {
        Map<String, Object> map = new HashMap<>();
        Object o = map.put("a", "a");
        Assert.assertNull(o);
        o = map.put("a", "b");
        System.out.println(o.toString());
        System.out.println(map);
    }

    @Test
    public void echoList() throws Exception {
        List<Object> list = new ArrayList<>();
        list.add("a");
        list.add("b");

        String content = this.objectMapper.writeValueAsString(list);

        this.mockMvc.perform(post("/test-post/echoList")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void create_Success() throws Exception {
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-post/create")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void create_Fail() throws Exception {
        this.testService.create(this.defaultModel);

        String expected = "";
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-post/create")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void read_Success() throws Exception{
        this.testService.create(this.defaultModel);

        String expected = this.objectMapper.writeValueAsString(this.defaultModel);
        String content = "key1";

        this.mockMvc.perform(post("/test-post/read")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void read_Fail() throws Exception{
        String expected = "";
        String content = "key1";

        this.mockMvc.perform(post("/test-post/read")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void update_Success() throws Exception {
        this.testService.create(this.defaultModel);

        this.defaultModel.setStr("newStr");
        this.defaultModel.setNum(1000000);

        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-post/update")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void update_Fail() throws Exception {
        this.defaultModel.setStr("newStr");
        this.defaultModel.setNum(1000000);

        String expected = "";
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-post/update")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void remove_Success() throws Exception {
        this.testService.create(this.defaultModel);

        String expected = this.objectMapper.writeValueAsString(this.defaultModel);
        String content = "key1";

        this.mockMvc.perform(post("/test-post/remove")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void remove_Fail() throws Exception {
        String expected = "";
        String content = "key1";

        this.mockMvc.perform(post("/test-post/remove")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void readList_Success() throws Exception{
        this.testService.create(this.defaultModel);

        TestModel model2 = new TestModel();
        model2.setKey("key2");
        model2.setNum(22);
        model2.setStr("2222");
        this.testService.create(model2);

        List<String> keyList = new ArrayList<>();
        keyList.add("key1");
        keyList.add("key2");
        keyList.add("key3");

        List<TestModel> expectedList = this.testService.readList(keyList);

        String expected = this.objectMapper.writeValueAsString(expectedList);
        String content = this.objectMapper.writeValueAsString(keyList);

        this.mockMvc.perform(post("/test-post/readList")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void readList_None() throws Exception{
        List<String> keyList = new ArrayList<>();
        keyList.add("key1");
        keyList.add("key2");
        keyList.add("key3");

        List<TestModel> expectedList = this.testService.readList(keyList);

        String expected = "[]";
        String content = this.objectMapper.writeValueAsString(keyList);

        this.mockMvc.perform(post("/test-post/readList")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void readAll() throws Exception{
        this.testService.create(this.defaultModel);

        TestModel model2 = new TestModel();
        model2.setKey("key2");
        model2.setNum(22);
        model2.setStr("2222");
        this.testService.create(model2);

        List<TestModel> expectedList = this.testService.readAll();

        String expected = this.objectMapper.writeValueAsString(expectedList);

        this.mockMvc.perform(post("/test-post/readAll")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }
}
```  

### REST API Controller Test

```java

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.hamcrest.CoreMatchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
import static org.springframework.test.web.servlet.setup.MockMvcBuilders.standaloneSetup;

@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration(classes = TestControllerConfig.class)
public class TestRestControllerTest {
    private MockMvc mockMvc;
    @Autowired
    private ObjectMapper objectMapper;
    @Autowired
    private TestRestController testRestController;
    @Autowired
    private TestService testService;
    private TestModel defaultModel;

    @Before
    public void setUp() throws Exception {
        this.mockMvc = standaloneSetup(this.testRestController).build();

        ArrayList<String> strList = new ArrayList<>();
        strList.add("a");
        strList.add("b");

        HashMap<String, String> map = new HashMap<>();
        map.put("a", "a");
        map.put("b", "b");

        this.defaultModel = new TestModel("key1", 1, "a", strList, map);
    }

    @After
    public void tearDown() throws Exception {
        this.testService.removeAll();
    }

    @Test
    public void create_Ok() throws Exception {
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void create_Fail() throws Exception {
        this.testService.create(this.defaultModel);

        String expected = "";
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(post("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    public void findOne_Ok() throws Exception {
        this.testService.create(this.defaultModel);

        String expected = this.objectMapper.writeValueAsString(this.defaultModel);
        String key = "key1";
        this.mockMvc.perform(get("/test-rest/" + key)
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void findList_Ok() throws Exception {
        this.testService.create(this.defaultModel);

        TestModel model2 = new TestModel();
        model2.setKey("key2");
        model2.setNum(22);
        model2.setStr("2222");
        this.testService.create(model2);

        List<String> keyList = new ArrayList<>();
        keyList.add("key1");
        keyList.add("key2");
        keyList.add("key3");

        List<TestModel> expectedList = this.testService.readList(keyList);

        String expected = this.objectMapper.writeValueAsString(expectedList);
        String content = this.objectMapper.writeValueAsString(keyList);

        this.mockMvc.perform(get("/test-rest/" + Arrays.toString(keyList.toArray()))
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void update_Ok() throws Exception {
        this.testService.create(this.defaultModel);

        this.defaultModel.setStr("aaaaaaaaaaaaaaaa");
        this.defaultModel.setNum(11111111);

        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(patch("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void update_Fail() throws Exception {
        this.defaultModel.setStr("aaaaaaaaaaaaaaaa");
        this.defaultModel.setNum(11111111);

        String expected = "";
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(patch("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void indate_Ok() throws Exception {
        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(put("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void indate2_Ok() throws Exception {
        this.testService.create(this.defaultModel);

        this.defaultModel.setStr("aaaaaaaaaaaaaaaa");
        this.defaultModel.setNum(11111111);

        String content = this.objectMapper.writeValueAsString(this.defaultModel);

        this.mockMvc.perform(put("/test-rest")
                .contentType(MediaType.APPLICATION_JSON)
                .content(content))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(content)));
    }

    @Test
    public void delete_Ok() throws Exception {
        this.testService.create(this.defaultModel);

        String expected = this.objectMapper.writeValueAsString(this.defaultModel);
        String key = "key1";

        this.mockMvc.perform(delete("/test-rest/" + key)
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }

    @Test
    public void delete_Fail() throws Exception {
        String expected = "";
        String key = "key1";

        this.mockMvc.perform(delete("/test-rest/" + key)
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo(expected)));
    }
}
```  

---
## Reference
[RESTful API Example with Spring Boot, Unit Test with MockMVC and Simple UI Integration with VueJS](https://hellokoding.com/restful-apis-example-with-spring-boot-integration-test-with-mockmvc-ui-integration-with-vuejs/)  
[How to Spring MVC Unit Test 스프링 MVC 단위 테스트](https://thswave.github.io/java/2015/03/02/spring-mvc-test.html)  
[Spring에서 REST 서비스를 위한 컨트롤러 생성과 컨트롤러 단위테스트 하기](http://blog.saltfactory.net/create-and-test-rest-conroller-in-spring/)  
[Testing a Spring Boot RESTful Service](https://medium.com/@tbrouwer/testing-a-spring-boot-restful-service-c61b981cac61)  
[Unit Testing of Spring MVC Controllers: REST API](https://www.petrikainulainen.net/programming/spring-framework/unit-testing-of-spring-mvc-controllers-rest-api/)  
[Spring REST Hello World Example](https://www.mkyong.com/spring-boot/spring-rest-hello-world-example/)  
[Spring REST Integration Test Example](https://www.mkyong.com/spring-boot/spring-rest-integration-test-example/)  
[Spring REST Validation Example](https://www.mkyong.com/spring-boot/spring-rest-validation-example/)  
