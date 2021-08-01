--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Cloud Config for DB"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Config 설정 저장소를 DB를 사용해서 구성하는 법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Cloud Config
    - JDBC
    - MySQL
    - Redis
toc: true
use_math: true
---  

## Environment Repository
[Spring Cloud Config for Git Repository]({{site.baseurl}}{% link _posts/spring/2021-07-28-spring-concept-spring-cloud-config-git.md %}) 
에서는 `Git Repository` 에 설정 파일을 저장해서 `Config Server` 와 연동하는 방법에 대해 알아봤다. 
`Spring Cloud Config` 는 `Git Repository` 외에도 설정 파일은 다른 저장소에 저장하고 연동할 수 있다.  

[Spring Cloud Config Server Environment Repository](https://cloud.spring.io/spring-cloud-config/reference/html/#_environment_repository)
문서를 보면 아래와 같은 저장소를 사용해서 설정파일 내용을 저장하고, 
`Spring Cloud Config Server` 와 연동 할 수 있다.  

- `File System`
- `JDBC`
- `Redis`
- `AWS S3`

이번 포스트에서는 `DB`(`MySQL`, `Redis`) 를 사용해서 설정파일 내용을 저장하고, 
`Spring Cloud Config Server` 와 연동하는 방법에 대해 알아본다.  

테스트는 `Spring Cloud Config Server` 에서 아래와 같이 요청했을 때 설정 파일을 읽어 응답을 주는 것을 활용해서 검증을 수행한다.  

```bash
$ curl http://spring-cloud-config-server-domain:{port}/{application}/{profile}/{label}
{
  "name": "lowercase",
  "profiles": [
    "dev"
  ],
  "label": "latest",
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "lowercase-dev",
      "source": {
        "type": "db",
        "group.key1": "db-dev-1",
        "group.key2": "db-dev-2"
      }
    }
  ]
}
```


### Spring Cloud Config for DB 구조
구조는 `Spring Cloud Config for Git Repository` 와 크게 다르지 않다. 
아래 그림을 보면 저장소만 `DB`(`MySQL`, `Redis`) 로 변경된 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/spring/concept-spring-cloud-config-db-1.png)


### Spring Cloud Server

#### build.gradle
`MySQL`, `Redis` 저장소와 연동을 위해 관련 의존성을 추가한다. 
그리고 로컬 테스트에서 사용할 `TestContainers` 관련 의존성도 추가해 준다.  

```groovy
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
    id 'org.springframework.boot' version '2.4.2'
}

ext {
    springCloudVersion = '2020.0.1'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
group 'com.windowforsun.springcloudconfig.configserver'
version '1.0-SNAPSHOT'


repositories {
    mavenCentral()
}

task printTask() {
    println(project.getProperties())
}

dependencies {
    compile('org.springframework.cloud:spring-cloud-config-server')
    runtimeOnly 'mysql:mysql-connector-java'
    implementation ('org.springframework.boot:spring-boot-starter-actuator')

    // db
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    // redis
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'

    testImplementation('org.springframework.boot:spring-boot-starter-test')
    testCompile "org.testcontainers:testcontainers:1.15.3"
    testCompile "org.testcontainers:junit-jupiter:1.15.3"
    testImplementation "org.testcontainers:mysql:1.15.3"
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
test {
    useJUnitPlatform()
}
```  

### application.yaml
테스트 구성은 `Profile` 을 `MySQL`, `Redis` 로 구분해서 구성한다.  

아래는 `MySQL`(`JDBC`)를 설정 저장소로 사용하는 `Profile` 인 `jdbcbacnend` 의 내용이 있는 `application-jdbcbackend.yaml` 파일 내용이다.  

```yaml
server:
  port: 8070
spring:
  autoconfigure:
    # jdbc 환경 구동 시 redis 자동설정 비활성화
    exclude: org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration, \
             org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration
  application:
    name: config-server
  datasource:
    hikari:
      connection-timeout: 5000
      maximum-pool-size: 10
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: root
  cloud:
    # jdbc 를 사용해서 MySQL 저장소에서 설정내용을 읽어오는 쿼리
    config:
      server:
        jdbc:
          sql: SELECT prop_key, value FROM cloud_config where application=? and profile=? and label=?
          order: 1
		  
  # jdbc 를 사용해서 설정내용을 연동할때 jdbc profile 를 추가로 활성해 해줘야 한다. 
#  profiles:
#    active: jdbc
```  

설정 파일에서 나와 있는 것과 같이, 
`MySQL` 을 사용해서 연동할때 설정파일을 저장해두는 테이블이 필요하다. 
테스트에서는 `cloud_config` 라는 테이블을 사용하고 테이블 생성쿼리는 아래와 같다.  

```sql
create table cloud_config (
    id integer not null auto_increment,
    application varchar(255),
    profile varchar(255),
    label varchar(255),
    prop_key varchar(255),
    value varchar(255),
    primary key (id)
);
```  

다음으로 아래는 `Redis` 를 설정 저장소로 사용하는 `Profile` 인 `redisbackend` 의 내용이 있는 `application-redisbackend.yaml` 파일 내용이다.  

```yaml
server:
  port: 8070
spring:
  # redis 환경 구동 시 jdbc 자동설정 비활성화
  autoconfigure:
    exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration, \
             org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
  application:
    name: config-server
  redis:
    host: localhost
    port: 6379

  # redis 를 사용해서 설정내용을 연동할때 redis profile 를 추가로 활성해 해줘야 한다. 
#  profiles:
#    active: redis
```  

`Redis` 는 설정내용 저장을 위해 `HashMap` 구조에 `Key`, `Value` 구조로 설정해 주면 된다.  

### DbConfigServerApplication
`Config Server` 를 실행시키는 메인클래스는 아래와 같다. 
기존 `Git Repository` 와 비교해서 변경사항은 없다.  

```java
@SpringBootApplication
@EnableConfigServer
public class DbConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DbConfigServerApplication.class, args);
    }
}
```  

### MySQL(JDBC)
먼저 `MySQL` 을 설정 저장소로 사용해서 `Spring Cloud Config Server` 와 연동하는 테스트를 살펴본다. 
`Spring Boot Test` 를 기반으로 테스트를 진행하고, 
`MySQL` 은 `TestContainers` 를 사용해서 테스트 애플리케이션에서 사용할 수 있도록 구성한다. 
그리고 `@BeforeEach` 메소드에서 테이블 생성 및 설정내용을 추가하는 작업을 수행한다.  

`@ActiveProfiles` 을 보면 `jdbcbackend` 와 `jdbc` 를 활성화 해주고 있다. 
앞서 설명한 것처럼 `JDBC` 를 사용해서 `Config Server` 를 연동할 때 `jdbc` `Profile` 를 꼭 활성화 해줘야 한다.  

```java
@ActiveProfiles(profiles = {"jdbcbackend", "jdbc"})
@SpringBootTest
@AutoConfigureMockMvc
@ExtendWith(SpringExtension.class)
@ContextConfiguration(initializers = JdbcConfigServerTest.Initializer.class)
public class JdbcConfigServerTest {
    @ClassRule
    public static MySQLContainer mysql = new MySQLContainer(DockerImageName.parse("mysql:8"))
            .withDatabaseName("test")
            .withUsername("root")
            .withPassword("root");

    static {
        mysql.start();
    }

    public static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
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
    private MockMvc mockMvc;
    public static final String URL = "/{application}/{profile}/{label}";
    private static boolean IS_INIT = false;
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    public void setUp() {
        if (IS_INIT) {
            return;
        }

        this.jdbcTemplate.execute(
                "create table cloud_config (\n" +
                        "    id integer not null auto_increment,\n" +
                        "    application varchar(255),\n" +
                        "    profile varchar(255),\n" +
                        "    label varchar(255),\n" +
                        "    prop_key varchar(255),\n" +
                        "    value varchar(255),\n" +
                        "    primary key (id)\n" +
                        ");"
        );

        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'dev', 'latest', 'type', 'jdbc');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'dev', 'latest', 'group.key1', 'jdbc-dev-1');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'dev', 'latest', 'group.key2', 'jdbc-dev-2');");

        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'real', 'latest', 'type', 'jdbc');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'real', 'latest', 'group.key1', 'jdbc-real-1');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('lowercase', 'real', 'latest', 'group.key2', 'jdbc-real-2');");

        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'dev', 'latest', 'type', 'JDBC');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'dev', 'latest', 'group.key1', 'JDBC-DEV-1');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'dev', 'latest', 'group.key2', 'JDBC-DEV-2');");

        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'REAL', 'latest', 'type', 'JDBC');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'REAL', 'latest', 'group.key1', 'JDBC-REAL-1');");
        this.jdbcTemplate.execute("insert into cloud_config (application, profile, label, prop_key, value) values ('uppercase', 'REAL', 'latest', 'group.key2', 'JDBC-REAL-2');");

        IS_INIT = true;
    }

    @Test
    public void lowercase_dev() throws Exception {
        // given
        String application = "lowercase";
        String profile = "dev";
        String label = "latest";

        // when
        this.mockMvc
                .perform(get("/{application}/{profile}/{label}", application, profile, label))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name", is(application)))
                .andExpect(jsonPath("$.profiles", contains(profile)))
                .andExpect(jsonPath("$.label", is(label)))
                .andExpect(jsonPath("$.propertySources[0].name", is("lowercase-dev")))
                .andExpect(jsonPath("$.propertySources[0].source.type", is("jdbc")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key1']", is("jdbc-dev-1")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key2']", is("jdbc-dev-2")))
        ;
    }

    @Test
    public void uppercase_real() throws Exception {
        // given
        String application = "uppercase";
        String profile = "real";
        String label = "latest";

        // when
        this.mockMvc
                .perform(get("/{application}/{profile}/{label}", application, profile, label))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name", is(application)))
                .andExpect(jsonPath("$.profiles", contains(profile)))
                .andExpect(jsonPath("$.label", is(label)))
                .andExpect(jsonPath("$.propertySources[0].name", is("uppercase-real")))
                .andExpect(jsonPath("$.propertySources[0].source.type", is("JDBC")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key1']", is("JDBC-REAL-1")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key2']", is("JDBC-REAL-2")))
        ;
    }
}
```  

### Redis
다음으로는 `Redis` 를 설정저장소로 사용해서 `Config Server` 와 연동하는 테스트다. 
먼저 진행한 `MySQL` 와 큰 차이는 없고 대부분 동일한 방식으로 테스트를 진행한다. 
`@BeforeEach` 메소드에서는 `Redis HashMap` 에 설정내용을 추가하고 있는 것을 확인 할 수 있다.  

`Redis` 와 `Config Server` 를 연동 할 때도 `@ActiveProfiles` 에서 활성화 하고 있는 것과 같이 `redis` `Profile` 을 필수로 활성화 해줘야 한다.  

```java
@ActiveProfiles(profiles = {"redisbackend", "redis"})
@SpringBootTest
@AutoConfigureMockMvc
@ExtendWith(SpringExtension.class)
@ContextConfiguration(initializers = RedisConfigServerTest.Initializer.class)
public class RedisConfigServerTest {
    @ClassRule
    public static GenericContainer redis = new GenericContainer(DockerImageName.parse("redis:5.0"))
            .withExposedPorts(6379);

    static {
        redis.start();
    }

    public static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            TestPropertyValues.of(
                    "spring.redis.host=" + redis.getHost(),
                    "spring.redis.port=" + redis.getFirstMappedPort()
            ).applyTo(applicationContext.getEnvironment());
        }
    }

    @Autowired
    private MockMvc mockMvc;
    private static boolean IS_INIT = false;
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @BeforeEach
    public void setUp() {
        if (IS_INIT) {
            return;
        }

        this.redisTemplate.opsForHash().putAll(
                "lowercase-dev",
                Map.of(
                        "type", "redis",
                        "group.key1", "redis-dev-1",
                        "group.key2", "redis-dev-2"
                )
        );
        this.redisTemplate.opsForHash().putAll(
                "lowercase-real",
                Map.of(
                        "type", "redis",
                        "group.key1", "redis-real-1",
                        "group.key2", "redis-real-2"
                )
        );
        this.redisTemplate.opsForHash().putAll(
                "uppercase-dev",
                Map.of(
                        "type", "REDIS",
                        "group.key1", "REDIS-DEV-1",
                        "group.key2", "REDIS-DEV-2"
                )
        );
        this.redisTemplate.opsForHash().putAll(
                "uppercase-real",
                Map.of(
                        "type", "REDIS",
                        "group.key1", "REDIS-REAL-1",
                        "group.key2", "REDIS-REAL-2"
                )
        );

        IS_INIT = true;
    }

    @Test
    public void lowercase_dev() throws Exception {
        // given
        String application = "lowercase";
        String profile = "dev";
        String label = "latest";

        // when
        this.mockMvc
                .perform(get("/{application}/{profile}/{label}", application, profile, label))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name", is(application)))
                .andExpect(jsonPath("$.profiles", contains(profile)))
                .andExpect(jsonPath("$.label", is(label)))
                .andExpect(jsonPath("$.propertySources[0].name", is("redis:lowercase-dev")))
                .andExpect(jsonPath("$.propertySources[0].source.type", is("redis")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key1']", is("redis-dev-1")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key2']", is("redis-dev-2")))
        ;
    }

    @Test
    public void uppercase_real() throws Exception {
        // given
        String application = "uppercase";
        String profile = "real";
        String label = "latest";

        // when
        this.mockMvc
                .perform(get("/{application}/{profile}/{label}", application, profile, label))

                // then
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name", is(application)))
                .andExpect(jsonPath("$.profiles", contains(profile)))
                .andExpect(jsonPath("$.label", is(label)))
                .andExpect(jsonPath("$.propertySources[0].name", is("redis:uppercase-real")))
                .andExpect(jsonPath("$.propertySources[0].source.type", is("REDIS")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key1']", is("REDIS-REAL-1")))
                .andExpect(jsonPath("$.propertySources[0].source.['group.key2']", is("REDIS-REAL-2")))
        ;
    }
}
```  

### 활용
[Spring Cloud Config with Spring Cloud Bus]({{site.baseurl}}{% link _posts/spring/2021-07-30-spring-concept-spring-cloud-config-git-bus.md %})
의 구성에서 `Spring Cloud Config Server` 만 이번 포스트에서 진행한 걸로 변경해주면, 
동일하게 `Spring Cloud Bus` 를 연동해서 함께 사용할 수 있다.  



---
## Reference
[JDBC Backend](https://cloud.spring.io/spring-cloud-config/reference/html/#_jdbc_backend)  
[Redis Backend](https://cloud.spring.io/spring-cloud-config/reference/html/#_redis_backend)  
[JDBC Backend Spring Cloud Config](https://www.devglan.com/spring-cloud/jdbc-backend-spring-cloud-config)  
[Spring Cloud Config Server with JDBC backend](https://medium.com/@kishansingh.x/spring-cloud-config-server-with-jdbc-backend-a8a629846115)  
[Spring Cloud Config Server Composite Configuration (JDBC + Redis + AWSS3)](https://medium.com/swlh/spring-cloud-config-server-composite-configuration-jdbc-redis-awss3-d849c4d94383)  
