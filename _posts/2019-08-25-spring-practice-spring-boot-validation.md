--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Validation"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Validation 을 통해 효율적인 유효성 검사를 수행하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Validation
---  

# 목표
- Spring Validation 에 대해 알아본다.
- Spring Validation Annotation 을 이용해서 유효성 검사를 수행한다.
- Custom Validation 을 만들어 유효성 검사를 추가한다.
- 유효성 검사를 실패했을 때, 에러 메시지를 응답해 준다.

# 방법
## 유효성 검사란
- API 요청의 요소들이 특정한 데이터 타입과, 특정 도메인의 제약사항을 만족하는지 검사한다.
- 제약 사항을 만족하지 않을 경우, 아래와 같은 Error 를 응답한다.
	- 유효성 검사의 Error 를 나타내는 메시지, 어느 Filed 인지, 올바른 형태는 무엇인지
	- HTTP 상태 코드
	- Error 응답에 중요한 정보를 포함해서는 안된다.
- 유효성 검사 에러에 권장하는 HTTP 상태코드는 400(Bad Request) 이다.

## Validation Annotation
- JSR-303 에서 기본적으로 아래와 같은 Annotation 을 통해 유효성 검사를 수행할 수 있다.
	- @AssertFalse : false 인지검사
	- @AssertTrue : true 인지검사
	- @DecimalMax : 지정된 값 이하 실수인지 검사
	- @DecimalMin : 지정된 값 이상 실수인지 검사
	- @Digit(integer=1, fraction=0.3) : 지정된 정수, 소수자리수 이내의 값인지 검사
	- @Future : 현재 시간보다 미래의 날짜인지 검사
	- @Past : 현재 시간보다 과거의 날짜인지 검사
	- @Max : 지정된 값 이하인지 검사
	- @Min : 지정된 값 이상인지 검사
	- @NotNull : Null 이 아닌지 검사
	- @Null : Null 값인지 검사
	- @Pattern(regex="[0-9]+",flag=) : 지정된 정규식을 만족하는지 검사
	- @Size(min=1, max=10) : 문자열, 배열 등의 크키가 주어진 값을 만족하는지 검사
	
- Hibernate 에서 기본적으로 아래와 같은 Annotation 을 제공한다.
	- @NotEmpty : 빈 값인지 검사
	- @URL : URL 형식인지 검사
	- @Range(min=1, max=10) : 지정된 범위를 만족하는 숫자인지 검사
	- @Email : Email 형식인지 검사
	- @Length(min=1, max=10) : 문자열 길이가 주어진 범위를 만족하는지 검사

### @Valid
- Bean Validation 으로 `Valid` Annotation 이 선언된 Bean 의 유효성 검사를 수행한다.

```java
public void method(@Valid MyDomain myDomain) {
	// ...
}
```  

- 위 코드에서 MyDomain 클래스의 필드에 선언된 Validation Annotation 에 맞게 유효성 검사를 수행한다.

### @Validated
- Validation Groups 을 위해 사용하기 위해 추가된 Validation Annotation 이다.

	```java
	public class MyDomain {
		public interface ValidationStepOne {}
		public interface ValidationStepTwo {}
		
		@NotNull(groups = {ValidationStepOne.class})
		@NotBlank(groups = {ValidationStepTwo.class})
		private String name;
		@Min(value = 1, groups = {ValidationStepOne.class})
		@Max(value = 100, groups = {ValidationStepOne.class})
		private int age;
	}
	
	public class MyService {
		@Validated(MyDomain.ValidationStepOne.class)
		public void checkStepOne(@Valid MyDomain myDomain) {
			// MyDomain 에 선언된 ValidationStepOne 그룹에 대한 유효성 검사
		}
		
		@Validated(MyDomain.ValidationStepTow.class)
		public void checkStepTwo(@Valid MyDomain myDomain) {
			// MyDomain 에 선언된 ValidationStepTwo 그룹에 대한 유효성 검사
		}
	}
	```  

- 클래스, 메서드에 선언된 변수들에 유효성 검사를 수행 할때도 사용할 수 있다.

	```java
	@Validated
	public class MyClass {
		public void method1(@NotNull String str) {
			// ....
		}
		
		public void method2(@NotEmpty String str) {
			// ....
		}
	}
	```  

	- 위 코드에서 `method1`, `method2` 에는 `Valid` Annotation 이 선언되지 않았지만 Class 레벨에서 선언된 `Validated` Annotation 이 있기 때문에 유효성 검사를 수행한다.
		- `MyDomain` 와 같은 클래스에 선언되어 있는 객체 변수에 대한 유효성검사를 수행할때는 `Valid` Annotation 을 선언해 주어야 한다.

## 의존성
- `spring-boot-starter-web` 의존성이 포함되어 있다면, `org.hibernate.validator` 가 자식으로 포함되기 때문에 바로 사용가능하다.
- 아래와 같이 Validation 관련 의존성을 추가할 수 있다.

	```java
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-validation</artifactId>
	</dependency>		
	```  
	
	```java
	<dependency> 
		<groupId>org.hibernate</groupId> 
		<artifactId>hibernate-validator</artifactId> 
		<version>4.3.2.Final</version> 
	</dependency>
	```

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-spring-boot-validation-1.png)

## pom.xml

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-redis</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-validation</artifactId>
	</dependency>

	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<optional>true</optional>
	</dependency>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<version>RELEASE</version>
	</dependency>
	<!-- Embedded Redis -->
	<dependency>
		<groupId>it.ozimov</groupId>
		<artifactId>embedded-redis</artifactId>
		<version>0.7.2</version>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```  

## Config
- EmbeddedRedisConfig

	```java
	@Configuration
    public class EmbeddedRedisConfig {
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
    }
	```  
	
## Application.properties

```
# Redis 포트 설정
spring.redis.port=60005
```  

## Domain

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
@Builder
@RedisHash("Account")
public class Account implements Serializable {
    @Id
    @Min(value = 1, message = "min 1")
    private long id;
    @NumberString
    private String numberString;
    @NotEmpty(message = "name is mandatory")
    private String name;
    @Min(groups = {ValidationStepOne.class}, value = 1, message = "min 1")
    @Max(groups = {ValidationStepTwo.class}, value = 100, message = "max 1")
    private int age;

    public interface ValidationStepOne { }
    public interface ValidationStepTwo { }
}
```  

- `Account` 도메인에는 `id`, `numberString`, `name`, `age` 필드를 가지고 있다.
	- `id` : Redis 에서 사용되는 아이디 필드로 `@Min` 을 통해 최소 1 값을 가져야하는 제약조건 설정
	- `numberString` : 숫자로 구성된 문자열 값을 가지는 필드로, `@NumberString` Custom Validation 을 통해 숫자로 구성된 문자열을 가지는 제약조건 설정
	- `name` : 이름을 저장하는 필드로 `@NotEmpty` 를 통해 null 값이나, 빈문자열을 가지지 않는 제약조건 설정
	- `age`: 나이를 저장하는 필드로 `ValidationStepOne` 에서는 `@Min` 을 통해 최소 1값을 가져야 하는 제약조건 설정했고, `ValidationStepTwo` 에서는 `@Max` 를 통해 최대 100 갑을 가지도록 제약조건 설정

## Repository

```java
@Repository
public interface AccountRepository extends CrudRepository<Account, Long> {
}
```  

- `AccountRepository` 인터페이스를 통해 Redis 에 `Account` 도메이 값을 저장한다.

## Service

```java
@Service
@Validated
public class AccountService {
    private AccountRepository accountRepository;

    public Account readById(@ExistId @Min(value = 1) long id) {
        return this.accountRepository.findById(id).orElse(null);
    }

    public Account save(@Valid Account account) {
        return this.accountRepository.save(account);
    }

    public Account update(@ExistId long id, @Valid Account account) {
        account.setId(id);
        return this.accountRepository.save(account);
    }

    public void deleteById(@ExistId long id) {
        this.accountRepository.deleteById(id);
    }

    @Validated(Account.ValidationStepOne.class)
    public Account checkStepOne(@Valid Account account) {
        return account;
    }

    @Validated(Account.ValidationStepTwo.class)
    public Account checkStepTwo(@Valid Account account) {
        return account;
    }

    @NotNull
    public Account checkNotNull(Account account) {
        return account;
    }

    @Autowired
    public void setAccountRepository(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }
}
```  

- `readById`, `update`, `deleteById` 메서드에는 `@ExistId` 라는 Custom Validation 이 선언되어 있어, 현재 Redis 저장소에 존재하는 아이디 값만 가질 수 있다.
- `checkStepOne`, `checkStepTwo` 메서드는 `Account` 클래스 필드에서 다르 필드의 제약조건은 모두 같지만 `age` 필드에 대한 Validation Group 에 따라 유효성 검사를 수행한다.
- `checkNotNull` 메서드는 리턴 값에 대한 유효성 검사를 수행하므로, 해당 메서드는 null 값을 리턴할 수 없다.

## Message
- GetAccountId

	```java
	@Getter
    @Setter
    @NoArgsConstructor
    public class GetAccountById {
        @ExistId
        private long id;
    }
	```  
	
- UpdateAccountById

	```java
	@Getter
    @Setter
    @NoArgsConstructor
    public class UpdateAccountById {
        @ExistId
        private long id;
    }
	```  
	
- DeleteAccountById

	```java
	@Getter
    @Setter
    @NoArgsConstructor
    public class DeleteAccountById {
        @ExistId
        private long id;
    }
	```  
	
- 위 3개의 Message 클래스는 Controller 의 요청파라미터에 대한 정의부분으로 Get, Update, Delete 시에 요청하는 아이디는 Redis 에 존재하지 않는 아이디 값을 가질 수 없다.
- ErrorDetail

	```java
	@Getter
    @Setter
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @ToString
    public class ErrorDetail {
        private long timestamp;
        private String message;
        private String detail;
    }
	```  
	
	- 유효성 검사에 실패 했을 때 Custom Response Message 를 정의한 클래스이다.

## Controller

```java
@RestController
@RequestMapping("/account")
public class AccountController {
    private AccountService accountService;

    @GetMapping(value = "/{id}")
    private Account readById(@Valid GetAccountById getAccountById) {
        return this.accountService.readById(getAccountById.getId());
    }

    @PostMapping
    public Account create(@Valid @RequestBody Account account) {
        return this.accountService.save(account);
    }

    @PutMapping(value = "/{id}")
    private Account updateById(@Valid UpdateAccountById updateAccountById, @Valid @RequestBody Account account) {
        return this.accountService.update(updateAccountById.getId(), account);
    }

    @DeleteMapping(value = "/{id}")
    private void deleteById(@Valid DeleteAccountById deleteAccountById) {
        this.accountService.deleteById(deleteAccountById.getId());
    }

    @Autowired
    public void setAccountService(AccountService accountService) {
        this.accountService = accountService;
    }
}
```  

- `AccountController` 에서는 요청을 받게 되면 선언된 Validation Annotation 에 따라 요청 값에 대한 검증을 수행한다.
- `Get`, `Put`, `Delete` 메서드에서 URL Path(PathVariable) 에 있는 id 값에 대한 유효성 검사를 위해 별도의 클래스를 선언해서 사용했다.
	- 다른 방법으로는 `Validated` Annotation 을 이용하는 방법이 있다.

## Custom Validation
- 유효성 검사 Annotation 을 사용자가 정의하기 위해서는 먼저 Custom Annotation 을 만들어 준다.

	```java
	@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = ExistIdValidator.class)
    public @interface ExistId {
        String message() default "Not Exist Id";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }
	```  
	
	```java
	@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = NumberStringValidator.class)
    public @interface NumberString {
        String message() default "String must be numberic";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }
	```  
	
- `ExistId` 는 존재하는 아이디인지에 대한 유효성 검사 Annotation 이고, `NumberString` 은 문자열로 구성된 숫자인지에 대한 유효성 검사이다.
- 두 Custom Annotation 은 `@Constraint` Annotation 에서 각 Annotation 의 제약조건을 검사할 클래스를 인자 값을 가지고 있다.
	- `ExistIdValidator` : 존재하는 아이디인지 검사
	- `NumberStringValidator` : 숫자로 구성된 문자열인지 검사
- 제약조건을 검사하는 클래스는 `ConstraintValidator` 인터페이스를 상속해서 구현한다.

	```java
	public class ExistIdValidator implements ConstraintValidator<ExistId, Long> {
        protected AccountRepository accountRepository;
    
        public ExistIdValidator(AccountRepository accountRepository) {
            this.accountRepository = accountRepository;
        }
    
        @Override
        public boolean isValid(Long l, ConstraintValidatorContext constraintValidatorContext) {
            return this.accountRepository.existsById(l);
        }
    
        @Override
        public void initialize(ExistId constraintAnnotation) {
        }
    }
	```  
	
	- 아이디 검사를 위해서는 Redis Repository 에 접근이 필요한데 이 객체에 대한 주입은 생성자를 통해 가능하다.
	
	```java
	public class NumberStringValidator implements ConstraintValidator<NumberString, String> {
        @Override
        public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
            return s.matches("[0-9]+");
        }
    
        @Override
        public void initialize(NumberString constraintAnnotation) {
        }
    }
	```  
	
## Custom Validation Response
- Validation Annotation 을 이용해서 유효성 검사가 실패하면 `MethodArgumentNotValidException` 과 `BindException` 이 발생한다.
- 유효성 검사 실패에 대해 에러 메시지 Custom 은 `ControllerAdvice` 를 통해 가능하다.

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    @ResponseBody
    public ResponseEntity<?> globalExceptionHandler(Exception exception, WebRequest webRequest) {
        ErrorDetail errorDetail = ErrorDetail.builder()
                .timestamp(System.currentTimeMillis())
                .message(exception.getMessage())
                .detail(webRequest.getDescription(false))
                .build();

        return new ResponseEntity<>(errorDetail, HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseBody
    public ResponseEntity<?> methodArgumentNotValidExceptionHandler(MethodArgumentNotValidException exception) {
        ErrorDetail errorDetail = ErrorDetail.builder()
                .timestamp(System.currentTimeMillis())
                .message("Validation Failed")
                .detail(exception.getBindingResult().toString())
                .build();

        return new ResponseEntity<>(errorDetail, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(BindException.class)
    @ResponseBody
    public ResponseEntity<?> bindExceptionHandler(BindException exception) {
        ErrorDetail errorDetail = ErrorDetail.builder()
                .timestamp(System.currentTimeMillis())
                .message("Bind Failed")
                .detail(exception.getBindingResult().toString())
                .build();

        return new ResponseEntity<>(errorDetail, HttpStatus.BAD_REQUEST);
    }
}
```  

## Test
### Service Test

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class AccountServiceTest {
    @Autowired
    private AccountRepository accountRepository;
    @Autowired
    private AccountService accountService;

    @Before
    public void setUp() {
        this.accountRepository.deleteAll();
    }

    @Test
    public void readById_ExistId_Expected_ReturnAccount() {
        // given
        long id = 1;
        this.accountRepository.save(Account.builder()
                .id(id)
                .build());

        // when
        Account actual = this.accountService.readById(id);

        // when
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(id));
    }

    @Test(expected = ConstraintViolationException.class)
    public void readById_NotExistId_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11111;

        // when
        Account account = this.accountService.readById(id);
    }

    @Test(expected = ConstraintViolationException.class)
    public void readById_ZeroId_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 0;

        // when
        Account account = this.accountService.readById(id);
    }

    @Test
    public void save_ValidAccount_Expected_ReturnAccount() {
        // given
        Account expected = Account.builder()
                .id(1l)
                .age(28)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.save(expected);

        // then
        assertThat(actual, equalTo(expected));
    }

    @Test(expected = ConstraintViolationException.class)
    public void save_AccountZeroAge_Expected_ThrowsConstraintViolationException() {
        // given
        Account expected = Account.builder()
                .id(1l)
                .age(0)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.save(expected);
    }

    @Test
    public void update_ValidIdAccount_Expected_ReturnAccount() {
        // given
        long id = 1111;
        Account expected = Account.builder()
                .id(id)
                .build();
        this.accountRepository.save(expected);
        expected.setNumberString("123");
        expected.setAge(24);
        expected.setName("name");

        // when
        Account actual = this.accountService.update(id, expected);

        // when
        assertThat(actual, equalTo(expected));
        assertThat(actual, equalTo(this.accountRepository.findById(id).orElse(null)));
    }

    @Test(expected = ConstraintViolationException.class)
    public void update_NotExistId_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.update(id, expected);
    }

    @Test(expected = ConstraintViolationException.class)
    public void update_ZeroAgeAccount_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();
        this.accountRepository.save(expected);
        expected.setAge(0);

        // when
        Account actual = this.accountService.update(id, expected);
    }

    @Test(expected = ConstraintViolationException.class)
    public void update_NullNameAccount_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();
        this.accountRepository.save(expected);
        expected.setName(null);

        // when
        Account actual = this.accountService.update(id, expected);
    }

    @Test(expected = ConstraintViolationException.class)
    public void update_NotNumberStringAccount_ThrowsConstraintViolationException() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();
        this.accountRepository.save(expected);
        expected.setNumberString("aaaww123");

        // when
        Account actual = this.accountService.update(id, expected);
    }

    @Test
    public void deleteById_ExistId_Expected_DeleteAccount() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();
        this.accountRepository.save(expected);

        // when
        this.accountService.deleteById(id);

        // then
        assertThat(this.accountRepository.existsById(id), is(false));
    }

    @Test(expected = ConstraintViolationException.class)
    public void deleteById_NotExistId_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11;

        // when
        this.accountService.deleteById(id);
    }

    @Test(expected = ConstraintViolationException.class)
    public void update_ReturnNull_Expected_ThrowsConstraintViolationException() {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("123")
                .build();
        this.accountRepository.save(expected);

        // when
        Account actual = this.accountService.update(id, expected);
    }

    @Test
    public void checkStepOne_Valid_Expected_ReturnAccount() {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(100)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.checkStepOne(expected);

        // then
        assertThat(actual, equalTo(expected));
    }

    @Test(expected = ConstraintViolationException.class)
    public void checkStepOne_Invalid_Expected_ThrowsConstraintViolationException() {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(0)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.checkStepOne(expected);
    }

    @Test
    public void checkStepTwo_Valid_Expected_ReturnAccount() {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(0)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.checkStepTwo(expected);

        // then
        assertThat(actual, equalTo(expected));
    }

    @Test(expected = ConstraintViolationException.class)
    public void checkStepTwo_Invalid_Expected_ThrowsConstraintViolationException() {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(101)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.checkStepOne(expected);
    }

    @Test
    public void checkNotNull_NotNull_Expected_ReturnAccount() {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(1)
                .name("name")
                .numberString("123")
                .build();

        // when
        Account actual = this.accountService.checkNotNull(expected);

        // then
        assertThat(actual, equalTo(expected));
    }

    @Test(expected = ConstraintViolationException.class)
    public void checkNotNull_Null_Expected_ThrowsConstraintViolationException() {
        // given
        Account expected = null;

        // when
        Account actual = this.accountService.checkNotNull(expected);
    }
}
```  

### Controller Test

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
@Validated
public class AccountControllerTest {
    @Autowired
    private MockMvc mockMvc;
    @Autowired
    private AccountRepository accountRepository;
    @Autowired
    private AccountService accountService;
    private ObjectMapper objectMapper = new ObjectMapper();
    private MediaType contentType = new MediaType(MediaType.APPLICATION_JSON.getType(),
            MediaType.APPLICATION_JSON.getSubtype(),
            Charset.forName("utf8"));

    @Before
    public void setUp() {
        this.accountRepository.deleteAll();
    }

    @Test
    public void create_Valid_Expected_ResponseEchoObj() throws Exception {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(1)
                .name("a")
                .numberString("123")
                .build();

        // when
        MvcResult result = this.mockMvc
                .perform(post("/account").
                        contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isOk())
                .andReturn();

        // then
        Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
        assertThat(actual, equalTo(expected));
    }

    @Test
    public void create_ZeroAge_Expected_BadRequest() throws Exception {
        // given
        Account expected = Account.builder()
                .id(1)
                .age(0)
                .name("a")
                .numberString("123")
                .build();

        // when
        MvcResult result = this.mockMvc
                .perform(post("/account")
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThan(1l), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void create_NullName_Expected_BadRequest() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        Account expected = Account.builder()
                .id(1)
                .age(1)
                .name("")
                .numberString("123")
                .build();

        // when
        MvcResult result = this.mockMvc
                .perform(post("/account").
                        contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void create_NotNumberString_Expected_BadRequest() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        Account expected = Account.builder()
                .id(1)
                .age(1)
                .name("a")
                .numberString("1111d1123")
                .build();

        // when
        MvcResult result = this.mockMvc
                .perform(post("/account").
                        contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void readById_ExistId_Expected_ResponseAccountIdIs1() throws Exception {
        // given
        long id = 1;
        this.accountRepository.save(Account.builder()
                .id(id)
                .build());

        // when
        MvcResult result = this.mockMvc
                .perform(get("/account/" + id))
                .andExpect(status().isOk())
                .andReturn();

        // then
        Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
        assertThat(actual.getId(), is(id));
    }

    @Test
    public void readById_NotExistId_Expected_BadRequest() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 111;

        // when
        MvcResult result = this.mockMvc
                .perform(get("/account/" + id))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Bind Failed"));
    }

    @Test
    public void updateById_ValidIdAccount_Expected_ResponseAccount() throws Exception {
        // given
        long id = 1223;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();
        this.accountService.save(expected);

        // when
        MvcResult result = this.mockMvc
                .perform(put("/account/" + id)
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isOk())
                .andReturn();

        // then
        Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
        assertThat(actual, equalTo(expected));
    }

    @Test
    public void updateById_NotExistId_Expected_StatusBadRequest_BindFail() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();

        // when
        MvcResult result = this.mockMvc
                .perform(put("/account/" + id)
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // when
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Bind Failed"));
    }

    @Test
    public void updateById_NullNameAccount_Expected_StatusBadRequest_ValidationFail() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();
        this.accountService.save(expected);
        expected.setName(null);

        // when
        MvcResult result = this.mockMvc
                .perform(put("/account/" + id)
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void updateById_ZeroAgeAccount_Expected_StatusBadRequest_ValidationFail() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();
        this.accountService.save(expected);
        expected.setAge(0);

        // when
        MvcResult result = this.mockMvc
                .perform(put("/account/" + id)
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void updateById_NotNumberStringAccount_Expected_StatusBadRequest_ValidationFail() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();
        this.accountService.save(expected);
        expected.setNumberString("1abc23");

        // when
        MvcResult result = this.mockMvc
                .perform(put("/account/" + id)
                        .contentType(this.contentType)
                        .content(this.objectMapper.writeValueAsString(expected)))
                .andExpect(status().isBadRequest())
                .andReturn();

        // then
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Validation Failed"));
    }

    @Test
    public void deleteById_ExistId_Expected_StatusOk() throws Exception {
        // given
        long id = 11;
        Account expected = Account.builder()
                .id(id)
                .name("name")
                .age(25)
                .numberString("1")
                .build();
        this.accountService.save(expected);

        // when
        MvcResult result = this.mockMvc
                .perform(delete("/account/" + id))
                .andExpect(status().isOk())
                .andReturn();
    }

    @Test
    public void deleteBy_NotExistId_Expected_StatusBadRequest_BindException() throws Exception {
        // given
        long startTime = System.currentTimeMillis();
        long id = 11;

        //  when
        MvcResult result = this.mockMvc
                .perform(delete("/account/" + id))
                .andExpect(status().isBadRequest())
                .andReturn();

        // when
        ErrorDetail errorDetail = this.objectMapper.readValue(result.getResponse().getContentAsString(), ErrorDetail.class);
        assertThat(errorDetail.getTimestamp(), allOf(
                greaterThanOrEqualTo(startTime), lessThanOrEqualTo(System.currentTimeMillis())
        ));
        assertThat(errorDetail.getMessage(), is("Bind Failed"));
    }
}
```  


---
## Reference
[Spring Bean Validation](https://phoenixnap.com/kb/spring-boot-validation-for-rest-services)   
[All You Need To Know About Bean Validation With Spring Boot](https://reflectoring.io/bean-validation-with-spring-boot/)   
[Spring Boot CRUD REST APIs Validation Example](https://www.javaguides.net/2018/09/spring-boot-crud-rest-apis-validation-example.html)   
[Implementing Validation for RESTful Services with Spring Boot](https://www.springboottutorial.com/spring-boot-validation-for-rest-services)   
[Java 와 Spring 의 Validationt](https://medium.com/@gaemi/java-%EC%99%80-spring-%EC%9D%98-validation-b5191a113f5c)   
[Spring REST custom validation](https://medium.com/@amit.dhodi/spring-rest-custom-validation-216e6d33a445)   
[Spring Custom Validator by example](http://dolszewski.com/spring/custom-validation-annotation-in-spring/)   
[Spring validator & Hibernate validator 를 이용한 선언적인 방식 유효성 검증기](https://syaku.tistory.com/346)   
