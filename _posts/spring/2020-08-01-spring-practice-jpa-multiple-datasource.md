--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Multiple DataSource(Database) Transaction"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Replication, Sharding 에서 필요한 JPA 다중 Datasource 를 Transaction 을 활용해서 구현해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - JPA
    - Replication
    - Sharding
    - DataSource
    - Transaction
toc: true
use_math: true
---  

## JPA Multiple DataSource
보편적으로 `Database` 튜닝을 위해서 `Replication` 이나 `Sharding` 을 구성한다. 
이를 통해 시스템 하나로 몰리던 부하를 여러 시스템으로 분산시킬 수 있다. 
이런 `Database` 구조를 사용하기 위해서는 별도의 `Proxy` 를 두거나, 
애플리케이션에서 이런 구조에 대한 추가 처리가 필요하다.  

애플리케이션에서 지원 가능하도록 하는 다양한 방법이 있겠지만, 
그 중 `Spring` 애플리케이션에서 코드를 통해 적용하는 아래 3가지 방법에 대해 알아본다. 
- `Transaction` 의 `readOnly` 속성을 사용하는 방법
- 패키지를 분리해서 사용하는 방법
- `Annotation` 을 사용하는 방법

여러 `DataSource` 를 사용하는 상황은 많지만 그중, `Replication` 상황을 가정해서 예제를 진행한다.  

## Docker Swarm 기반 Database 구성
테스트를 위해서 `Replication` 구조의 `Database` 는 [여기]({{site.baseurl}}{% link _posts/docker/2020-07-31-docker-practice-mysql-replication-template.md %})
에서 구성한 템플릿과 방식을 사용해서 아래와 같이 `docker-compose.yaml` 파일을 작성해서 구성한다. 

```yaml
version: '3.7'

services:
  master-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master/conf.d/:/etc/mysql/conf.d
      - ./master/init-1/:/docker-entrypoint-initdb.d/
    ports:
      - 33000:3306

  slave-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-1
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
    networks:
      - db-net
    volumes:
      - ./slave/conf.d/:/etc/mysql/conf.d
      - ./slave/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34000:3306

  master-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master/conf.d/:/etc/mysql/conf.d
      - ./master/init-2/:/docker-entrypoint-initdb.d/
    ports:
      - 33001:3306

  slave-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-2
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
    networks:
      - db-net
    volumes:
      - ./slave/conf.d/:/etc/mysql/conf.d
      - ./slave/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34001:3306

  adminer:
    image: adminer
    restart: always
    ports:
      - 8888:8080
    networks:
      - db-net

networks:
  db-net:
```  

`master-db-1`, `master-db-2` 서비스의 차이는 `master/init-*/_init.sql` 파일인데, 각 파일 내용은 아래와 같다.  

`master/init-1/_init.sql`

```sql

create database test;

use test;

CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam(value) values('a');
insert into exam(value) values('b');
insert into exam(value) values('c');

create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;
```  


`master/init-2/_init.sql`

```sql

create database test;

use test;

CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam2(value2) values('a');
insert into exam2(value2) values('b');
insert into exam2(value2) values('c');

create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;
```  


템플릿 실행은 `docker-compose up` 혹은 `docker-compose up --build` 명령으로 할 수 있다. 
현재 `Database` 들이 호스트 하나에서 모두 실행되기 때문에 애플리케이션에서 `Database` 의 구분은 포트로 한다. 
각 `Database` 비교 정보를 나열하면 아래와 같다. 
   
- `master-db-1` : 33000 포트, exam 테이블
- `master-db-2` : 33001 포트, exam2 테이블
- `slave-db-1` : 34000 포트, exam 테이블, `master-db-1` 의 `slave`
- `slave-db-1` : 34001 포트, exam2 테이블, `master-db-2` 의 `slave`


## Spring 애플리케이션
애플리케이션은 `Spring Boot` 기반이고, 빌드 툴은 `Gradle` 을 사용했다. 
프로젝트 디렉토리 구조는 아래와 같다. 


```
.
├── aop
│   ├── build.gradle
|   .. 생략 ..
├── build.gradle
├── gradlew
├── gradlew.bat
├── package
│   ├── build.gradle
|   .. 생략 ..
├── settings.gradle
└── transaction
    ├── build.gradle
    .. 생략 ..
```  

하나의 프로젝트에 `aop`, `package`, `transaction` 이라는 이름으로 3가지 모듈로 구성했다. 
각 모듈이 앞서 설명한 `JPA Multiple DataSource` 를 구성하는 하나의 방법에 대한 구현체이다. 

먼저 `build.gradle` 파일 내용은 아래와 같다. 

```groovy
plugins {
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'org.springframework.boot' version '2.2.1.RELEASE'
    id 'java'
}

group 'com.windowforsun'
version '1.0-SNAPSHOT'

bootJar {
    enabled = false
}

sourceCompatibility = 1.8

allprojects {
    repositories {
        mavenCentral()
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'
    apply plugin: 'org.springframework.boot'

    dependencies {
        compile 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testCompile group: 'junit', name: 'junit', version: '4.12'
        testImplementation group: 'org.hamcrest', name: 'hamcrest-all', version: '1.3'
    }
}
```  

`settings.gradle` 파일은 아래와 같다. 

```groovy
rootProject.name = 'replicationexam'
include 'demo'
include 'transaction'
include 'package'
include 'aop'
```  

각 애플리케이션에 필요한 내용이 있는 `application.yaml` 내용은 아래와 같다.

```yaml
spring:
  datasource:
    masterdb1:
      jdbc-url: jdbc:mysql://localhost:33000/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    slavedb1:
      jdbc-url: jdbc:mysql://localhost:34000/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    masterdb2:
      jdbc-url: jdbc:mysql://localhost:33001/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    slavedb2:
      jdbc-url: jdbc:mysql://localhost:34001/test?serverTimezone=Asia/Seoul
      username: root
      password: root
  jpa:
    database: mysql
    hibernate:
      use-new-id-generator-mappings: false
      ddl-auto: update
    generate-ddl: true

logging:
  level:
    com.windowforsun: DEBUG
    org.springframework.jdbc.datasource.SimpleDriverDataSource: DEBUG
    org.springframework.jdbc.datasource: DEBUG
    org.hibernate.SQL: DEBUG
```  

- `.spring.datasource` : 하위 필드에 필요한 데이터베이스 연결에 필요한 `jdbc-url`, `username`, `password` 정보를 설정한다. 
커넥션 풀로는 `hikari` 를 사용하기 때문에 `jdbc-url` 필드를 사용한다. 
- `.spring.jpa` : 하위 필드에 `jpa` 관련 설정을 한다. 
- `.logging` : 애플리케이션에서 로그를 출력할 패키지나 클래스 이름과 레벨을 명시해 준다. 

`Database` 관련 상수인 `EnumDB` 내용은 아래와 같다. 

```java
public enum EnumDB {
    MASTER_DB_1,
    SLAVE_DB_1,
    MASTER_DB_2,
    SLAVE_DB_2
}
```  

공통적인 내용은 여기까지 이고, 
이제 부턴 앞서 설명한 3가지 방법에 대해 알아본다. 
각 방법이 정상적으로 동작하는 지에 대한 판단은 테스트 코드와 로그를 사용한다. 


### Transaction
`Transaction` 의 `readOnly` 필드를 사용하는 방법이다. 
`Master` 에서는 `CRUD` 연산을 모두 할 수 있지만, `Slave` 에서는 `R` 연산만 수행한다는 특징을 사용한다. 
`Transaciton` 의 `readOnly` 필드 값이 `true` 이면 `Slave` 의 `DataSource` 를 설정해 주는 방식이다.   

데이터베이스는 `Master1`, `Slave1` 만 사용한다.  

디렉토리 구조는 아래와 같다. 

```
transaction
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── transaction
    │   │               ├── TransactionApplication.java
    │   │               ├── config
    │   │               │   ├── DataSourceConfig.java
    │   │               │   └── RoutingDataSource.java
    │   │               ├── constant
    │   │               │   └── EnumDB.java
    │   │               ├── domain
    │   │               │   └── Exam.java
    │   │               ├── repository
    │   │               │   └── ExamRepository.java
    │   │               └── service
    │   │                   ├── ExamMasterService.java
    │   │                   └── ExamSlaveService.java
    │   └── resources
    │       └── application.yaml
    └── test
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── transaction
        │               └── service
        │                   ├── ExamMasterServiceTest.java
        │                   ├── ExamSlaveServiceTest.java
        │                   └── MultipleDataSourceTest.java
        └── resources
```  

우선 별도의 설명이 필요없는 여타 클래스 내용은 아래와 같다. 

```java
@SpringBootApplication
public class TransactionApplication {
    public static void main(String[] args) {
        SpringApplication.run(TransactionApplication.class, args);
    }
}
```  

```java
@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table(
        name = "exam"
)
public class Exam {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private long id;
    @Column
    private String value;
    @Column
    private LocalDateTime datetime;
}
```  

```java
@Repository
public interface ExamRepository extends JpaRepository<Exam, Long> {
}
```  

생성되는 `DataSource` 는 `Master`, `Slave` 2개이기 때문에 이를 적절한 상황에 라우팅해주는 역할이 필요하다. 
이러한 역할을 수행하는 클래스가 `RoutingDataSource` 클래스이다. 

```java
@Slf4j
public class RoutingDataSource extends AbstractRoutingDataSource {
    private static ThreadLocal<EnumDB> latestDB = new ThreadLocal<>();

    static {
        latestDB.set(EnumDB.MASTER_DB_1);
    }

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isSlave = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        EnumDB dataSource = EnumDB.MASTER_DB_1;

        if(isSlave) {
            dataSource = EnumDB.SLAVE_DB_1;
        }

        log.debug(dataSource.name());
        latestDB.set(dataSource);

        try {
            Thread.sleep(100);
        } catch(Exception e) {

        }

        return dataSource;
    }

    public static EnumDB getLatestDB() {
        return latestDB.get();
    }
}
```  

`RoutingDataSource` 클래스는 `org.springframework.jdbc.datasource.lookup` 패키지의 `AbstractRoutingDataSource` 를 상속해 구현한다. 
`AbstractionRoutingDatasource` 클래스의 `determineCurrentLookupKey()` 메소드 구현을 통해 상황마다 적절한 지정된 `DataSource` 키를 반환해 결정 할 수 있다.  

현재 트랜잭션의 상태가 `readOnly` 상태이면 `Slave DataSource` 키를 반환하고, 
`readOnly` 상태가 아니면 `Master DataSource` 키를 반환한다.  

`ThreadLocal` 타임의 `latestDB` 프로퍼티는 테스트를 위해 추가했다. 
그리고 `Thread.sleep(100);` 또한 테스트를 진행하며, `Replication` 에서 발생할수 있는 복제 지연 현상에 방어책으로 임시로 추가한 내용이다.  

`Database`, `DataSource` 관련 설정 파일인 `DataSourceConfig` 클래스 내용은 아래와 같다. 

```java
@Configuration
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
@EnableJpaRepositories(basePackages = {"com.windowforsun.transaction"})
@EnableTransactionManagement
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.masterdb1")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slavedb1")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource masterDataSource,
            @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        HashMap<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(EnumDB.MASTER_DB_1, masterDataSource);
        dataSourceMap.put(EnumDB.SLAVE_DB_1, slaveDataSource);
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource);

        return routingDataSource;
    }

    @Primary
    @Bean
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}
```  

`@EnableAutoConfiguration` 의 `exclude` 필드에 `DataSourceAutoConfiguration.class` 를 설정해서 의존성에서 사이클 현상이 발생하지 않도록 한다. 
`DataSourceAutoConfiguration` 을 제외시키지 않으면 해당 클래스에서 `DataSource` 의 빈을 다시 참조하기 때문에 사이클 현상이 발생한다.  

그리고 `@EnableJpaRepositories` 의 `basePackages` 필드에 `Repository` 클래스가 포함된 패키지를 설정해준다. 
해당 필드가 설정되지 않으면 `Repository` 빈들과 다른 빈들간의 의존성이 만들어지지 않는다.  

마지막으로 트랜잭션을 사용해야하기 때문에 `@EnableTransactionManagement` 를 선언해준다.  

`DataSource` 빈은 `masterDatasource` 와 `slaveDataSource` 가 있다. 
각 빈을 생성할 때 `@ConfigurationProperties` 와 `prefix` 를 사용해서 `properties` 의 내용을 바탕으로 `DataSource` 빈을 생성한다.  

`routingDataSource` 에서는 `RoutingDataSource` 클래스에서 사용 할 수 있는 `DataSource` `Lookup Table` 관련 빈을 생성한다. 
파라미터로 선언한 `masterDataSource` 와 `slaveDataSource` 를 받아 `EnumDB` 의 값과 매칭 시켜 `Map` 구조로 설정 한다. 
그리고 기본 `DataSource` 로는 `masterDataSource` 를 설정한다.  

`dataSource` 는 `@Primary` 를 선언해 `DataSource` 빈을 찾게 되면 해당 빈이 주로 사용된다. 
그리고 `LazyConnectionDataSourceProxy` 를 바탕으로 `DataSource` 를 만들어 내부 흐름을 보완한다. 
이는 트랜잭션 시작시에는 `Conneciton Proxy` 객체를 리턴해서 사용하다가, 
식제 쿼리가 발생하는 시점에 `getConnection()` 을 바탕으로 실제 커넥션을 사용하도록 한다.  

실제로 `Repository` 를 사용해서 `Master DataSource` 관련 기능을 구현하는 `ExamMasterService` 의 내용은 아래와 같다. 

```java
@Transactional
@Service
public class ExamMasterService {
    private final ExamRepository examRepository;

    public ExamMasterService(ExamRepository examRepository) {
        this.examRepository = examRepository;
    }

    public Exam save(Exam exam) {
        return this.examRepository.save(exam);
    }

    public void delete(Exam exam) {
        this.examRepository.delete(exam);
    }

    public void deleteById(long id) {
        this.examRepository.deleteById(id);
    }

    public void deleteAll() {
        this.examRepository.deleteAll();
    }
}
```  

`@Transaction` 을 선언했고, 메소드는 `save`, `delete` 와 같은 `Master` 관련 동작만 구현돼있다. 
`readOnly` 를 설정하지 않는 방법으로 `Master DataSource` 에 연결 될 수 있도록 한다. 

다음으로 `Slave DataSource` 관련 기능을 구현하는 `ExamSlaveService` 의 내용은 아래와 같다.

```java
@Transactional(readOnly = true)
@Service
public class ExamSlaveService {
    public final ExamRepository examRepository;

    public ExamSlaveService(ExamRepository examRepository) {
        this.examRepository = examRepository;
    }

    public Exam readById(long id) {
        return this.examRepository.findById(id).orElse(null);
    }

    public List<Exam> readAll() {
        return this.examRepository.findAll();
    }
}
```  

`ExamMasterService` 와는 달리 `@Transaction` 에서 `readOnly` 필드에 `true` 값을 설정했다. 
이렇게 `readOnly` 를 설정 하는 방법으로 `Slave DataSource` 에 연결 될 수 있도록 한다. 
메소드는 `Slave` 동작인 `read` 관련 동작만 구현돼있다.  

### 테스트
아래 테스트 클래스를 바탕으로 해당 `Service` 클래스가 정상적으로 동작함을 확인 할 수 있다. 

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ExamMasterServiceTest {
    @Autowired
    private ExamMasterService examMasterService;

    @Test
    public void save_Master() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();

        // when
        this.examMasterService.save(exam);

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
    }

    @Test
    public void delete_Master() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam);

        // when
        this.examMasterService.deleteById(exam.getId());

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ExamSlaveServiceTest {
    @Autowired
    private ExamSlaveService examSlaveService;
    @Autowired
    private ExamRepository examRepository;

    @Before
    public void setUp() throws Exception{
        this.examRepository.deleteAll();
    }

    @Test
    public void readById_Slave() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examRepository.save(exam);

        // when
        Exam result = this.examSlaveService.readById(exam.getId());

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));
        assertThat(result, notNullValue());
    }

    @Test
    public void readAll_Slave() {
        // given
        Exam exam1 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examRepository.save(exam1);
        Exam exam2 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examRepository.save(exam2);

        // when
        List<Exam> result = this.examSlaveService.readAll();

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));
        assertThat(result, hasSize(2));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MultipleDataSourceTest {
    @Autowired
    private ExamMasterService examMasterService;
    @Autowired
    private ExamSlaveService examSlaveService;
    @Autowired
    private ExamRepository examRepository;

    @Before
    public void setUp() {
        this.examRepository.deleteAll();
    }

    @Test
    public void insert_select_delete() {
        EnumDB actual;

        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        this.examSlaveService.readAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.examMasterService.delete(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
    }

    @Test
    public void insert_insert_select_select_delete_delete() {
        EnumDB actual;

        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        Exam exam2 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam2);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        this.examSlaveService.readById(exam.getId());
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.examSlaveService.readById(exam2.getId());
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.examMasterService.delete(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        this.examMasterService.delete(exam2);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
    }
}
```  

`MultipleDataSourceTest.insert_select_delete()` 의 테스트를 수행하며 발생한 로그는 아래와 같다. 

```
2020-08-02 01:56:17.962 DEBUG 21388 --- [           main] org.hibernate.SQL                        : insert into exam (datetime, value) values (?, ?)
2020-08-02 01:56:17.962 DEBUG 21388 --- [           main] c.w.t.config.RoutingDataSource           : MASTER_DB_1
2020-08-02 01:56:18.109 DEBUG 21388 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Setting JDBC Connection [Lazy Connection proxy for target DataSource [com.windowforsun.transaction.config.RoutingDataSource@6f667ad1]] read-only
2020-08-02 01:56:18.116 DEBUG 21388 --- [           main] org.hibernate.SQL                        : select exam0_.id as id1_0_, exam0_.datetime as datetime2_0_, exam0_.value as value3_0_ from exam exam0_
2020-08-02 01:56:18.116 DEBUG 21388 --- [           main] c.w.t.config.RoutingDataSource           : SLAVE_DB_1
2020-08-02 01:56:18.228  INFO 21388 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Starting...
2020-08-02 01:56:18.270  INFO 21388 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Start completed.
2020-08-02 01:56:18.283 DEBUG 21388 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Resetting read-only flag of JDBC Connection [HikariProxyConnection@2099033503 wrapping com.mysql.cj.jdbc.ConnectionImpl@16ccd2bc]
2020-08-02 01:56:18.287 DEBUG 21388 --- [           main] org.hibernate.SQL                        : select exam0_.id as id1_0_0_, exam0_.datetime as datetime2_0_0_, exam0_.value as value3_0_0_ from exam exam0_ where exam0_.id=?
2020-08-02 01:56:18.287 DEBUG 21388 --- [           main] c.w.t.config.RoutingDataSource           : MASTER_DB_1
2020-08-02 01:56:18.420 DEBUG 21388 --- [           main] org.hibernate.SQL                        : delete from exam where id=?
```  

로그를 확인하면 `Master DataSource` 를 사용할 때는 `Setting JDBC Connection ... read-only` 로그가 발생하지 않지만, 
`Slave DataSource` 를 사용할 때는 해당 로그가 발생하는 것을 확인 할 수 있다. 

---
## Reference
[Use replica database for read-only transactions](https://blog.pchudzik.com/201911/read-from-replica/)  