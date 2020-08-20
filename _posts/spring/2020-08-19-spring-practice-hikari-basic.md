--- 
layout: single
classes: wide
title: "[Spring 실습] HikariCP 기본 개념과 설정"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '경량 JDBC 커넥션 풀인 HikariCP 와 구성법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - HikariCP
    - MySQL
    - JPA
toc: true
use_math: true
---  

## HikariCP
`HikariCP` 는 경량 `JDBC` 커넥션 풀 프레임워크로, 2012년도 처음으로 발표 되었다. 
경랑 커넥션 풀인 만큼 기존에 많이 사용되던 `c3po`, `dbcp2`, `tomcat`, `vibur` 보다 뛰어난 성능을 보여준다고 한다. 
아래는 [HikariCP-benchmark](https://github.com/brettwooldridge/HikariCP-benchmark) 
에서 확인 할 수 있는 벤치마크 결과이다. 

![그림1](({{site.baseurl}}/img/spring/practice-hikaricp-basic-1.png))

이러한 성능의 결과인지 `Spring Boot 2.0` 부터 기본 커넥션 풀로 채용 되었다. 
본 포스트에서는 `Spring Boot` 프로젝트에서 `HikariCP` 를 설정하고 사용하는 기본적인 부분에 대해 알아보도록 한다. 

## 의존성
`Maven`, `Gradle` 에서 사용할 수 있는 의존성은 
[여기](https://mvnrepository.com/artifact/com.zaxxer/HikariCP)
에서 확인 할 수 있다.  

`Spring Boot` 가 아니거나 버전이 낮다면 위 링크를 통해 의존성을 추가하고, 
`Spring Boot 2.0` 버전대라면 아래 의존성에 `HikariCP` 의존성이 포함되어 있어 별도로 추가없이 사용할 수 있다. 

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
```  

```yaml
compile group: 'org.springframework.boot', name: 'spring-boot-starter-data-jdbc'
```  

테스트 프로젝트에서 사용한 빌드 툴은 `Gradle` 이고 `build.gradle` 내용은 아래와 같다. 

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.10.BUILD-SNAPSHOT'
    id 'io.spring.dependency-management' version '1.0.10.RELEASE'
    id 'java'
}

allprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'
    apply plugin: 'org.springframework.boot'

    group = 'com.windowforsun'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '1.8'

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }

    repositories {
        mavenCentral()
        maven { url 'https://repo.spring.io/milestone' }
        maven { url 'https://repo.spring.io/snapshot' }
    }

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jdbc'
        compileOnly 'org.projectlombok:lombok'
        runtimeOnly 'mysql:mysql-connector-java'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
        testImplementation group: 'org.hamcrest', name: 'hamcrest-all', version: '1.3'
    }

    test {
        useJUnitPlatform()
    }
}
```  

## 테스트 Database
테스트 용도로 사용할 `Docker` 기반 `MySQL` 템플릿인 `docker-compose.yaml` 파일 내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  mysql:
    image: mysql:8
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - 22277:3306
    volumes:
      - ./init/:/docker-entrypoint-initdb.d/
```  

`root` 계정의 비밀번호는 `root` 로 설정하고, 서비스 포트인 `3306`은 외부 `22277` 로 포워딩한다. 
그리고 초기 `DB` 및 테이블을 설정하는 `init/init.sql` 파일을 볼륨으로 마운트한다.  

`init/init.sql` 파일 내용은 아래와 같다. 

```sql
create database test1;

use test1;

CREATE TABLE `exam1` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value1` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

insert into exam1(value1) values('a');
insert into exam1(value1) values('b');


create database test2;
use test2

CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```  

`test1`, `test2` 데이터베이스를 생성하고 각각 `exam1`, `exam2` 테이블을 생성한다. 
그리고 `test1.exam1` 테이블에만 먼저 2개 `row` 를 추가해 준다.  

서비스는 `docker-compose.yaml up --build` 또는 `docker-compose.yaml up` 명령으로 실행 할 수 있다. 

## 기본 DataSource 설정
`DataSource` 가 하나만 필요하다면 `application.yaml` 에서 기본 `DataSource` 설정 형식으로, 
간단하게 `HikariCP` 를 설정할 수 있다. 테스트 `MySQL` 에 연결하는 설정이 작성된 `application.yaml` 파일 내용은 아래와 같다. 

```yaml
spring:
  # datasource 설정 정보
  datasource:
    url: jdbc:mysql://localhost:22277/test1
    username: root
    password: root

logging:
  level:
    # hikaricp 관련 로그 추가
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari: TRACE
```  

- `.spring.datasource.url` : `test1` 데이터베이스에 접속하는 `jdbc` 의 `URL` 정보이다. 
- `.spring.datasource.username` : `MySQL` 의 계정명을 설정한다.
- `.spring.datasource.password` : `MySQL` 의 계정 비밀번호를 설정한다.
- `.logging.level.com.zaxxer.hikari.HikariConfig` : 해당 클래스의 `DEBUG` 로그를 출력한다.
- `.logging.level.com.zaxxer.hikari` : 해당 패키지의 `TRACE` 로그를 출력한다.

별도의 구현클래스는 존재하지 않는다. 
바로 위 설정을 테스트하는 테스트 코드의 내용은 아래와 같다. 

```java
@SpringBootTest
public class SimpleHikariCpTest {
    @Autowired
    private DataSource dataSource;
    private Connection con;
    private PreparedStatement psmt;

    @BeforeEach
    public void setUp() throws Exception {
        this.con = this.dataSource.getConnection();
    }

    @AfterEach
    public void tearDown() throws Exception {
        this.psmt.close();
        this.con.close();
    }

    @Test
    public void mySqlServerId() throws Exception {
        // given
        this.psmt = con.prepareStatement("show variables like 'server_id'");

        // when
        ResultSet rs = this.psmt.executeQuery();
        rs.next();

        // then
        int actual = rs.getInt(2);
        assertThat(actual, is(1));
    }

    @Test
    public void exam1TableCount() throws Exception {
        // given
        this.psmt = this.con.prepareStatement("select count(1) from exam1");

        // when
        ResultSet rs = this.psmt.executeQuery();
        rs.next();

        // then
        int actual = rs.getInt(1);
        assertThat(actual, is(2));
    }
}
```  

테스트를 수행하면 정상적으로 `test1` 데이터베이스의 아이디를 가져오고, `test1.exam1` 테이블에 대한 정보를 가져오는 것을 확인할 수 있다. 
그리고 출력되는 로그중 아래와 같이 `HikariCP` 관련 로그 내용도 확인 가능하다. 
해당 내용은 `application.yaml` 에서 설정한 부분에 대한 로그 출력이다. 

```
.. Hikari 설정 정보 로그 ..
2020-08-19 20:54:15.430 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-1 - configuration:
2020-08-19 20:54:15.433 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : allowPoolSuspension.............false
2020-08-19 20:54:15.434 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : autoCommit......................true
2020-08-19 20:54:15.434 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : catalog.........................none
2020-08-19 20:54:15.434 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionInitSql...............none
2020-08-19 20:54:15.435 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionTestQuery.............none
2020-08-19 20:54:15.435 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionTimeout...............30000
2020-08-19 20:54:15.435 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : dataSource......................none
2020-08-19 20:54:15.437 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : dataSourceClassName.............none
2020-08-19 20:54:15.437 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : dataSourceJNDI..................none
2020-08-19 20:54:15.438 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : dataSourceProperties............{password=<masked>}
2020-08-19 20:54:15.438 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : driverClassName................."com.mysql.cj.jdbc.Driver"
2020-08-19 20:54:15.438 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : exceptionOverrideClassName......none
2020-08-19 20:54:15.438 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : healthCheckProperties...........{}
2020-08-19 20:54:15.439 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : healthCheckRegistry.............none
2020-08-19 20:54:15.439 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : idleTimeout.....................600000
2020-08-19 20:54:15.439 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : initializationFailTimeout.......1
2020-08-19 20:54:15.439 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : isolateInternalQueries..........false
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : jdbcUrl.........................jdbc:mysql://localhost:22277/test1
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : leakDetectionThreshold..........0
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : maxLifetime.....................1800000
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : maximumPoolSize.................10
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : metricRegistry..................none
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : metricsTrackerFactory...........none
2020-08-19 20:54:15.440 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : minimumIdle.....................10
2020-08-19 20:54:15.441 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : password........................<masked>
2020-08-19 20:54:15.441 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : poolName........................"HikariPool-1"
2020-08-19 20:54:15.441 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : readOnly........................false
2020-08-19 20:54:15.441 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : registerMbeans..................false
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : scheduledExecutor...............none
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : schema..........................none
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : threadFactory...................internal
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : transactionIsolation............default
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : username........................"root"
2020-08-19 20:54:15.442 DEBUG 46772 --- [           main] com.zaxxer.hikari.HikariConfig           : validationTimeout...............5000

.. Hikari 커넥션 풀 로그 ..
2020-08-19 20:54:24.132 DEBUG 46772 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@459cfcca
2020-08-19 20:54:24.250 DEBUG 46772 --- [l-1 housekeeper] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Pool stats (total=1, active=0, idle=1, waiting=0)
2020-08-19 20:54:24.346 DEBUG 46772 --- [extShutdownHook] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Before shutdown stats (total=1, active=0, idle=1, waiting=0)
2020-08-19 20:54:24.351 DEBUG 46772 --- [nnection closer] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Closing connection com.mysql.cj.jdbc.ConnectionImpl@459cfcca: (connection evicted)
2020-08-19 20:54:24.421 DEBUG 46772 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@521dec92
2020-08-19 20:54:24.422 DEBUG 46772 --- [nnection closer] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Closing connection com.mysql.cj.jdbc.ConnectionImpl@521dec92: (connection evicted)
2020-08-19 20:54:24.424 DEBUG 46772 --- [extShutdownHook] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - After shutdown stats (total=0, active=0, idle=0, waiting=0)
```  

위 `HikariConfig` 로그를 보면 기본 옵션 설정 값에 대한 정보를 확인해 볼 수 있다.  

기본 `DataSource` 를 사용해서 `HikariCP` 를 설정할 때, 옵션에 대한 설정은 아래와 같이 `application.yaml` 에 작성 할 수 있다. 

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:22277/test1
    username: root
    password: root
    hikari:
      maximumPoolSize: 2
      connectionTimeout: 10000

.. 생략 ..
```  

최대 커넥션 풀 크기를 2로 설정하고 커넥션 풀에 대한 타임아웃을 10초로 설정했다. 
이 설정으로 다시 위 테스트 코드를 수행하고 로그를 확인하면 아래와 같이 설정내용이 적용된 것을 확인 할 수 있다. 

```
2020-08-19 20:59:25.356 DEBUG 54876 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-1 - configuration:

.. 생략 ..

2020-08-19 20:59:25.364 DEBUG 54876 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionTimeout...............10000

.. 생략 ..

2020-08-19 20:59:25.369 DEBUG 54876 --- [           main] com.zaxxer.hikari.HikariConfig           : maximumPoolSize.................2

.. 생략 ..
```  

## 다중 DataSource 설정
애플리케이션 하나에서 여러 `Database` 를 사용하는 경우 여러개의 `DataSource` 설정이 필요하다. 
앞서 살펴본 기본 `DataSource` 설정과 어느정도 비슷한 부분이 있지만, 
다른 설정내용도 존재한다.  

`test1` 데이터베이스, `test2` 데이터베이스정보를 설정한 `application.yaml` 내용은 아래와 같다. 
기본 `DataSource` 를 사용해서 설정할 때와 가장 크게 다른 점은 `url` 의 키값을 사용하지 않고, `jdbc-url` 을 키값을 사용한다는 점이다. 

```yaml
spring:
  datasource:
    test1:
      jdbc-url: jdbc:mysql://localhost:22277/test1
      username: root
      password: root

    test2:
      jdbc-url: jdbc:mysql://localhost:22277/test2
      username: root
      password: root

logging:
  level:
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari: TRACE
```  

- `.spring.datasource.test1` : `test1` 데이터베이스에 대한 설정내용을 기술한다. 
- `.spring.datasource.test2` : `test2` 데이터베이스에 대한 설정내용을 기술한다. 

`Spring Boot` 에서 제공하는 자동설정을 사용할 수 없기 때문에, 
별도로 `DataSource` 를 설정하는 설정파일을 작성해야 한다. 
각 `DataSource` 를 설정하는 `DataSourceConfig` 클래스는 아래와 같다. 

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties("spring.datasource.test1")
    @Primary
    public DataSource dataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.test2")
    public DataSource test2DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
}
```  

`@ConfigurationProperties` 어노테이션을 사용해서 각 프로퍼티 값을 구분해 설정을 수행할 수 있다.  

구성한 2개의 `DataSource` 를 테스트하는 테스트 코드는 아래와 같다. 

```java
@SpringBootTest
public class MultipleDataSourceTest {
    @Autowired
    private DataSource dataSource;
    @Autowired
    @Qualifier("test2DataSource")
    private DataSource test2DataSource;
    private Connection con;
    private PreparedStatement psmt;

    @AfterEach
    public void tearDown() throws Exception {
        if(this.con != null) {
            this.con.close();
        }
        if(this.psmt != null) {
            this.psmt.close();
        }
    }

    @Test
    public void exam1TableCount() throws Exception {
        // given
        this.con = this.dataSource.getConnection();
        this.psmt = this.con.prepareStatement("select count(1) from exam1");

        // when
        ResultSet rs = this.psmt.executeQuery();
        rs.next();

        // then
        int actual = rs.getInt(1);
        assertThat(actual, is(2));
    }

    @Test
    public void exam2TableCount() throws Exception {
        // given
        this.con = this.test2DataSource.getConnection();
        this.psmt = this.con.prepareStatement("select count(1) from exam2");

        // when
        ResultSet rs = this.psmt.executeQuery();
        rs.next();

        // then
        int actual = rs.getInt(1);
        assertThat(actual, is(0));
    }
}
```  

테스트 코드를 실행하면 분리된 2개의 `DataSource` 에서 정상적으로 테이블 조회가 되는 것을 확인 할 수 있다. 
출력 되는 로그를 확인하면, 아래와 같이 2개의 `DataSource` 에 대한 설정 로그가 출력 된다. 

```
2020-08-19 21:42:05.073 DEBUG 44400 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-1 - configuration:

.. 생략 ..

2020-08-19 21:11:05.082 DEBUG 44400 --- [           main] com.zaxxer.hikari.HikariConfig           : validationTimeout...............5000
2020-08-19 21:11:05.082  INFO 44400 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-08-19 21:11:10.648  INFO 44400 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.


2020-08-19 21:11:10.723 DEBUG 44400 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-2 - configuration:

.. 생략 ..

2020-08-19 21:11:10.728 DEBUG 44400 --- [           main] com.zaxxer.hikari.HikariConfig           : validationTimeout...............5000
2020-08-19 21:11:10.728  INFO 44400 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Starting...
2020-08-19 21:11:10.788  INFO 44400 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Start completed.

2020-08-19 21:11:10.852  INFO 44400 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Shutdown completed.
2020-08-19 21:11:10.880  INFO 44400 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
```  

구성한 각 `DataSource` 에 `HikariCP` 관련 옵션을 설정하고 싶을 경우 아래와 같은 방식으로 적용할 수 있다. 

```yaml
spring:
  datasource:
    test1:
      jdbc-url: jdbc:mysql://localhost:22277/test1
      username: root
      password: root
      maximumPoolSize: 1
      connectionTimeout: 10000
      

    test2:
      jdbc-url: jdbc:mysql://localhost:22277/test2
      username: root
      password: root
      maximumPoolSize: 2
      connectionTimeout: 20000
```  

옵션을 적용하고 다시 테스트 코드를 실행한 다음, 
로그를 확인하면 아래와 같이 옵션 값이 각 `DataSource`에 적용된 것을 확인 할 수 있다. 

```
2020-08-20 20:28:51.368 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-1 - configuration:
2020-08-20 20:28:51.371 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : allowPoolSuspension.............false

.. 생략 ..

2020-08-20 20:28:51.371 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionTimeout...............10000
2020-08-20 20:28:51.375 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : jdbcUrl.........................jdbc:mysql://localhost:22277/test1
2020-08-20 20:28:51.375 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : maximumPoolSize.................1

.. 생략 ..

2020-08-20 20:28:51.377  INFO 18120 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-08-20 20:28:57.769  INFO 18120 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.


2020-08-20 20:28:57.890 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : HikariPool-2 - configuration:

.. 생략 ..

2020-08-20 20:28:57.893 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : connectionTimeout...............2000
2020-08-20 20:28:57.895 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : jdbcUrl.........................jdbc:mysql://localhost:22277/test2
2020-08-20 20:28:57.896 DEBUG 18120 --- [           main] com.zaxxer.hikari.HikariConfig           : maximumPoolSize.................2

.. 생략 ..

2020-08-20 20:28:57.897  INFO 18120 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Starting...
2020-08-20 20:28:57.944  INFO 18120 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Start completed.
```  

## JPA + DataSource 설정
`JPA` 는 `Java Persistent API` 의 약자로 `Java ORM` 의 규격이다. 
이를 구현한 구현체로는 `Hiernetes` ,` OpenJPA` 등이 있다.  

`JPA` 가 사용하는 `DataSource` 를 `HikariCP` 로 설정하는 방법은 어렵지않다. 
기본 케넥션 풀이 `HikariCP` 이기 때문에 `DataSource` 하나만 사용하는 경우 자동 설정을 사용해서 간편하게 구성할 수 있다.  

아래는 `JPA` 에 대한 의존성으로 `build.gradle` 에 추가가 필요하다. 

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
}
```  

그리고 `application.yaml` 은 기본 `DataSource` 의 설정과 동일하다. 

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:22277/test1
    username: root
    password: root

logging:
  level:
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari: TRACE
```  

그리고 테이블과 매핑되는 엔티티 클래스와 `Repository` 클래스의 내용은 아래와 같다. 

```java
@Entity(name = "exam1")
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Exam1 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String value1;
    private LocalDateTime datetime;
}
```  

```java
@Repository
public interface Exam1Repository extends JpaRepository<Exam1, Long> {

}
```  

정상동작 여부를 확인하는 테스트 코드는 아래와 같다. 

```java
@SpringBootTest
public class JpaHikariTest {
    @Autowired
    private Exam1Repository exam1Repository;

    @Test
    public void findById() {
        // given
        long id = 1;

        // when
        Exam1 actual = this.exam1Repository.findById(id).orElse(null);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(id));
    }

    @Test
    public void findAll() {
        // when
        List<Exam1> actual = this.exam1Repository.findAll();

        // then
        assertThat(actual, hasSize(2));
    }
}
```  

## 다중 JPA + DataSource 설정
`JPA` 에서 다중 `DataSource` 를 구성하는 방법은 다양하게 존재한다. 
이전 포스트인 [여기]({{site.baseurl}}{% link _posts/spring/2020-08-06-spring-practice-jpa-multiple-datasource-aop.md %})
에서 몇가지 방법에 대해 다룬적이 있다. 
이번 예제에서는 복잡한 설정 없이 구성할 수 있는, 패키지를 사용한 방법으로 설정을 진행한다.  

`JPA` 의존성이 추가된 상태에서, 
`application.yaml` 의 내용은 다중 `DataSource` 의 설정 내용과 비슷하다. 

```yaml
spring:
  datasource:
    test1:
      jdbc-url: jdbc:mysql://localhost:22277/test1
      username: root
      password: root

    test2:
      jdbc-url: jdbc:mysql://localhost:22277/test2
      username: root
      password: root


logging:
  level:
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari: TRACE
```  

먼저 `test1` 데이터베이스에 대한 `DataSource` 를 설정하고 이를 `JPA` 관련 패키지와 연결하는 설정 내용은 아래와 같다. 

```java
@Configuration
@EnableJpaRepositories(
        basePackages = {"com.windowforsun.jpamultipledatasource.repository.test1"},
        entityManagerFactoryRef = "test1EntityManagerFactory",
        transactionManagerRef = "test1TransactionManager"
)
public class Test1DataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public Test1DataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.test1")
    public DataSource test1DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean test1EntityManagerFactory(EntityManagerFactoryBuilder builder) {
        Map<String, ?> properties = this.hibernateProperties.determineHibernateProperties(
                this.jpaProperties.getProperties(), new HibernateSettings()
        );

        return builder
                .dataSource(this.test1DataSource())
                .properties(properties)
                .packages("com.windowforsun.jpamultipledatasource.domain.test1")
                .persistenceUnit("test1")
                .build();
    }

    @Bean
    @Primary
    public PlatformTransactionManager test1TransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(this.test1EntityManagerFactory(builder).getObject()));
    }
}
```  

`@EnableJpaRepositories` 의 `basePackages` 필드에 `test1` 에 해당하는 `Repository` 패키지를 설정한다. 
그리고 구성에 필요한 `entityManagerFactoryRef` 의 빈이름, `transactionManagerRef` 빈이름을 각각 설정해 주면된다.  

`test1` 설정의 각 빈은 `@Primary` 설정으로 타입에 디해 기본 빈으로 설정해 준다. 
`@Primary` 가 없으면 `JPA` 초기화 작업 중 관련 빈을 찾지 못해 에러가 발생한다.  

`entityManagerFactory` 빈을 생성할 때, `DataSource` 에서 사용하는 엔티티 패키지를 설정해야 한다. 
그리고 생성한 `entityManagerFactory` 빈을 사용해서 `transactionManager` 빈을 생성해 주면 된다.  

동일한 방식으로 `test2` 데이터베이스에 대한 설정 내용은 아래와 같다. 

```java

@Configuration
@EnableJpaRepositories(
        basePackages = {"com.windowforsun.jpamultipledatasource.repository.test2"},
        entityManagerFactoryRef = "test2EntityManagerFactory",
        transactionManagerRef = "test2TransactionManager"
)
public class Test2DataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public Test2DataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.test2")
    public DataSource test2DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean test2EntityManagerFactory(EntityManagerFactoryBuilder builder) {
        Map<String, ?> properties = this.hibernateProperties.determineHibernateProperties(
                this.jpaProperties.getProperties(), new HibernateSettings()
        );

        return builder
                .dataSource(this.test2DataSource())
                .properties(properties)
                .packages("com.windowforsun.jpamultipledatasource.domain.test2")
                .persistenceUnit("test2")
                .build();
    }

    @Bean
    public PlatformTransactionManager test2TransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(this.test2EntityManagerFactory(builder).getObject()));
    }
}
```  

이제 각 엔티티 클래스와 `Repository` 클래스는 아래와 같다. 

```java
package com.windowforsun.jpamultipledatasource.domain.test1;

@Entity(name = "exam1")
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Exam1 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String value1;
    private LocalDateTime datetime;
}
```  

```java
package com.windowforsun.jpamultipledatasource.domain.test2;

@Entity(name = "exam2")
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Exam2 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String value2;
    private LocalDateTime datetime;
}
```  

```java
package com.windowforsun.jpamultipledatasource.repository.test1;

@Repository
public interface Exam1Repository extends JpaRepository<Exam1, Long> {
}
```  

```java
package com.windowforsun.jpamultipledatasource.repository.test2;

@Repository
public interface Exam2Repository extends JpaRepository<Exam2, Long> {
}
```  

각 `Repository` 클래스를 테스트하는 테스트 코드의 내용은 아래와 같다. 

```java
@SpringBootTest
public class Test1JpaDataSourceTest {
    @Autowired
    private Exam1Repository exam1Repository;

    @Test
    public void findAll() {
        // when
        List<Exam1> actual = this.exam1Repository.findAll();

        // then
        assertThat(actual, hasSize(2));
    }

    @Test
    public void findById() {
        // given
        long id = 1;

        // when
        Exam1 actual = this.exam1Repository.findById(id).orElse(null);

        // then
        assertThat(actual, notNullValue());
        assertThat(actual.getId(), is(id));
    }
}
```  

```java
@SpringBootTest
public class Test2JpaDataSourceTest {
    @Autowired
    private Exam2Repository exam2Repository;

    @AfterEach
    public void tearDown() {
        this.exam2Repository.deleteAll();
    }

    @Test
    public void findAll() {
        // when
        List<Exam2> actual = this.exam2Repository.findAll();

        // then
        assertThat(actual, hasSize(0));
    }

    @Test
    public void findById() {
        // given
        long id = 1;

        // when
        Exam2 actual = this.exam2Repository.findById(id).orElse(null);

        // then
        assertThat(actual, nullValue());
    }

    @Test
    public void save() {
        // given
        Exam2 exam2 = Exam2.builder()
                .value2("aa")
                .build();

        // when
        this.exam2Repository.save(exam2);

        // then
        assertThat(exam2.getId(), greaterThan(0l));
    }
}
```  

---
## Reference
[brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)  
[Introduction to HikariCP](https://www.baeldung.com/hikaricp)  
