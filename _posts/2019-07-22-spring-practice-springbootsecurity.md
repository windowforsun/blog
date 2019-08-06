--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Spring Security Form Login"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Spring Security 로 Form 기반 로그인을 수행하고 권한을 관리하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Spring Security
    - Redis
---  

# 목표
- Spring Security 로 권한을 관리한다.
- Spring Security 을 사용해서 Form 기반 로그인을 수행한다.

# 방법
- Redis 를 Spring Security 에서 사용할 계정정보의 저장소로 사용한다.
- 사용자들에게 권한을 부여한다.
- 사용자들의 권한에 따라 접근을 제어한다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-1.png)

## pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
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
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session</artifactId>
        <version>1.3.1.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>it.ozimov</groupId>
        <artifactId>embedded-redis</artifactId>
        <version>0.7.2</version>
    </dependency>
</dependencies>
```  

## Application.properties

```properties
# 세션 시간
server.servlet.session.timeout=30s
# 세션 저장소
spring.session.store-type=redis
spring.redis.port=60555
```  

## Java Config(Redis, Spring Security)
### EmbeddedRedis

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
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        return redisTemplate;
    }
}
```  

### Spring Security

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private AccountService accountService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
        		// csrf 설정은 비활성화
                .csrf().disable()
        // 인증이 필요한 요청 경로
        .authorizeRequests()
                .antMatchers("/user/**").hasRole("USER")
                .antMatchers("/admin/**").hasRole("ADMIN")
                // 이외 모든 요청들은 인증이 필요함
                .anyRequest().authenticated()
        .and()
        		// 로그인 관련 설정
                .formLogin()
                .defaultSuccessUrl("/user")
                .usernameParameter("username")
                .passwordParameter("password")
                .permitAll()
        .and()
        		// 로그아웃 관련 설정
                .logout()
                .logoutSuccessUrl("/login")
                .permitAll();
    }

    // 패스워드 인코딩에서 사용할 방식
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        // 인증, 세션 없이 허용되는 경로
        web.ignoring()
                .antMatchers("/all/**")
                .antMatchers("/user/all")
                .antMatchers("/admin/all");
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
        		// 유저 정보(저장소) 서비스 빈 등록
                .userDetailsService(this.accountService)
                // 패스워드 인코딩 빈 등록
                .passwordEncoder(this.passwordEncoder());
    }
}
```  

- Spring Security 설정을 위해 설정 파일에 `@EnableWebSecurity` Annotation 을 추가한다.
- 메서드에 대한 보안 설정을 위해 `@EnableGlobalMethodSecurity` Annotation 을 설정해 준다.
- 설정 클래스는 `WebSecurityConfigurerAdapter` 를 상속하고 위 클래스에 있는 메서드를 오버라이드해서 간편하게 설정 할 수 있다.
- `configure(HttpSecurity)` 메서드는 인터셉터로 요청을 안전하게 보호하기위한 설정으로, URL 경로의 접근과 권한 설정을 할 수 있다.
	- `authenticationProvider(AuthenticationProvider)` 을 통해 사용자 인증을 커스텀 할 수 있다.
	- `successHandler(SimpleUrlAuthenticationSuccessHandler)` 을 통해 인증 성공시 처리를 커스텀 할 수 있다.
	- `failureHandler(SimpleUrlAuthenticationFailureHandler)` 을 통해 인증 실패시 처리를 커스텀 할 수 있다.
- `configure(WebSecurity)` 메서드에서는 Spring Security 와 필터 연결설정으로, 리소스와 같이 세션정보가 필요없이 접근 가능한 URL 경로에 대한 설정을 할 수 있다.
- `configure(AuthenticationManagerBuilder)` 메서드는 사용자 세부 서비스(저장소)에 대한 설정을 할 수 있다.
- `antMatchers(<URL>)` 메서드는 해당하는 URL 경로에 대한 접근 설정을 하는데, 아래와 같은 항목에 대해 지정 할 수있다.
	- `anonymouse()` : 인증되지 않은 사용자가 접근 가능하다.
	- `authenticated()` : 인증된 사용자만 접근 가능하다.
	- `fullyAuthenticated()` : 완전히 인정된 사용자만 접근 가능하다.
	- `hasRole()`, `hasAnyRole()` : 해당 권한을 가진 사용자만 접근 가능하다.
	- `hasAuthority()`, `hasAnyAuthority()` : 해당 권한을 가진 사용자만 접근 가능하다.
	- `hasIpAddress()` : 해당 아이피 주소를 가진 사용자만 접근 가능하다.
	- `access()` : SpEL 표현식에 맞는 사용자만 접근 가능하다.
	- `not()` : 접근 제한을 해제한다.
	- `permitAll()`, `denyAll()` : 모든 접근을 허용한다, 모든 접근은 제한한다.
	- `rememberMe()` : 리멤버 기능(로그인 정보 유지)을 통해 로그인한 사용자만 접근 가능하다.
- 접근 권한을 설정하는 부분의 `Role` 과 `Authority` 차이는 단순한 표현의 차이로, `Role` 은 `ADMIN` 으로 표기하고, `Authority` 는 `ROLE_ADMIN` 으로 표기한다.


### Etc Config

```java
@Configuration
public class BeanConfig {
    @Bean
    @Autowired
    public String setUpAccount(AccountService accountService) {
        Account account = Account.builder()
                .insertTime(System.currentTimeMillis())
                .username("testAdmin1")
                .password("authToken1")
                .isAccountNonExpired(true)
                .isAccountNonLocked(true)
                .isCredentialsNonExpired(true)
                .isEnabled(true)
                .authorities(Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"), new SimpleGrantedAuthority("ROLE_ADMIN")))
                .build();

        accountService.createAccount(account);

        account = Account.builder()
                .insertTime(System.currentTimeMillis())
                .username("testUser1")
                .password("authToken2")
                .isAccountNonExpired(true)
                .isAccountNonLocked(true)
                .isCredentialsNonExpired(true)
                .isEnabled(true)
                .authorities(Arrays.asList(new SimpleGrantedAuthority("ROLE_USER")))
                .build();

        accountService.createAccount(account);

        return null;
    }
}
```  

- 더미 유저를 등록하는 빈

## 구현 클래스 (Domain, Repository, Service)

- 유저 정보를 저장하는 `Account` 클래스는 Spring Security 의 유저 정보 인터페이스인 `UserDetails` 인터페이스를 구현해야 한다.

	```java
	@Getter
	@Setter
	@AllArgsConstructor
	@NoArgsConstructor
	@ToString
	@RedisHash("Account")
	@EqualsAndHashCode
	@Builder
	public class Account implements UserDetails {
	    @Id
	    private String username;
	    @Indexed
	    private String password;
	    @Indexed
	    private String nickname;
	    private boolean isAccountNonExpired;
	    private boolean isAccountNonLocked;
	    private boolean isCredentialsNonExpired;
	    private boolean isEnabled;
	    private Collection<? extends GrantedAuthority> authorities;
	}
	```  
	
	- `Account` 의 `authorities` 는 해당 유저가 가진을 권한과 관련된 필드이다.
	
- `Account` 의 `Redis` 저장소에 대한 동작을 수행하는 `AccountRepository` 인터페이스 이다.

	```java
	@Repository
    public interface AccountRepository extends CrudRepository<Account, String> {
    }
	```  
	
- `Account` 의 서비스 인터페이스는 아래와 같이 정의해 준다.

	```java
	public interface AccountService extends UserDetailsService {
        public Account readAccount(String username);
        public Account createAccount(Account account);
        public void deleteAccount(String username);
    }
	```  
	
	- `AccountService` 는 `UserDetailService` 를 상속해야 Spring Security 에서 유저 정보를 통한 동작을 수행할 수 있다.

- `AccountServiceImpl` 클래스는 아래와 같다.

	```java
	@Service
    public class AccountServiceImpl implements AccountService {
        @Autowired
        private AccountRepository accountRepository;
        @Autowired
        private PasswordEncoder passwordEncoder;
    
        @Override
        public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
            return this.accountRepository.findById(s).get();
        }
    
        @Override
        public Account readAccount(String username) {
            return this.accountRepository.findById(username).orElse(null);
        }
    
        @Override
        public Account createAccount(Account account) {
            String encodedPassword = this.passwordEncoder.encode(account.getPassword());
            account.setPassword(encodedPassword);
            return this.accountRepository.save(account);
        }
    
        @Override
    	@Secured("ROLE_ADMIN")
        public void deleteAccount(String username) {
            this.accountRepository.deleteById(username);
        }
    }
	```  
	
	- `Config` 파일에서 확인 한 것과 같이 비밀번호는 설정파일에서 선언된 `PasswordEncoder` 빈을 통해 암호화해서 저장한다.
	- Spring Security 는 `Config` 에서 등록한 `AccountService` 빈에서 `loadUserByUsername` 을 호출해 해당 유저의 정보를 조회 한다. 
	-  `deleteAccount()` 메서드에 메서드 보안으로 `ADMIN` 권한을 가진 사용자만 호출 할 수 있도록 설정하였다.
	
- `Repository`, `Service` 테스트 코드는 아래와 같다.

	```java
	@RunWith(SpringRunner.class)
    @SpringBootTest
    public class AccountRepositoryTest {
        @Autowired
        private AccountRepository accountRepository;
        private List<Account> accountList;
    
        @Before
        public void setUp() {
            this.accountList = new ArrayList<>();
        }
    
        @After
        public void tearDown() {
            for(Account account : this.accountList) {
                this.accountRepository.deleteById(account.getUsername());
            }
        }
    
        @Test
        public void save_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADIMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            Account result = this.accountRepository.save(expected);
    
            assertThat(result, is(instanceOf(Account.class)));
            assertThat(result, is(equalTo(expected)));
        }
    
        @Test
        public void findById_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADIMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            this.accountRepository.save(expected);
    
            Account result = this.accountRepository.findById(expected.getUsername()).get();
    
            assertThat(result, is(instanceOf(Account.class)));
            assertThat(result.getUsername(), is(expected.getUsername()));
            assertThat(result.getPassword(), is(expected.getPassword()));
        }
    }
	```  
	
	```java
	@RunWith(SpringRunner.class)
    @SpringBootTest
    public class AccountServiceTest {
        @Autowired
        private AccountRepository accountRepository;
        @Autowired
        private AccountService accountService;
        private List<Account> accountList;
    
        @Before
        public void setUp() {
            this.accountList = new ArrayList<>();
        }
    
        @After
        public void tearDown() {
            for(Account account : this.accountList) {
                this.accountRepository.deleteById(account.getUsername());
            }
        }
    
        @Test
        public void createUser_Expected_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADIMIN")))
                    .build();
    
            this.accountList.add(expected);
            Account result = this.accountService.createAccount(expected);
    
            assertThat(result, is(instanceOf(Account.class)));
            assertThat(result, is(equalTo(expected)));
        }
    
        @Test
        public void readUser_Expected_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            expected =  this.accountService.createAccount(expected);
            Account result = this.accountService.readAccount(expected.getUsername());
    
            assertThat(result, is(instanceOf(Account.class)));
            assertThat(result.getUsername(), is(expected.getUsername()));
            assertThat(result.getPassword(), is(expected.getPassword()));
        }
    
        @Test
        public void loadUserByUsername_Expected_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADIMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            this.accountService.createAccount(expected);
            UserDetails result = this.accountService.loadUserByUsername(expected.getUsername());
    
            assertThat(result, is(instanceOf(UserDetails.class)));
            assertThat(result.getUsername(), is(expected.getUsername()));
            assertThat(result.getPassword(), is(expected.getPassword()));
        }
    
        @Test
        @WithMockUser(roles = {"ADMIN"})
        public void deleteAccount_Expected_success() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            this.accountService.createAccount(expected);
            this.accountService.deleteAccount(expected.getUsername());
    
            Account result = this.accountService.readAccount(expected.getUsername());
            assertThat(result, not(instanceOf(Account.class)));
            assertThat(result, is(nullValue()));
        }
    
        @Test(expected = AccessDeniedException.class)
        @WithMockUser(roles = {"USER"})
        public void deleteAccount_User_Expected_AccessDenied() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            this.accountService.createAccount(expected);
            this.accountService.deleteAccount(expected.getUsername());
        }
    
        @Test(expected = AuthenticationCredentialsNotFoundException.class)
        public void deleteAccount_UnAuth_Expected_AccessDenied() {
            Account expected =  Account.builder()
                    .insertTime(System.currentTimeMillis())
                    .username("1")
                    .password("authToken1")
                    .isAccountNonExpired(true)
                    .isAccountNonLocked(false)
                    .isCredentialsNonExpired(true)
                    .isEnabled(true)
                    .authorities(Arrays.asList(new SimpleGrantedAuthority("USER"), new SimpleGrantedAuthority("ADMIN")))
                    .build();
    
            this.accountList.add(expected);
    
            this.accountService.createAccount(expected);
            this.accountService.deleteAccount(expected.getUsername());
        }
    }
	```  

## Controller
- 인증, 세션 없이 모두 접근 가능한 `/all` URL Controller 는 아래와 같다.

	```java
	@RestController
	@RequestMapping("/all")
	public class AllController {
	    @GetMapping
	    public String main() {
	        return "All Main Page";
	    }
	
	    @GetMapping("/inner")
	    public String innerAll() {
	        return "All Inner Page";
	    }
	}
	```  
	
- `USER` 의 권한을 가져야 접근이 가능한 `/user` URL Controller 는 아래와 같다.
	- `/user/all` 은 접근 제한을 두지 않은 경로이다.
	
	```java
	@RestController
    @RequestMapping("/user")
    public class UserController {
        @GetMapping
        public String main() {
            return "User Main Page";
        }
    
        @GetMapping("/inner")
        public String innerAll() {
            return "User Inner Page";
        }
    
        @GetMapping("/all")
        public String publicPath() {
            return "User All Page";
        }
    }
	```  
	
- `ADMIN` 의 권한을 가져야 접근 가능한 `/admin` URL Controller 는 아래와 같다.
	- `/admin/all` 은 접근 제한을 두지 않은 경로 이다.
	
	```java
	@RestController
    @RequestMapping("/admin")
    public class AdminController {
        @GetMapping
        public String main() {
            return "Admin Main Page";
        }
    
        @GetMapping("/inner")
        public String inner() {
            return "Admin Inner Page";
        }
    
        @GetMapping("/all")
        public String all() {
            return "Admin All Page";
        }
    }
	```  

## Test
### Unit Test - AllControllerTest

```java
RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class AllControllerTest {
    @Autowired
    private TestRestTemplate testRestTemplate;

    @Test
    public void all_Expected_Status_OK() {
        ResponseEntity<String> responseEntity = this.testRestTemplate.getForEntity("/all", String.class);

        assertThat(responseEntity.getStatusCode(), is(HttpStatus.OK));
    }

    @Test
    public void all_Expected_Response_Ok() {
        ResponseEntity<String> responseEntity = this.testRestTemplate.getForEntity("/all", String.class);

        assertThat(responseEntity.getBody(), is("All Main Page"));
    }

    @Test
    public void allInner_Expected_Status_Ok() {
        ResponseEntity<String> responseEntity = this.testRestTemplate.getForEntity("/all/inner", String.class);

        assertThat(responseEntity.getStatusCode(), is(HttpStatus.OK));
    }

    @Test
    public void allInner_Expected_Response_Ok() {
        ResponseEntity<String> responseEntity = this.testRestTemplate.getForEntity("/all/inner", String.class);

        assertThat(responseEntity.getBody(), is("All Inner Page"));
    }
}
```  

### Unit Test - UsereControllerTest

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    @Autowired
    private AccountService accountService;

    @Test
//    @WithUserDetails("testUser1")
    @WithMockUser(roles = "USER")
    public void user_Auth_Expected_Status_Ok() throws Exception {
        this.mockMvc.perform(get("/user")).andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    public void user_Auth_Expected_Response_Ok() throws Exception{
        MvcResult mvcResult = this.mockMvc
                .perform(get("/user"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("User Main Page"));
    }

    @Test
    @WithAnonymousUser
    public void user_UnAuth_Expected_Status_RedirectLogin() throws Exception {
        this.mockMvc.perform(get("/user"))
                .andExpect(status().isFound())
                .andExpect(redirectedUrl("http://localhost/login"));
    }

    @Test
    @WithMockUser(roles = "USER")
    public void userInner_Auth_Expected_Status_Ok() throws Exception{
        this.mockMvc.perform(get("/user/inner")).andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    public void userInner_Auth_Expected_Response_Ok() throws Exception{
        MvcResult mvcResult = this.mockMvc
                .perform(get("/user/inner"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("User Inner Page"));
    }

    @Test
    @WithAnonymousUser
    public void userInner_UnAuth_Expected_Status_RedirectLogin() throws Exception {
        this.mockMvc.perform(get("/user/inner"))
                .andExpect(status().isFound())
                .andExpect(redirectedUrl("http://localhost/login"));
    }

    @Test
    @WithAnonymousUser
    public void userAll_UnAuth_Expected_Status_Ok() throws Exception {
        this.mockMvc.perform(get("/user/all")).andExpect(status().isOk());
    }

    @Test
    @WithAnonymousUser
    public void userAll_UnAuth_Expected_Response_Ok() throws Exception{
        MvcResult mvcResult = this.mockMvc
                .perform(get("/user/all"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("User All Page"));
    }
}
```  

### Unit Test - AdminControllerTest

```java
@RunWith(SpringRunner.class)
@SpringBootTest()
@AutoConfigureMockMvc
public class AdminControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
//    @WithUserDetails("testAdmin1")
    @WithMockUser(roles = "ADMIN")
    public void admin_Auth_Expected_Status_Ok() throws Exception{
        this.mockMvc
                .perform(get("/admin"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    public void admin_Auth_Expected_Response_Ok() throws Exception {
        MvcResult mvcResult = this.mockMvc
                .perform(get("/admin"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("Admin Main Page"));
    }

    @Test
    @WithAnonymousUser
    public void admin_UnAuth_Expected_Status_RedirectLogin() throws Exception {
        this.mockMvc
                .perform(get("/admin"))
                .andExpect(status().isFound())
                .andExpect(redirectedUrl("http://localhost/login"));
    }

    @Test
    @WithMockUser(roles = "USER")
    public void admin_UserAuth_Expected_Status_Forbidden() throws Exception {
        this.mockMvc
                .perform(get("/admin"))
                .andExpect(status()
                        .isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    public void adminInner_Auth_Expected_Status_Ok() throws Exception {
        this.mockMvc
                .perform(get("/admin/inner"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    public void adminInner_Auth_Expected_Response_Ok() throws Exception {
        MvcResult mvcResult = this.mockMvc
                .perform(get("/admin/inner"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("Admin Inner Page"));
    }

    @Test
    @WithAnonymousUser
    public void adminInner_UnAuth_Expected_Status_RedirectLogin() throws Exception {
        this.mockMvc
                .perform(get("/admin/inner"))
                .andExpect(status().isFound())
                .andExpect(redirectedUrl("http://localhost/login"));
    }

    @Test
    @WithMockUser(roles = "USER")
    public void adminInner_UserAuth_Expected_Status_Forbidden() throws Exception {
        this.mockMvc
                .perform(get("/admin/inner"))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithAnonymousUser
    public void adminAll_UnAuth_Expected_Status_Ok() throws Exception {
        this.mockMvc
                .perform(get("/admin/all"))
                .andExpect(status().isOk());
    }

    @Test
    @WithAnonymousUser
    public void adminAll_UnAuth_Expected_Response_Ok() throws Exception {
        MvcResult mvcResult = this.mockMvc
                .perform(get("/admin/all"))
                .andExpect(status().isOk())
                .andReturn();

        assertThat(mvcResult.getResponse().getContentAsString(), is("Admin All Page"));
    }
}
```  

### Browser Test
- `http://<server-ip>:<port>/all`(`/user/all`, `/admin/all`) 의 인증과 세션이 필요없는 경로로 접속 가능하다.

![그림 1]({{site.baseurl}}/img/spring/practice-springbootsecurity-1.png)

![그림 2]({{site.baseurl}}/img/spring/practice-springbootsecurity-2.png)

![그림 9]({{site.baseurl}}/img/spring/practice-springbootsecurity-8.png)

![그림 10]({{site.baseurl}}/img/spring/practice-springbootsecurity-10.png)

- `http://<server-ip>:<port>` 로 접속할 때 인증이 필요한 경로(`/user`, `/admin` 등) 으로 접속하면 자동으로 `/login` 경로로 Redirect 된다.

![그림 3]({{site.baseurl}}/img/spring/practice-springbootsecurity-3.png)

- `USER` 권한을 가진 계정으로 로그인에 성공하면 `/user` 페이지로 이동 된다.

![그림 4]({{site.baseurl}}/img/spring/practice-springbootsecurity-4.png)

- `/user`, `/user/inner` 등의 `USER` 권한이 있어야 하거나, 권한이 필요없는 경로에는 접근이 되지만, `/admin` 경로에는 접근이 되지 않는다.

![그림 5]({{site.baseurl}}/img/spring/practice-springbootsecurity-5.png)

![그림 6]({{site.baseurl}}/img/spring/practice-springbootsecurity-6.png)

- `ADMIN` 권한을 가진 계정으로 로그인에 성공하면 동일하게 `/user` 페이지로 이동 되지만, `/admin`, `/admin/inner` 경로에 접근이 가능하다.

![그림 7]({{site.baseurl}}/img/spring/practice-springbootsecurity-7.png)

![그림 8]({{site.baseurl}}/img/spring/practice-springbootsecurity-8.png)

---
## Reference
[Securing a Web Application](https://spring.io/guides/gs/securing-web/)   
[[스프링 프레임워크] 스프링 시큐리티 -1.스프링 시큐리티 모듈의 이해](https://m.blog.naver.com/kimnx9006/220633299198)   
[초보가 이해하는 스프링 시큐리티](https://okky.kr/article/382738)   
[Spring boot에서 Spring security를 사용하여 로그인 하기](https://wedul.site/170)   
[Spring Boot 기반 Spring Security 회원가입 / 로그인 구현하기](https://xmfpes.github.io/spring/spring-security/)   
[Spring boot - Spring Security(스프링 시큐리티) 란? 완전 해결!](https://coding-start.tistory.com/153)   
