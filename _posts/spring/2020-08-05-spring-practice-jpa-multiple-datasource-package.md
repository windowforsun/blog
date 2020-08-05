--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Multiple DataSource(Database) Package"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Replication, Sharding 에서 필요한 JPA 다중 Datasource 를 Package 을 활용해서 구현해 보자'
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
    - Package
toc: true
use_math: true
---  

## JPA Multiple DataSource
[여기]({{site.baseurl}}{% link _posts/spring/2020-08-01-spring-practice-jpa-multiple-datasource-transaction.md %})
포스트에서는 `Transaction` 을 사용해서 `Replication` 구성에서 다중 `DataSource` 를 활용하는 방법에 대해 알아봤다. 
이번 글에서는 미리 소개한 3가지 방법 중 나머지인 패키지를 사용해서 이를 활용하는 방법에 대해 알아본다. 

## Package 방법
`Package` 를 사용하는 방법은 `JPA` 관련 설정에 `DataSource` 와 해당하는 `JpaRepository` 패키지를 묶어 구성하는 방법을 사용한다. 
`Master` 에 해당하는 `DataSource` 관련 설정 파일에 `Master` 연산을 수행하는 `JpaRepository` 패키지만 따로 분리해 구현 한후 해당 패키지를 등록하고, 
`Slave` 또한 `DataSource` 관련 설정 파일에 `Slave` 연산을 수행하는 `JpaRepository` 패키지를 등록해서 `DataSource` 가 분리 될 수 있도록 한다.  

데이터 베이스는 `Master1`, `Slave1` 만 사용한다.  

디렉토리 구조는 아래와 같다. 

<!-- tree transaction -I "build" -->
```
package
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── mpackage
    │   │               ├── PackageApplication.java
    │   │               ├── config
    │   │               │   ├── MasterDataSourceConfig.java
    │   │               │   └── SlaveDataSourceConfig.java
    │   │               ├── constant
    │   │               │   └── EnumDB.java
    │   │               ├── domain
    │   │               │   └── Exam.java
    │   │               ├── repository
    │   │               │   ├── master
    │   │               │   │   └── ExamMasterRepository.java
    │   │               │   └── slave
    │   │               │       └── ExamSlaveRepository.java
    │   │               └── service
    │   │                   ├── ExamMasterService.java
    │   │                   └── ExamSlaveService.java
    │   └── resources
    │       └── application.yaml
    └── test
        └── java
            └── com
                └── windowforsun
                    └── mpackage
                        └── service
                            ├── ExamMasterServiceTest.java
                            ├── ExamSlaveServiceTest.java
                            └── MultipleDataSourceTest.java
```  

우선 별도로 설명이 필요하지 않는 클라스 내용은 아래와 같다. 

```java
@SpringBootApplication
public class PackageApplication {
    public static void main(String[] args) {
        SpringApplication.run(PackageApplication.class, args);
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

    @PreUpdate
    @PrePersist
    public void preUpdate() throws Exception{
        Thread.sleep(150);
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

이전 `Transaction` 을 사용하는 방법에서는 여러개 `DataSource` 를 만들고, 
`AbstractRoutingDataSource` 구현과 `RoutingDataSource` 를 사용해서 이를 적절하게 라우팅하는 방법을 사용했다. 
`Package` 를 사용하는 방법은 특정 패키지에 해당하는 `Repository` 들이 설정된 `DataSource` 를 사용하도록 매핑 시키는 방법을 사용한다.  

프로젝트 구조를 보면 알 수 있듯이, 
`Repository` 관련 패키지는 `com.windowforsun.mpackage.repository.master` 와 `com.windowforsun.mpackage.repository.slave` 두가지 종류가 있다. 
아래 설정 파일에서는 이를 각각 다른 설정 파일을 사용해서 `DataSource` 설정을 수행한다. 

```java
@Configuration
@EnableJpaRepositories(basePackages = {"com.windowforsun.mpackage.repository.master"})
public class MasterDataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public MasterDataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.masterdb1")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder builder) {
        Map<String, ?> properties = this.hibernateProperties.determineHibernateProperties(
                this.jpaProperties.getProperties(), new HibernateSettings()
        );

        return builder.dataSource(masterDataSource())
                .properties(properties)
                .packages("com.windowforsun.mpackage.domain")
                .persistenceUnit(EnumDB.MASTER_DB_1.name())
                .build();
    }

    @Bean
    @Primary
    PlatformTransactionManager transactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(entityManagerFactory(builder).getObject()));
    }
}
```  

- `@EnableJpaRepositories` 의 `basePackages` 필드에 `com.windowforsun.mpackage.repository.master` 를 설정해, 
해당 패키지에 있는 `Repository` 클래스에 대한 설정을 수행한다.  

- `com.windowforsun.mpackage.repository.master` 패키지에 해당하는 `DataSource` 설정으로 
`Master` 데이터베이스를 사용하는 `DataSource` 를 설정한다.  

- `masterDataSource` 빈은 `spring.datasource.masterdb1` 에 해당하는 설정 내용을 가져와 `DataSource` 빈을 생성하고, 
해당 빈을 기본 `DataSource` 빈으로 설정했다.  

- `entityManagerFactory` 빈은 `masterDataSource` 빈을 `EntityManagerFactoryBuilder` 를 사용해서 `com.windowforsun.mpakcage.domain` 에 해당하는 도메인 객체를 사용하도록 설정한다.  

- `transactionManager` 빈에서는 `entityManagerFactory` 빈으로 `JpaTransactionManager` 빈을 생성하고 해서, 
기본 `TransactionManager` 빈으로 설정한다.  

```java
@Configuration
@EnableJpaRepositories(
        basePackages = {"com.windowforsun.mpackage.repository.slave"},
        entityManagerFactoryRef = "slaveEntityManagerFactory",
        transactionManagerRef = "slaveTransactionManager"
)
public class SlaveDataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public SlaveDataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slavedb1")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean slaveEntityManagerFactory(EntityManagerFactoryBuilder builder) {
        Map<String, ?> properties = this.hibernateProperties.determineHibernateProperties(
                this.jpaProperties.getProperties(), new HibernateSettings()
        );

        return builder.dataSource(slaveDataSource())
                .properties(properties)
                .packages("com.windowforsun.mpackage.domain")
                .persistenceUnit(EnumDB.SLAVE_DB_1.name())
                .build();
    }

    @Bean
    PlatformTransactionManager slaveTransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(slaveEntityManagerFactory(builder).getObject()));
    }
}
```  

- `@EnableJpaRepositories` 의 `basePakcages` 필드에 `com.windowforsun.mpackage.repository.slave` 패키지를 설정해, 
해당 패키지에 있는 `Repository` 클래스에 대한 설정을 수행한다. 
그리고 `entityManagerFactoryRef`, `transactionManagerRef` 필드에 해당 클래스에서 생성하는 빈의 참조를 설정해서, 
기본 빈을 참조하지 않도록 한다. 

두 설정파일에서 사용하는 도메인 패키지가 `com.windowforsun.mpakcage.domain` 으로 동일한 것을 확인 할 수 있다. 
만약 두 `DataSource` 에서 사용하는 도메인이 다르다면 다른 패키지로 분리할 수도 있다.  

이제 `com.windowforsun.mpackage.repository` 패키지 하위에 `master`, `slave` 패키지를 만들고 관련 `Repository` 클래스를 구현하면 아래와 같다. 

```java
@Repository
public interface ExamMasterRepository extends JpaRepository<Exam, Long> {
}
```  

- `Master` 의 데이터베이스를 사용하는 `Repository` 클래스로 `com.windowforsun.mpackage.repository.master` 패키지에 포함된다. 

```java
@Repository
public interface ExamSlaveRepository extends JpaRepository<Exam, Long> {
}
```  

- `Slave` 의 데이터베이스를 사용하는 `Repository` 클래스로 `com.windowforsun.mpackage.repository.slave` 패키지에 포함된다. 

각 `Repository` 클래스를 사용하는 `Server` 클래스의 구현 내용은 아래와 같다. 

```java
@Service
public class ExamMasterService {
    private final ExamMasterRepository examMasterRepository;

    public ExamMasterService(ExamMasterRepository examMasterRepository) {
        this.examMasterRepository = examMasterRepository;
    }

    public Exam save(Exam exam) {
        return this.examMasterRepository.save(exam);
    }

    public void delete(Exam exam) {
        this.examMasterRepository.delete(exam);
    }

    public void deleteById(long id) {
        this.examMasterRepository.deleteById(id);
    }

    public void deleteAll() {
        this.examMasterRepository.deleteAll();
    }
}
```  

```java
@Service
public class ExamSlaveService {
    private final ExamSlaveRepository examSlaveRepository;

    public ExamSlaveService(ExamSlaveRepository examSlaveRepository) {
        this.examSlaveRepository = examSlaveRepository;
    }

    public Exam readById(long id) {
        return this.examSlaveRepository.findById(id).orElse(null);
    }

    public List<Exam> readAll() {
        return this.examSlaveRepository.findAll();
    }
}
```  

### 테스트

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ExamMasterServiceTest {
    @Autowired
    private ExamMasterService examMasterService;
    @Autowired
    private ExamMasterRepository examMasterRepository;

    @Before
    public void setUp() {
        this.examMasterRepository.deleteAll();
    }

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
        assertThat(exam.getId(), greaterThan(0l));
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
        this.examMasterService.delete(exam);

        // then
        Exam actual = this.examMasterRepository.findById(exam.getId()).orElse(null);
        assertThat(actual, nullValue());
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
    private ExamMasterService examMasterService;

    @Before
    public void setUp() {
        this.examMasterService.deleteAll();
    }

    @Test
    public void readById_Slave() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam);

        // when
        Exam result = this.examSlaveService.readById(exam.getId());

        // then
        assertThat(result.getId(), is(exam.getId()));
    }

    @Test
    public void readAll_Slave() {
        // given
        Exam exam1 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam1);
        Exam exam2 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam2);

        // when
        List<Exam> result = this.examSlaveService.readAll();

        // then
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
    private ExamMasterRepository examMasterRepository;

    @Before
    public void setUp() {
        this.examMasterRepository.deleteAll();
    }

    @Test
    public void insert_select_delete() {
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam);
        assertThat(exam.getId(), greaterThan(0l));

        List<Exam> list = this.examSlaveService.readAll();
        assertThat(list, hasSize(1));

        this.examMasterService.delete(exam);
        boolean isExists = this.examMasterRepository.existsById(exam.getId());
        assertThat(isExists, is(false));
    }

    @Test
    public void insert_insert_select_select_delete_delete() {
        Exam exam1 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam1);
        assertThat(exam1.getId(), greaterThan(0l));

        Exam exam2 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examMasterService.save(exam2);
        assertThat(exam2.getId(), greaterThan(0l));

        Exam result1 = this.examSlaveService.readById(exam1.getId());
        assertThat(result1, notNullValue());
        assertThat(result1.getId(), is(exam1.getId()));

        Exam result2 = this.examSlaveService.readById(exam2.getId());
        assertThat(result2, notNullValue());
        assertThat(result2.getId(), is(exam2.getId()));

        this.examMasterService.delete(exam1);
        boolean isExists1 = this.examMasterRepository.existsById(exam1.getId());
        assertThat(isExists1, is(false));

        this.examMasterService.delete(exam2);
        boolean isExists2 = this.examMasterRepository.existsById(exam2.getId());
        assertThat(isExists2, is(false));
    }
}
```  

`MultipleDataSourceTest.insert_select_delete()` 의 테스트를 수행하며 발생한 로그는 아래와 같다. 

```
2020-08-06 23:25:02.517 DEBUG 10384 --- [           main] org.hibernate.SQL                        : insert into exam (datetime, value) values (?, ?)
2020-08-06 23:25:02.657 DEBUG 10384 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Setting JDBC Connection [HikariProxyConnection@89144445 wrapping com.mysql.cj.jdbc.ConnectionImpl@2fd4312a] read-only
2020-08-06 23:25:02.662 DEBUG 10384 --- [           main] org.hibernate.SQL                        : select exam0_.id as id1_0_, exam0_.datetime as datetime2_0_, exam0_.value as value3_0_ from exam exam0_
2020-08-06 23:25:02.670 DEBUG 10384 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Resetting read-only flag of JDBC Connection [HikariProxyConnection@89144445 wrapping com.mysql.cj.jdbc.ConnectionImpl@2fd4312a]
2020-08-06 23:25:02.675 DEBUG 10384 --- [           main] org.hibernate.SQL                        : select exam0_.id as id1_0_0_, exam0_.datetime as datetime2_0_0_, exam0_.value as value3_0_0_ from exam exam0_ where exam0_.id=?
2020-08-06 23:25:02.680 DEBUG 10384 --- [           main] org.hibernate.SQL                        : delete from exam where id=?
2020-08-06 23:25:02.705 DEBUG 10384 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Setting JDBC Connection [HikariProxyConnection@435460010 wrapping com.mysql.cj.jdbc.ConnectionImpl@a99c42c] read-only
2020-08-06 23:25:02.733 DEBUG 10384 --- [           main] org.hibernate.SQL                        : select count(*) as col_0_0_ from exam exam0_ where exam0_.id=?
2020-08-06 23:25:02.750 DEBUG 10384 --- [           main] o.s.jdbc.datasource.DataSourceUtils      : Resetting read-only flag of JDBC Connection [HikariProxyConnection@435460010 wrapping com.mysql.cj.jdbc.ConnectionImpl@a99c42c]
```  

`Slave DataSource` 를 사용하는 `select` 연산이 발생하기 전에 `Setting JDBC Connection ... read-only` 로그가 발생하는 것을 확인 할 수 있다. 


---
## Reference
[Use replica database for read-only transactions](https://blog.pchudzik.com/201911/read-from-replica/)  