--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot Spring Security"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 Spring Security 로 로그인 및 권한을 관리하자'
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
- Spring Security 기반 로그인을 수행한다.

# 방법
- Spring Security 의 저장소는 Redis 를 사용한다.

## Spring Security

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
</dependencies>
```  

## Application.properties

```properties
# 세션 시간
server.servlet.session.timeout=30s
# 세션 저장소
spring.session.store-type=redis
```  

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
	
	- `Account` 의 `authorities` 는 해당 유저가 가진을 권한과 관련된 프로퍼티이다.
	
- `Account` 의 `Redis` 관련 동작을 수행하는 `AccountRepository` 인터페이스 이다.

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
        public void deleteAccount(String username) {
            this.accountRepository.deleteById(username);
        }
    }
	```  
	
	- `Config` 파일에서 확인 할 수 있지만 비밀번호는 설정파일에서 선언된 `PasswordEncoder` 빈을 통해 암호화해서 저장한다.
	
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
        public void createUser_success() {
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
        public void readUser_success() {
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
        public void loadUserByUsername_success() {
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
        public void deleteAccount_success() {
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
            this.accountService.deleteAccount(expected.getUsername());
    
            Account result = this.accountService.readAccount(expected.getUsername());
            assertThat(result, not(instanceOf(Account.class)));
            assertThat(result, is(nullValue()));
        }
    }
	```  
	
## Form 로그인 기반, Spring Security Config

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private AccountService accountService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/user").hasAuthority("USER")
                .antMatchers("/admin").hasAuthority("ADMIN")
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .and()
                .logout();
    }
    
    

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(this.accountService).passwordEncoder(this.passwordEncoder());
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public HttpSessionStrategy httpSessionStrategy() {
        return new HeaderHttpSessionStrategy();
    }
}
```  

- 간단한 예제를 위해 `csrf` 설정은 `disable` 한 상태로 진행한다.


---
## Reference
[]()   
