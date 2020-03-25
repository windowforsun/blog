--- 
layout: single
classes: wide
title: "[Spring 실습] Filter, Interceptor 를 이용한 요청/응답 값 컨트롤"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '요청/응답 값을 Filter 를 사용해서 컨트롤 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - Filter
  - Interceptor
---  

## 요청/응답 값 컨트롤
- 요청/응답 값을 공통적으로 컨트롤해야 하는 경우가 있다.
- 암/복호화, 값 수정, 로깅 등의 경우가 있을 수 있다.
- `Filter` 나 `Interceptor` 에 있는 `ServletRequest`, `ServletResponse`, `HttpServletRequest`, `HttpServletResponse` 는 몇가지 제약사항이 있어 컨트롤이 불가능하다.
- `HttpServletRequestWrapper`, `HttpServletResponseWrapper` 을 사용하면 요청/응답 값을 원하는대로 컨트롤 할 수 있다.

## 예제
- `HttpServletRequestWrapper`, `HttpServletResponseWrapper` 을 통해 요청/응답을 `Wrapping` 한다.
- `Filter` 와 `Interceptor` 를 사용해서 요청/응답에 원하는 값을 추가하는 방식으로 예제를 진행한다.

### 디렉토리 구조

```bash
src
├─main
│  ├─java
│  │  └─com
│  │      └─windowforsun
│  │          └─exam
│  │              │  ExamApplication.java
│  │              │
│  │              ├─advice
│  │              │      ExceptionAdvice.java
│  │              │
│  │              ├─config
│  │              │      AppConfig.java
│  │              │
│  │              ├─controller
│  │              │  │  AllController.java
│  │              │  │
│  │              │  └─a
│  │              │          AController.java
│  │              │
│  │              ├─exception
│  │              │      CustomException.java
│  │              │
│  │              ├─filter
│  │              │      AFilter.java
│  │              │      AllFilter.java
│  │              │
│  │              ├─interceptor
│  │              │      AInterceptor.java
│  │              │      AllInterceptor.java
│  │              │
│  │              └─wrapper
│  │                      CustomServletInputStream.java
│  │                      CustomServletOutputStream.java
│  │                      ModifyRequestWrapper.java
│  │                      ModifyResponseWrapper.java
│  │
│  └─resources
│      │  application.properties
│      │
│      ├─static
│      └─templates
└─test
  └─java
      └─com
          └─windowforsun
              └─exam
                  │
                  └─controller
                          AControllerTest.java
                          AllControllerTest.java
```  

### pom.xml

```xml
<dependencies>
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

### Application

```java
@SpringBootApplication
public class ExamApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExamApplication.class, args);
    }
}
```  

### Java Config

```java
@Configuration
public class AppConfig implements WebMvcConfigurer {
    @Autowired
    private AllInterceptor allInterceptor;
    @Autowired
    private AInterceptor aInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(this.allInterceptor).addPathPatterns("/**").order(Ordered.HIGHEST_PRECEDENCE + 1);
        registry.addInterceptor(this.aInterceptor).addPathPatterns("/a/**").order(Ordered.HIGHEST_PRECEDENCE + 2);
    }
    
    @Bean
    public FilterRegistrationBean allFilter() {
        FilterRegistrationBean bean = new FilterRegistrationBean(new AllFilter());
        bean.setOrder(1);
        bean.addUrlPatterns("/*");

        return bean;
    }

    @Bean
    public FilterRegistrationBean aFilter() {
        FilterRegistrationBean bean = new FilterRegistrationBean(new AFilter());
        bean.setOrder(2);
        bean.addUrlPatterns("/a/*");
        return bean;
    }
}
```  

- `Filter` 는 설정된 우선순위에 따라 `AllFilter`, `AFilter` 순으로 실행된다.
	- `AllFilter` 는 모든 URL 에서 실행된다.
	- `AFilter` 는 `/a` 의 하위 URL 에서 실행된다.
- `Interceptor` 도 설정된 우선순위에 따라 `AllInterceptor`, `AInterceptor` 순으로 실행된다.
	- `AllInterceptor` 는 모든 URL 에서 실행된다.
	- `AInterceptor` 는 `/a` 의 하위 URL 에서 실행된다.
	
### Util

```java
public class Util {
    public static String getMethodName(Throwable throwable) {
        return throwable.getStackTrace()[0].getMethodName();
    }

    public static String getClassName(Throwable throwable) {
        return throwable.getStackTrace()[0].getClassName();
    }

    public static String getSignature(Throwable throwable) {
        return getClassName(throwable) + "." + getMethodName(throwable);
    }

    public static String getSignature(Throwable throwable, String info) {
        return getSignature(throwable) + ":" + info;
    }
}
```  

- `getMethodName()`, `getClassName()` 은 인자값의 `Throwable` 을 사용해서 호출한 곳의 클래스와 메소드이름을 리턴한다.
- `getSignature()` 는 클래스와 메소드 이름을 통해 고유한 정보를 만들어 리턴한다.
	- `a.b.c.D` 클래스에서 `method()` 메소드가 `getSignature()` 를 호출하면 리턴값은 `a.b.c.D.method` 가 된다.
	

### CustomException 

```java
public class CustomException extends Exception {

}
```  

- `CustomException` 는 `Controller` 에서 발생하는 예외를 의미한다.

	
### Controller

```java
@RestController
public class AllController {
    @PostMapping
    public List<String> post(@RequestBody List<String> list) throws Exception{
        list.add("response");
        return list;
    }
    
    @PostMapping("/exception")
    public List<String> exception(@RequestBody List<String> list) throws Exception{
        list.add("response");
        throw new CustomException();
    }
}
```  

```java
@RestController
@RequestMapping("/a")
public class AController {
    @PostMapping
    public List<String> post(@RequestBody List<String> list) throws Exception{
        list.add("response");
        return list;
    }
    
    @PostMapping("/exception")
    public List<String> exception(@RequestBody List<String> list) throws Exception{
        list.add("response");
        throw new CustomException();
    }
}
```  

- `Controller` 에서는 요청값 `list` 에 `response` 라는 문자열을 넣고 다시 응답 값으로 리턴한다.

### Wrapper

```java
public class ModifyRequestWrapper extends HttpServletRequestWrapper {
    private ObjectMapper objectMapper;
    private byte[] buffer;
    private ByteArrayInputStream bufferStream;

    public ModifyRequestWrapper(HttpServletRequest request) {
        super(request);
        this.objectMapper = new ObjectMapper();

        try {
            this.buffer = StreamUtils.copyToByteArray(request.getInputStream());
            this.bufferStream = new ByteArrayInputStream(this.buffer);
        } catch(Exception e) {
            throw new RuntimeException();
        }
    }

    @Override
    public ServletInputStream getInputStream() {
        return new CustomServletInputStream(this.bufferStream);
    }

    public byte[] getBodyBytes() {
        return this.buffer;
    }

    public String getBodyString() {
        return new String(this.buffer);
    }

    public void add(String str) throws IOException {
        List r = this.objectMapper.readValue(this.buffer, List.class);
        r.add(str);
        this.write(r);
    }

    public void write(byte[] bytes) {
        this.buffer = bytes;
        this.bufferStream = new ByteArrayInputStream(this.buffer);
    }

    public void write(String str) {
        this.buffer = str.getBytes();
        this.bufferStream = new ByteArrayInputStream(this.buffer);
    }

    public void write(Object obj) throws IOException {
        this.buffer = this.objectMapper.writeValueAsBytes(obj);
        this.bufferStream = new ByteArrayInputStream(this.buffer);
    }
}
```  

- `ModifyRequestWrapper` 는 `HttpServletRequestWrapper` 의 하위 클래스로 `HttpServletRequest` 를 커스텀 하게 사용할 수 있도록 하는 클래스이다.
- 생성자에 `HttpServletRequest` 를 인자 값으로 받아 초기화 한다.
- `ModifyRequestWrapper` 는 요청 값을 임시로 저장해두고 컨트롤하는 역할을 수행한다.
- `add()` 는 예제에서 사용하는 요청 값(List)에 인자값으로 받은 문자열을 추가하는 메소드이다.

```java
public class CustomServletInputStream extends ServletInputStream {
    private ByteArrayInputStream bufferStream;

    public CustomServletInputStream(ByteArrayInputStream bufferStream) {
        this.bufferStream = bufferStream;
    }

    @Override
    public boolean isFinished() {
        return this.bufferStream.available() == 0;
    }

    @Override
    public boolean isReady() {
        return false;
    }

    @Override
    public void setReadListener(ReadListener readListener) {

    }

    @Override
    public int read() throws IOException {
        return this.bufferStream.read();
    }
}
```  

- `CustomServletInputStream` 은 `ModifyRequestWrapper` 에서 사용하는 `ServletInputStream` 의 하위 클래스이다.
- 생성자에서 `ModifyRequestWrapper` 로 부터 받은 `ByteArrayInputStream` 을 필드에 설정한다.
- `CustomServletInputStream` 의 사용자는 `ModifyRequestWrapper` 의 `ByteArrayInputStream` 을 사용하도록 하는 역할을 수행한다.

```java
public class ModifyResponseWrapper extends HttpServletResponseWrapper {
    private ByteArrayOutputStream buffer;
    private PrintWriter writer;
    private CustomServletOutputStream output;
    private ObjectMapper objectMapper;

    public ModifyResponseWrapper(HttpServletResponse response) {
        super(response);
        this.objectMapper = new ObjectMapper();
        this.buffer = new ByteArrayOutputStream();
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        this.output = new CustomServletOutputStream(this.buffer);
        return this.output;
    }

    @Override
    public PrintWriter getWriter() throws IOException {
        this.writer = new PrintWriter(new OutputStreamWriter(this.buffer, Charset.forName("utf-8")));
        return this.writer;
    }

    @Override
    public void flushBuffer() throws IOException {
        super.flushBuffer();

        if(this.writer != null) {
            this.writer.flush();
        } else if(this.output != null) {
            this.output.flush();
        }
    }

    public byte[] getBodyBytes() {
        if(this.writer != null) {
            this.writer.flush();
        }
        return this.buffer.toByteArray();
    }

    public String getBodyString() {
        if(this.writer != null) {
            this.writer.flush();
        }
        return this.buffer.toString();
    }

    public void add(String str) throws IOException{
        List list = this.objectMapper.readValue(this.getBodyString(), List.class);
        list.add(str);
        this.write(list);
    }

    public void write(byte[] bytes) throws IOException {
        this.buffer.reset();
        this.buffer.write(bytes);

    }

    public void write(String str) throws IOException {
        this.buffer.reset();
        this.buffer.write(str.getBytes());
    }

    public void write(Object obj) throws IOException {
        this.buffer.reset();
        this.buffer.write(this.objectMapper.writeValueAsBytes(obj));
    }
}
```  

- `ModifyResponseWrapper` 는 `HttpServletResponseWrapper` 의 하위 클래스로 `HttpServletResponse` 를 커스텀하게 사용할 수 있도록하는 클래스이다.
- 생성자에서 `HttpServletResponse` 를 받아 초기화 한다.
- `ModifyResponseWrapper` 는 응답 값을 임시로 저장해두고 컨트롤하는 역할을 수행한다.
- `add()` 는 예제에서 사용하는 응답 값(List)에 인자값으로 받은 문자열을 추가하는 메소드이다.

```java
public class CustomServletOutputStream extends ServletOutputStream {
    private ByteArrayOutputStream buffer;

    public CustomServletOutputStream(ByteArrayOutputStream buffer) {
        this.buffer = buffer;
    }

    @Override
    public boolean isReady() {
        return false;
    }

    @Override
    public void setWriteListener(WriteListener writeListener) {

    }

    @Override
    public void close() throws IOException {
        this.buffer.close();
    }

    @Override
    public void flush() throws IOException {
        this.buffer.flush();
    }

    @Override
    public void write(int b) throws IOException {
        this.buffer.write(b);
    }

    @Override
    public void write(byte[] b) throws IOException {
        this.buffer.write(b);
    }

    @Override
    public void write(byte[] b, int off, int len) throws IOException {
        this.buffer.write(b, off, len);
    }
}
```  

- `CustomServletOutputStream` 은 `ModifyResponseWrapper` 에서 사용하는 `ServletOutputStream` 의 하위 클래스이다.
- 생성자에서 `ModifyResponseWrapper` 로 부터 받은 `ByteArrayOutputStream` 을 필드에 설정한다.
- `CustomServletOutputStream` 의 사용자는 `ModifyResponseWrapper` 의 `ByteArrayOutputStream` 을 사용하도록 하는 역할을 수행한다.

### Filter

```java
public class AFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        ModifyRequestWrapper modifyRequestWrapper = new ModifyRequestWrapper(request);
        ModifyResponseWrapper modifyResponseWrapper = new ModifyResponseWrapper(response);

        modifyRequestWrapper.add(Util.getSignature(new Throwable(), "before"));
        filterChain.doFilter(modifyRequestWrapper, modifyResponseWrapper);
        modifyResponseWrapper.add(Util.getSignature(new Throwable(), "after"));

        servletResponse.getOutputStream().write(modifyResponseWrapper.getBodyBytes());
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void destroy() {
    }
}
```  

```java
public class AllFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        ModifyRequestWrapper modifyRequestWrapper = new ModifyRequestWrapper(request);
        ModifyResponseWrapper modifyResponseWrapper = new ModifyResponseWrapper(response);

        modifyRequestWrapper.add(Util.getSignature(new Throwable(), "before"));
        filterChain.doFilter(modifyRequestWrapper, modifyResponseWrapper);
        modifyResponseWrapper.add(Util.getSignature(new Throwable(), "after"));

        servletResponse.getOutputStream().write(modifyResponseWrapper.getBodyBytes());
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void destroy() {
    }
}
```  

- `Filter` 에서는 `doFilter()` 의 인자값의 `servletRequest`, `servletResponse` 를 `ModifyRequestWrapper`, `ModifyResponseWrapper` 의 인스턴스로 만들어준다.
- 예제 테스트를 위해 `filterChain.doFilter()` 수행 전에는 `ModifyRequestWrapper` 의 `add()` 메소드를 사용해서 현재 클래스, 메소드 정보를 요청 값에 추가한다.
- `filterChain.doFilter()` 수행 후에는 `ModifyResponseWrapper` 의 `add()` 메소드를 사용해서 현재 클래스 메소드 정보를 응답 값에 추가한다.
- `doFilter()` 마지막 부분에서는 인자값의 `servletResponse` 의 `getOutputStream()` 을 사용해서 `ModifyResponseWrapper` 의 응답 값을 전달해줘야 실제 응답 값이 제대로 전달된다.


### Interceptor 

```java
@Component
public class AllInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        ModifyRequestWrapper modifyRequestWrapper = (ModifyRequestWrapper) request;
        modifyRequestWrapper.add(Util.getSignature(new Throwable()));
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        ModifyResponseWrapper modifyResponseWrapper = (ModifyResponseWrapper) response;
        modifyResponseWrapper.add(Util.getSignature(new Throwable()));
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        ModifyResponseWrapper modifyResponseWrapper = (ModifyResponseWrapper) response;
        modifyResponseWrapper.add(Util.getSignature(new Throwable()));
    }
}
```  

```java
@Component
public class AInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        ModifyRequestWrapper modifyRequestWrapper = (ModifyRequestWrapper) request;
        modifyRequestWrapper.add(Util.getSignature(new Throwable()));
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        ModifyResponseWrapper modifyResponseWrapper = (ModifyResponseWrapper) response;
        modifyResponseWrapper.add(Util.getSignature(new Throwable()));
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        ModifyResponseWrapper modifyResponseWrapper = (ModifyResponseWrapper) response;
        modifyResponseWrapper.add(Util.getSignature(new Throwable()));
    }
}
```  

- `Filter` 의 `doFilter()` 메소드를 통해 전달해준 값은 `ModifyRequestWrapper`, `ModifyResponseWrapper` 의 인스턴스이다.
- `Interceptor` 의 인자에서 형식은 `HttpServletRequest`, `HttpServletResponse` 이지만 이를 `Wrapper` 클래스 타입으로 변환해서 사용할 수 있다.
- `Filter` 와 동일하게 요청/응답 값을 컨트롤 할 수 있다.
- `preHandler()` 메소드에서는 요청 전에 현재 클래스 메소드의 정보를 요청 값에 추가한다.
- `postHandler()` 메소드에서는 응답 후에 현재 클래스 메소드의 정보를 응답 값에 추가한다.

### ExceptionAdvice

```java
@ControllerAdvice
public class ExceptionAdvice {
    @ExceptionHandler(CustomException.class)
    public ResponseEntity<?> customException(CustomException customException) {
        List<String> list  = new LinkedList<>(Arrays.asList(Util.getSignature(new Throwable())));
        return new ResponseEntity<>(list, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```  

- `ExceptionAdvice` 는 `@ControllerAdvice` 가 선언된, `Controller` 에서 발생하는 예외를 처리하는 클래스이다.
- `customException()` 은 `CustomException` 예외를 처리하는 메소드로, 응답 값에 현재 클래스 메소드의 정보를 넣어 응답한다.

### 테스트

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class AllControllerTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void post_Response() {
        // given
        List<String> list = new LinkedList<>();
        list.add("request");

        // when
        List<String> actual = this.restTemplate.postForObject("/", list, List.class);

        // then
        System.out.println(Arrays.toString(actual.toArray()));
        assertThat(actual, contains(
                // Client
                "request",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                // Controller
                "response",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }

    @Test
    public void exception_Response() {
        // given
        List<String> list = new LinkedList<>();
        list.add("request");

        // when
        List<String> actual = this.restTemplate.postForObject("/exception", list, List.class);

        // then
        System.out.println(Arrays.toString(actual.toArray()));
        assertThat(actual, contains(
                // ControllerAdvice
                "com.windowforsun.exam.advice.ExceptionAdvice.customException",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class AControllerTest {
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    public void post_Response() {
        // given
        List<String> list = new LinkedList<>();
        list.add("request");

        // when
        List<String> actual = this.restTemplate.postForObject("/a", list, List.class);

        // then
        System.out.println(Arrays.toString(actual.toArray()));
        assertThat(actual, contains(
                // Client
                "request",
                // Filter
                "com.windowforsun.exam.filter.AllFilter.doFilter:before",
                "com.windowforsun.exam.filter.AFilter.doFilter:before",
                // Interceptor
                "com.windowforsun.exam.interceptor.AllInterceptor.preHandle",
                "com.windowforsun.exam.interceptor.AInterceptor.preHandle",
                // Controller
                "response",
                // Interceptor
                "com.windowforsun.exam.interceptor.AInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AllInterceptor.postHandle",
                "com.windowforsun.exam.interceptor.AInterceptor.afterCompletion",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AFilter.doFilter:after",
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }

    @Test
    public void exception_Response() {
        // given
        List<String> list = new LinkedList<>();
        list.add("request");

        // when
        List<String> actual = this.restTemplate.postForObject("/a/exception", list, List.class);

        // then
        System.out.println(Arrays.toString(actual.toArray()));
        assertThat(actual, contains(
                // ControllerAdvice
                "com.windowforsun.exam.advice.ExceptionAdvice.customException",
                // Interceptor
                "com.windowforsun.exam.interceptor.AInterceptor.afterCompletion",
                "com.windowforsun.exam.interceptor.AllInterceptor.afterCompletion",
                // Filter
                "com.windowforsun.exam.filter.AFilter.doFilter:after",
                "com.windowforsun.exam.filter.AllFilter.doFilter:after"
        ));
    }
}
```  

- 테스트 결과는 요청 경로(`Filter`, `Interceptor`), 예외 발생 여부에 따라 달라진다.
- 예외가 발생하지 않은 경우 결과는 `Client -> Filter -> Interceptor -> Controller -> Interceptor -> Filter` 순으로 실행된다.
- 예외가발생한 경우 결과는 `Client -> Filter -> Interceptor -> Controller -> ControllerAdvice -> Interceptor -> Filter` 순으로 실행된다.
	- 응답은 `ControllerAdvice` 부터 `Filter` 까지의 값이다.

---
## Reference
[how modify the response body with java filter?](https://stackoverflow.com/questions/51826475/how-modify-the-response-body-with-java-filter)  
[How to modify HTTP response using Java Filter](https://www.codejava.net/java-ee/servlet/how-to-modify-http-response-using-java-filter)  
[[ JSP ] Servlet Filter modify request body](https://https://shonm.tistory.com/549.com/questions/51826475/how-modify-the-response-body-with-java-filter)  
[[Servlet] 서블릿 필터](https://devbox.tistory.com/entry/Servlet-%EC%84%9C%EB%B8%94%EB%A6%BF-%ED%95%84%ED%84%B0%EC%99%80-%EC%9D%B4%EB%B2%A4%ED%8A%B8-1)  
