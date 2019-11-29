--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Maven Multi Module 구성 과 빌드"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 프로젝트를 Maven Multi Module 을 사용해서 구성하고 빌드를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - Maven
    - Docker
---  

# 목표
- Spring 프로젝트를 모듈 단위로 분리해서 구성한다.
- 모듈 단위 분리를 통해 공통적으로 사용하는 부분들은 하나의 공통 모듈로 구성한다.
- 모듈 단위로 분리된 프로젝트를 빌드해서 하나의 애플리케이션을 구성한다.
- Intellij 를 사용해서 프로젝트를 구성한다.

# 방법
- `Gradle` 과 `Maven` 에서는 프로젝트를 모듈단위로 구성할 수 있도록 기능을 제공한다.
- 모듈 단위로 분리해서 프로젝트를 구성하는 것은 중복코드를 줄일수 있다는 부분에서 큰 장점이 될 수 있다.
- 잘못 사용할 경우에는 구성하고 있는 프로젝트에 치명적인 독이 될 수도 있으므로 주의하고, 잘 구성해서 사용해야 한다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-1.png)

- 단순한 예시로 구성한 프로젝트의 구조는 아래와 같다.
	- `root` : 아래 3개의 모둘을 포함하는 프로젝트로 아래 모듈을 연결해주는 역할과 공통적으로 가지는 의존성으로 구성되어 있다.
	- `core` : 공통적으로 사용하는 코드를 포함된 모듈(domain, repository)
	- `firstapp` : `core` 모듈을 사용한 Web Application 으로 Get 관련 API 를 제공한다.
	- `secondapp` : `core` 모듈을 사용한 Web Application 으로 Create, Update 관련 API 를 제공한다.
	
## root 프로젝트
- `root` 프로젝트의 하위에 모듈을 구성하기 전에 프로젝트를 생성한다.
- `File -> New -> Profject...` 를 눌러 Maven 프로젝트를 만든다.

	![그림 2]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-2.png)
	
	![그림 3]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-3.png)
	
	![그림 4]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-4.png)
	
- `root` 프로젝트의 `pom.xml` 에 하위 모듈들에서 공통으로 사용하는 의존성을 추가해준다.

	```xml	
	<!-- 생략 -->	
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <groupId>com.windowforsun</groupId>
    <artifactId>root</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    
	<!-- 생략 -->
	
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    
	<!-- 생략 -->
	
	```  
	
	- 프로젝트의 `packaging` 은 `pom` 이다.
	
## core 모듈
- 프로젝트에서 오른쪽 클릭을 한 후 `core` 모듈을 추가해준다.

	![그림 5]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-5.png)
	
	![그림 6]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-6.png)
	
	![그림 7]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-7.png)
	
	![그림 8]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-8.png)
	
- 아래처럼 `root` 프로젝트 하위에 `core` 모듈이 추가 된것을 확인 할 수 있다.

	![그림 9]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-9.png)

- `root` 프로젝트의 `pom.xml` 에는 아래와 같이 `core` 모듈이 추가 된것을 확인 할 수 있다.

	```xml
    <modules>
        <module>core</module>
    </modules>
	```  

- `core` 모듈은 Model 과 Repository 의 구현체가 포함된 공통 모듈이기 때문에 관련 의존성과 빌드관련 설정을 `pom.xml` 에 추가한다.

	```xml
	<!-- 생략 -->
    <parent>
        <artifactId>root</artifactId>
        <groupId>com.windowforsun</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>core</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <classifier>exec</classifier>
                </configuration>
            </plugin>
        </plugins>
    </build>
	```  
	
- 완성된 `core` 모듈은 아래와 같다.

	![그림 10]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-10.png)
	
- `com.windowforsun.domain.Account` 

	```java
	@Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @Entity
    public class Account {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private long id;
        private String name;
        private int age;
    }
	```  
	
- `com.windowforsun.repository.AccountRepository`

	```java
	@Repository
    public interface AccountRepository extends JpaRepository<Account, Long> {
    }
	```  
	
- `com.windowforsun.CoreApplication`

	```java
	@SpringBootApplication
    public class CoreApplication {
        public static void main(String[] args) {
            SpringApplication.run(CoreApplication.class, args);
        }
    }
	```  
	
## firstapp 모듈
- `core` 모듈을 추가한 것과 같이 `firstapp` 이름의 모듈을 추가해준다.

	![그림 11]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-11.png)

- `root` 프로젝트의 `pom.xml` 에는 아래와 같이 `firstapp` 모듈이 추가 된것을 확인 할 수 있다.

	```xml
    <modules>
        <module>core</module>
        <module>firstapp</module>
    </modules>
	```  
	
- `firstapp` 모듈은 `core` 모듈을 사용해서 Controller 를 제공하기 때문에 관련 의존성과 빌드 설정을 `pom.xml` 에 해준다.

	```xml
    <parent>
        <artifactId>root</artifactId>
        <groupId>com.windowforsun</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>firstapp</artifactId>
    <packaging>jar</packaging>

    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>

		<!-- core 모듈 의존성 -->
        <dependency>
            <groupId>com.windowforsun</groupId>
            <artifactId>core</artifactId>
            <version>1.0-SNAPSHOT</version>
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
	```  
	
- 완성된 `firstapp` 모듈은 아래와 같다.

	![그림 12]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-12.png)
	
- `com.windowforsun.controller.AccountController`

	```java
	@RestController
    @RequiredArgsConstructor
    @RequestMapping("/account")
    public class AccountController {
        private final AccountRepository accountRepository;
    
        @GetMapping("/{id}")
        public Account readById(@PathVariable long id) {
            return this.accountRepository.findById(id).orElse(null);
        }
    
        @GetMapping
        public List<Account> readAll() {
            return this.accountRepository.findAll();
        }
    }
	```  
	
- `com.windowforsun.FirstAppApplication`

	```java
	@SpringBootApplication
    public class FirstAppApplication {
        public static void main(String[] args) {
            SpringApplication.run(FirstAppApplication.class, args);
        }
    }
	```  
	
- `application.yml`

	```yaml
	# 로컬 구동시 환경 설정
	server:
      port: 8080
    
    # 로컬 구동시 h2 DB를 사용한다.
    
    ---
    # 도커로 구동시 환경 설정
    server:
      port: 8080
      
    spring:
      profiles: docker
    
      datasource:
        hikari:
          jdbc-url: jdbc:mysql://module-mysql:3306/test
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
	```  
	
- `http/readById.http`

	```http request
	GET http://localhost:80/account/1
	```  
	
- `http/readAll.http`

	```http request
	GET http://localhost:80/account
	```  
	
- `com.windowforsun.controller.TestAccountController`

	```java
	@RunWith(SpringRunner.class)
    @SpringBootTest(classes = {FirstAppApplication.class})
    @AutoConfigureMockMvc
    public class TestAccountController {
        @Autowired
        private MockMvc mockMvc;
        @Autowired
        private AccountRepository accountRepository;
        private ObjectMapper objectMapper = new ObjectMapper();
    
        @Before
        public void setUp() {
            this.accountRepository.deleteAll();
        }
    
        @Test
        public void readById_존재하는아이디는_해당Account를응답한다() throws Exception {
            // given
            Account account = Account.builder()
                    .age(1)
                    .name("name")
                    .build();
            account = this.accountRepository.save(account);
            long id = account.getId();
    
            // when
            MvcResult result = this.mockMvc
                    .perform(get("/account/{id}", id)
                            .accept(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
            assertThat(actual.getAge(), is(account.getAge()));
            assertThat(actual.getName(), is(account.getName()));
        }
    
        @Test
        public void readById_존재하지않는아이디는_Null을응답한다() throws Exception{
            // given
            long id = 123123;
    
            // when
            MvcResult result = this.mockMvc
                    .perform(get("/account/{id}", id)
                            .accept(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            assertThat(result.getResponse().getContentAsString(), is(emptyString()));
        }
    
        @Test
        public void readAll_전체Account정보를_배열로응답한다() throws Exception{
            // given
            Account account1 = Account.builder()
                    .age(1)
                    .name("name")
                    .build();
            this.accountRepository.save(account1);
            Account account2 = Account.builder()
                    .age(2)
                    .name("name2")
                    .build();
            this.accountRepository.save(account2);
    
            // when
            MvcResult result = this.mockMvc
                    .perform(get("/account")
                            .accept(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            List<Account> actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), List.class);
            System.out.println(Arrays.toString(actual.toArray()));
            assertThat(actual, hasSize(2));
        }
    
        @Test
        public void readAll_정보가없으면_빈배열을응답한다() throws Exception{
            // when
            MvcResult result = this.mockMvc
                    .perform(get("/account")
                            .accept(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            List<Account> actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), List.class);
            assertThat(actual, empty());
        }
    }
	```  
	
## secondapp 모듈
- `core` 모듈을 추가한 것과 같이 `secondapp` 이름의 모듈을 추가해준다.

	![그림 13]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-13.png)

- `root` 프로젝트의 `pom.xml` 에는 아래와 같이 `secondapp` 모듈이 추가 된것을 확인 할 수 있다.

	```xml
    <modules>
        <module>core</module>
        <module>firstapp</module>
        <module>secondapp</module>
    </modules>
	```  
	
- `secondapp` 모듈은 `core` 모듈을 사용해서 Controller 를 제공하기 때문에 관련 의존성과 빌드 설정을 `pom.xml` 에 해준다.
	- `firstapp` 모듈과 `secondapp` 모듈은 같은 Controller 에서 제공하는 메서드만 다르기 때문에 대부분의 설정이 비슷하다.

	```xml
    <parent>
        <artifactId>root</artifactId>
        <groupId>com.windowforsun</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>secondapp</artifactId>
    <packaging>jar</packaging>

    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>

		<!-- core 모듈 의존성 -->
        <dependency>
            <groupId>com.windowforsun</groupId>
            <artifactId>core</artifactId>
            <version>1.0-SNAPSHOT</version>
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
	```  
	
- 완성된 `secondapp` 모듈은 아래와 같다.

	![그림 14]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-14.png)
	
- `com.windowforsun.controller.AccountController`

	```java
	@RestController
    @RequiredArgsConstructor
    @RequestMapping("/account")
    public class AccountController {
        private final AccountRepository accountRepository;
    
        @PostMapping
        public Account create(@RequestBody Account account) {
            return this.accountRepository.save(account);
        }
    
        @PutMapping("/{id}")
        public Account updateById(@PathVariable long id, @RequestBody Account account) {
            Account result = null;
    
            if(this.accountRepository.existsById(id)) {
	            account.setId(id);
	            result = this.accountRepository.save(account);
            }
    
            return result;
        }
    }
	```  
	
- `com.windowforsun.SecondAppApplication`

	```java
	@SpringBootApplication
    public class SecondAppApplication {
        public static void main(String[] args) {
            SpringApplication.run(SecondAppApplication.class, args);
        }
    }
	```  
	
- `application.yml`

	```yaml
	# 로컬 구동시 환경 설정
	server:
      port: 8081
    
    # 로컬 구동시 h2 DB를 사용한다.
    
    ---    
  	# 도커로 구동시 환경 설정    
    server:
      port: 8080
      
    spring:
      profiles: docker
    
      datasource:
        hikari:
          jdbc-url: jdbc:mysql://module-mysql:3306/test
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
	```  
	
- `http/create.http`

	```http request	
	POST http://localhost:81/account
    Content-Type: application/json
    
    {
      "name": "name1",
      "age": 1
    }
	```  
	
- `http/updateById.http`

	```http request
	
	PUT http://localhost:81/account/1
    Content-Type: application/json
    
    {
      "name": "name11",
      "age": 11
    }
	```  
	
- `com.windowforsun.controller.TestAccountController`

	```java
	@RunWith(SpringRunner.class)
    @SpringBootTest(classes = {SecondAppApplication.class})
    @AutoConfigureMockMvc
    public class TestAccountController {
        @Autowired
        private MockMvc mockMvc;
        @Autowired
        private AccountRepository accountRepository;
        private ObjectMapper objectMapper = new ObjectMapper();
    
        @Before
        public void setUp() {
            this.accountRepository.deleteAll();
        }
    
        @Test
        public void create_생성된_Account를응답한다() throws Exception {
            // given
            Account account = Account.builder()
                    .age(1)
                    .name("name")
                    .build();
            String jsonStr = this.objectMapper.writeValueAsString(account);
    
            // when
            MvcResult result = this.mockMvc
                    .perform(post("/account")
                            .content(jsonStr)
                            .accept(MediaType.APPLICATION_JSON)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
            assertThat(actual.getAge(), is(account.getAge()));
            assertThat(actual.getName(), is(account.getName()));
        }
    
        @Test
        public void updateById_존재하는아이디_업데이트된Account를응답한다() throws Exception {
            // given
            Account account = Account.builder()
                    .age(1)
                    .name("name")
                    .build();
            this.accountRepository.save(account);
            long id = account.getId();
            account.setAge(11);
            account.setName("name11");
            String jsonStr = this.objectMapper.writeValueAsString(account);
    
            // when
            MvcResult result = this.mockMvc
                    .perform(put("/account/{id}", id)
                            .content(jsonStr)
                            .accept(MediaType.APPLICATION_JSON)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            Account actual = this.objectMapper.readValue(result.getResponse().getContentAsString(), Account.class);
            assertThat(actual.getAge(), is(account.getAge()));
            assertThat(actual.getName(), is(account.getName()));
        }
    
        @Test
        public void updateById_존재하지않는아이디_Null을반환한다() throws Exception {
            // given
            long id = 12312312;
            Account account = Account.builder()
                    .age(1)
                    .name("name")
                    .build();
            String jsonStr = this.objectMapper.writeValueAsString(account);
    
            // when
            MvcResult result = this.mockMvc
                    .perform(put("/account/{id}", id)
                            .content(jsonStr)
                            .accept(MediaType.APPLICATION_JSON)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andReturn();
    
            // then
            assertThat(result.getResponse().getContentAsString(), is(emptyString()));
        }
    }
	```  
	
## Docker 구성 및 빌드
- Docker 를 구성하는 파일은 아래와 같다.

	![그림 15]({{site.baseurl}}/img/spring/practice-mavenmultimodulebuild-15.png)
	
- `docker-compose.yml

	```yaml
	version: '3.7'
    
    services:
      firstapp:
        build:
          context: ./../
          dockerfile: docker/firstapp/Dockerfile
        ports:
          - "80:8080"
        networks:
          - module-net
        depends_on:
          - module-mysql
          
      secondapp:
        build:
          context: ./../
          dockerfile: docker/secondapp/Dockerfile
        ports:
          - "81:8080"
        networks:
          - module-net
        depends_on:
          - module-mysql
    
      module-mysql:
        build:
          context: mysql
        ports:
          - "3306:3306"
        env_file:
          - .env
        volumes:
          - ./mysql/conf:/etc/mysql/conf.d/source
          - ./mysql/sql:/docker-entrypoint-initdb.d
          - ./mysql-data:/var/lib/mysql
        command:
          - --default-authentication-plugin=mysql_native_password
        networks:
          - module-net
    
    networks:
      module-net:
	```  
	
	- `firstapp` 은 80 포트, `secondapp` 은 81 포트를 사용한다.
	
- `firstapp/Dockerfile`

	```dockerfile
	### BUILD image
	FROM maven:3-jdk-11 as builder
	# create app folder for sources
	RUN mkdir -p /build
	WORKDIR /build
	#COPY pom.xml /build
	COPY pom.xml /build
	#Copy source code
	COPY ./core /build/core
	COPY ./firstapp /build/firstapp
	COPY ./secondapp /build/secondapp
	#packaging project
	RUN mvn package
	
	FROM openjdk:11-slim as runtime
	# define application name
	ENV APP_NAME firstapp
	#Set app home folder
	ENV APP_HOME /app
	#Possibility to set JVM options (https://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
	ENV JAVA_OPTS=""
	#Create base app folder
	RUN mkdir $APP_HOME
	#Create folder to save configuration files
	RUN mkdir $APP_HOME/config
	#Create folder with application logs
	RUN mkdir $APP_HOME/log
	VOLUME $APP_HOME/log
	VOLUME $APP_HOME/config
	WORKDIR $APP_HOME
	#Copy executable jar file from the builder image
	COPY --from=builder /build/$APP_NAME/target/*.jar app.jar
    #Execute app docker profile
	ENTRYPOINT exec java $JAVA_OPTS -Dspring.profiles.active=docker -jar app.jar $0 $@
	```  
	
- `secondapp/Dockerfile`

	```dockerfile
	### BUILD image
    FROM maven:3-jdk-11 as builder
    # create app folder for sources
    RUN mkdir -p /build
    WORKDIR /build
    #COPY pom.xml /build
    COPY pom.xml /build
    #Copy source code
    COPY ./core /build/core
    COPY ./firstapp /build/firstapp
    COPY ./secondapp /build/secondapp
    #packaging project
    RUN mvn package
    
    FROM openjdk:11-slim as runtime
    # define application name
    ENV APP_NAME secondapp
    #Set app home folder
    ENV APP_HOME /app
    #Possibility to set JVM options (https://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
    ENV JAVA_OPTS=""
    #Create base app folder
    RUN mkdir $APP_HOME
    #Create folder to save configuration files
    RUN mkdir $APP_HOME/config
    #Create folder with application logs
    RUN mkdir $APP_HOME/log
    VOLUME $APP_HOME/log
    VOLUME $APP_HOME/config
    WORKDIR $APP_HOME
    #Copy executable jar file from the builder image
    COPY --from=builder /build/$APP_NAME/target/*.jar app.jar
    #Execute app docker profile
    ENTRYPOINT exec java $JAVA_OPTS -Dspring.profiles.active=docker -jar app.jar $0 $@
	```  
	
- `mysql/Dockerfile`

	```dockerfile
	FROM mysql:8.0.17
    
    RUN apt-get update
    RUN apt-get install -y libevent-dev
	```  
	
- `mysql/config/custom.cnf`

	```
	bind-address=0.0.0.0
	```  
	
- `mysql/sql/init.sql`

	```sql
	create table account (
      id bigint not null auto_increment,
      name varchar(255),
      age bigint,
      primary key (id)
    ) engine = InnoDB;
	```  
	
- `.env`

	```
	MYSQL_DATABASE=test
    MYSQL_USER=hello
    MYSQL_PASSWORD=hello
    MYSQL_ROOT_PASSWORD=root
	```  
	
- `docker-compose up --build` 를 통해 빌드를 한다.

## 테스트
- 각 모듈에 있는 `http` 를 통해 테스트를 진행한다.
- `secondapp` 의 `create.http`
	
	```
	POST http://localhost:81/account
    
    HTTP/1.1 200 
    Content-Type: application/json
    Transfer-Encoding: chunked
    Date: Sat, 23 Nov 2019 18:55:37 GMT
    
    {
      "id": 1,
      "name": "name1",
      "age": 1
    }
	```  
	
- `firstapp` 의 `readById.http`

	```
	GET http://localhost:80/account/1
    
    HTTP/1.1 200 
    Content-Type: application/json
    Transfer-Encoding: chunked
    Date: Sat, 23 Nov 2019 18:56:25 GMT
    
    {
      "id": 1,
      "name": "name1",
      "age": 1
    }
	```  
	
- `firstapp` 의 `readAll.http`

	```
	GET http://localhost:80/account
    
    HTTP/1.1 200 
    Content-Type: application/json
    Transfer-Encoding: chunked
    Date: Sat, 23 Nov 2019 18:57:03 GMT
    
    [
      {
        "id": 1,
        "name": "name1",
        "age": 1
      }
    ]
	```  
	
- `secondapp` 의 `updateById.http`

	```
	PUT http://localhost:81/account/1
    
    HTTP/1.1 200 
    Content-Type: application/json
    Transfer-Encoding: chunked
    Date: Sat, 23 Nov 2019 18:57:42 GMT
    
    {
      "id": 1,
      "name": "name11",
      "age": 11
    }
	```  
	
## 모듈 `src/test` 에 있는 클래스 사용하기
- 사용할 클래스가 있는 모듈 `pom.xml` 에 아래 내용을 추가한다.

	```xml
	<plugin>
	  <groupId>org.apache.maven.plugins</groupId>
	  <artifactId>maven-jar-plugin</artifactId>
	  <version>2.2</version>
	  <executions>
	    <execution>
	      <goals>
	        <goal>test-jar</goal>
	      </goals>
	    </execution>
	  </executions>
	</plugin>
	```  
	
- 사용하는 모듈 `pom.xml` 에 아래 의존성을 추가한다.

	```xml
	<dependency>
	  <groupId>com.windowforsun</groupId>
	  <artifactId>core</artifactId>
	  <version>1.0</version>
	  <type>test-jar</type>
	  <scope>test</scope>
	</dependency>
	```  

	
---
## Reference
[메이븐 다중 모듈 프로젝트에 대해](https://windwolf.tistory.com/18)  
[멀티모듈 설계 이야기 with Spring, Gradle](http://woowabros.github.io/study/2019/07/01/multi-module.html)  
[maven 멀티 모듈](http://wonwoo.ml/index.php/post/601)  
[Multi-Module Project With Spring Boot](https://www.baeldung.com/spring-boot-multiple-modules)  
[Could not find artifact in a multi-module project](https://stackoverflow.com/questions/41105954/could-not-find-artifact-in-a-multi-module-project?rq=1)