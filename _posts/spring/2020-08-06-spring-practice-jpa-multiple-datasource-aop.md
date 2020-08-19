--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Multiple DataSource(Database) AOP"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Replication, Sharding 에서 필요한 JPA 다중 Datasource 를 AOP 을 활용해서 구현해 보자'
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
    - AOP
toc: true
use_math: true
---  

## JPA Multiple DataSource
[여기]({{site.baseurl}}{% link _posts/spring/2020-08-01-spring-practice-jpa-multiple-datasource-transaction.md %})
에서는 `Transaction`, 
[여기]({{site.baseurl}}{% link _posts/spring/2020-08-05-spring-practice-jpa-multiple-datasource-package.md %})
에서는 `Package` 를 사용해서 `Replication` 구성에서 다중 `DataSource` 를 활용하는 방법에 대해 알아봤다. 
이번 글에서는 미리 소개한 3가지 방법 중 나머지인 `AOP` 를 사용해서 이를 활용하는 방법에 대해 알아본다. 

## AOP 방법
이전에 설명했었던 `Transaction` 와 `Package` 를 사용하는 방법은 기존 `Spring` 의 큰 흐름에서 여러 `DataSource` 를 사용할 수 있는 구조를 만드는 방법이였다. 
이를 위해 `Transaction` 은 `readOnly` 필드를 활용했고, `Package` 는 `DataSource` 별 사용하는 패키지를 명시하는 방법을 사용했다.  

`AOP` 는 기존 방법들과는 달리 `AOP` 를 통해 보다 사용자의 커스텀한 설정이 가능한 방법이다. 
데이터베이스는 `Master` 1, 2와 `Slave` 1, 2 모두 사용한다. 
프로젝트 디렉토리 구조는 아래와 같다.  

<!-- tree aop -I "build" -->
```
aop
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── windowforsun
    │   │           └── aop
    │   │               ├── AopApplication.java
    │   │               ├── aop
    │   │               │   ├── SetDataSource.java
    │   │               │   └── SetDataSourceAspect.java
    │   │               ├── compoment
    │   │               │   └── RoutingDatasourceManager.java
    │   │               ├── config
    │   │               │   ├── DataSourceConfig.java
    │   │               │   └── RoutingDataSource.java
    │   │               ├── constant
    │   │               │   └── EnumDB.java
    │   │               ├── domain
    │   │               │   ├── Exam.java
    │   │               │   └── Exam2.java
    │   │               ├── repository
    │   │               │   ├── Exam2Repository.java
    │   │               │   └── ExamRepository.java
    │   │               └── service
    │   │                   ├── Exam2MasterService.java
    │   │                   ├── Exam2SlaveService.java
    │   │                   └── ExamService.java
    │   └── resources
    │       └── application.yaml
    └── test
        └── java
            └── com
                └── windowforsun
                    └── aop
                        └── service
                            ├── Exam2MasterServiceTest.java
                            ├── Exam2SlaveServiceTest.java
                            ├── ExamServiceTest.java
                            └── MultipleDataSourceTest.java
```  

우선 별도 설명을 하지 않는 여타 클래스는 아래와 같다. 

```java
@SpringBootApplication
@EnableAspectJAutoProxy
public class AopApplication {
    public static void main(String[] args) {
        SpringApplication.run(AopApplication.class, args);
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
@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table(
        name = "exam2"
)
public class Exam2 {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private long id;
    @Column
    private String value2;
    @Column
    private LocalDateTime datetime;

}
```  

```java
@Repository
public interface ExamRepository extends JpaRepository<Exam, Long> {
}
```  

```java
@Repository
public interface Exam2Repository extends JpaRepository<Exam2, Long> {
}
```  

`DataSource` 설정 관련 클래스의 내용은 대부분 `Transaction` 방법과 동일한 구성이 많다. 

```java
@EnableAspectJAutoProxy
@Configuration
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
@EnableJpaRepositories(basePackages = {"com.windowforsun.aop"})
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.masterdb1")
    public DataSource master1DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slavedb1")
    public DataSource slave1DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.masterdb2")
    public DataSource master2DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slavedb2")
    public DataSource slave2DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public DataSource routingDataSource(
            @Qualifier("master1DataSource") DataSource masterDB1DataSource,
            @Qualifier("slave1DataSource") DataSource slaveDB1DataSource,
            @Qualifier("master2DataSource") DataSource masterDB2DataSource,
            @Qualifier("slave2DataSource") DataSource slaveDB2DataSource
    ) {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        HashMap<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(EnumDB.MASTER_DB_1, masterDB1DataSource);
        dataSourceMap.put(EnumDB.SLAVE_DB_1, slaveDB1DataSource);
        dataSourceMap.put(EnumDB.MASTER_DB_2, masterDB2DataSource);
        dataSourceMap.put(EnumDB.SLAVE_DB_2, slaveDB2DataSource);

        routingDataSource.setTargetDataSources(dataSourceMap
                .entrySet()
                .stream()
                .collect(toMap(Map.Entry::getKey, Map.Entry::getValue)));
        routingDataSource.setDefaultTargetDataSource(dataSourceMap.get(EnumDB.MASTER_DB_1));

        return routingDataSource;
    }


    @Primary
    @Bean
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}
```  

- `@EnableJpaAutoConfiguration` 의 `exclude` 필드에 `DataSourceAutoConfiguration` 를 설정해서 사이클로 의존성을 가지는 현상을 회피한다. 
- `@EnableJpaRepositories` 에 `com.windowforsun.aop` 를 기본 패키지로 설정한다. 
- 프로퍼티에 있는 정보를 읽어 총 4개(`master1DataSource`, `master2Datasource`, `slave1DataSource`, `slave2DataSource`) `DataSource` 를 각각 생성한다. 
- `routingDataSource` 에서는 생성된 4개의 `DataSource` 빈을 `RoutingDataSource` 에 `Enum` 키와 함께 설정한다. 
기본 값은 `master1DataSource` 로 설정한다. 
- `dataSource` 빈은 `routingDataSource` 빈을 사용해서 `LazyConnectionDataSourceProxy` 를 통해 기본 `DataSource` 를 생성한다. 
관련 좀 더 자세한 설명은 `Transaction` 방식에서 확인 할 수 있다. 

생성된 `DataSource` 빈을 라우팅 해주는 클래스의 내용은 아래와 같다. 

```java
@Slf4j
public class RoutingDataSource extends AbstractRoutingDataSource {
    private static ThreadLocal<EnumDB> latestDB = new ThreadLocal<>();

    @Override
    protected Object determineCurrentLookupKey() {
        EnumDB dataSource = RoutingDatasourceManager.getCurrentDataSource();

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

- `Transaction` 방식과 동일하게 라우팅은 `AbstractRoutingDataSource` 의 `determineCurrentLookupKey()` 메소드의 구현으로, 
설정 파일에서 생성한 `RoutingDataSource` 에 있는 적절한 키를 리턴하는 방법을 사용한다. 
- `latestDB` 필드는 사용되는 실제 `DataSource` 의 로깅과 테스트를 위해 사용한다. 
- `determineCurrentLookupKey()` 메소드에서는 `RoutingDataSourceManager` 의 `getCurrentDataSource()` 메소드로 리턴받은 `DataSource` 키를 리턴하는 역할을 수행한다. 

`AOP` 를 통해 현재 사용하려는 `DataSource` 의 키를 관리하는 클래스의 내용은 아래와 같다. 

```java
public class RoutingDatasourceManager {
    private static final ThreadLocal<EnumDB> currentDataSource = new ThreadLocal<>();

    static {
        currentDataSource.set(EnumDB.MASTER_DB_1);
    }

    public static void setCurrentDataSource(EnumDB enumDB) {
        currentDataSource.set(enumDB);
    }

    public static EnumDB getCurrentDataSource() {
        return currentDataSource.get();
    }

    public static void removeCurrentDataSource() {
        currentDataSource.remove();
    }
}
```  

- `currentDataSource` 는 `ThreadLocal` 타입으로 현재 사용하는 `DataSource` 의 키를 저장하는 필드이다. 
- 구현된 메소드는 `currentDataSource` 필드를 외부에서 사용할 수 있도록 구현된 인터페이스이다. 

`AOP` 는 `Annotation` 을 사용해서 동작을 수행하도록 구현했다. 
`Annotation` 과 이를 처리하는 `AOP` 의 구현 내용은 아래와 같다. 

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
@Inherited
@Documented
public @interface SetDataSource {
    EnumDB dataSource() default EnumDB.MASTER_DB_1;
}
```  

- `SetDataSource` 라는 `Annotation` 은 메소드와 클래스 레벨에서 사용가능하다. 


```java
@Component
@Aspect
@Slf4j
public class SetDataSourceAspect {
    @Pointcut("@annotation(setDataSource)")
    public void hasAnnotation(SetDataSource setDataSource) {

    }

    @Pointcut("@within(setDataSource)")
    public void withinAnnotation(SetDataSource setDataSource) {

    }

    @Around("hasAnnotation(setDataSource)")
    public Object aroundMethod(ProceedingJoinPoint joinPoint, SetDataSource setDataSource) throws Throwable {
        return this.processAroundJoinPoint(joinPoint, setDataSource);
    }

    @Around("withinAnnotation(setDataSource)")
    public Object aroundClass(ProceedingJoinPoint joinPoint, SetDataSource setDataSource) throws Throwable {
        return this.processAroundJoinPoint(joinPoint, setDataSource);
    }

    public Object processAroundJoinPoint(ProceedingJoinPoint joinPoint, SetDataSource setDataSource) throws Throwable {
        EnumDB dataSource = setDataSource.dataSource();
        RoutingDatasourceManager.setCurrentDataSource(dataSource);

        log.debug("aspect : {}", dataSource);

        try {
            return joinPoint.proceed();
        } finally {
            RoutingDatasourceManager.removeCurrentDataSource();
        }
    }
}
```  

- `hasAnnotation()` 메소드는 메소드 레벨에 해당 `Annotation` 이 선언되었는지 판별한다. 
- `withinAnnotation()` 메소드는 클래스 레벨에 해당 `Annotation` 이 선언되었는지 판별한다. 
- `aroundMethod()` 는 메소드에 선언된 `Annotation` 의 `AOP` 동작을 수행한다. 
- `aroundClass()` 는 클래스에 선언된 `Annotation` 의 `AOP` 동작을 수행한다. 

`AOP` 와 `Annotation` 을 사용하는 서비스 클래스는 아래와 같다. 

```java
@Service
public class ExamService {
    private final ExamRepository examRepository;

    public ExamService(ExamRepository examRepository) {
        this.examRepository = examRepository;
    }

    @SetDataSource(dataSource = EnumDB.MASTER_DB_1)
    public Exam save(Exam exam) {
        return this.examRepository.save(exam);
    }

    @SetDataSource(dataSource = EnumDB.MASTER_DB_1)
    public void delete(Exam exam) {
        this.examRepository.delete(exam);
    }

    @SetDataSource(dataSource = EnumDB.MASTER_DB_1)
    public void deleteById(long id) {
        this.examRepository.deleteById(id);
    }

    @SetDataSource(dataSource = EnumDB.MASTER_DB_1)
    public void deleteAll() {
        this.examRepository.deleteAll();
    }

    @SetDataSource(dataSource = EnumDB.SLAVE_DB_1)
    public Exam readById(long id) {
        return this.examRepository.findById(id).orElse(null);
    }

    @SetDataSource(dataSource = EnumDB.SLAVE_DB_1)
    public List<Exam> readAll() {
        return this.examRepository.findAll();
    }
}
```  

```java
@Service
@SetDataSource(dataSource = EnumDB.MASTER_DB_2)
public class Exam2MasterService {
    private final Exam2Repository exam2Repository;

    public Exam2MasterService(Exam2Repository exam2Repository) {
        this.exam2Repository = exam2Repository;
    }

    public Exam2 save(Exam2 exam2) {
        return this.exam2Repository.save(exam2);
    }

    public void delete(Exam2 exam2) {
        this.exam2Repository.delete(exam2);
    }

    public void deleteById(long id) {
        this.exam2Repository.deleteById(id);
    }

    public void deleteAll() {
        this.exam2Repository.deleteAll();
    }
}
```  

```java
@Service
@SetDataSource(dataSource = EnumDB.SLAVE_DB_2)
public class Exam2SlaveService {
    private final Exam2Repository exam2Repository;

    public Exam2SlaveService(Exam2Repository exam2Repository) {
        this.exam2Repository = exam2Repository;
    }

    public Exam2 readById(long id) {
        return this.exam2Repository.findById(id).orElse(null);
    }

    public List<Exam2> readAll() {
        return this.exam2Repository.findAll();
    }
}
```  

### 테스트
아래 테스트 코드를 확인해보면 `Annotation` 에 설정한 `DataSource` 에 따라 정상적인 동작이 수행되는 것을 확인 할 수 있다. 

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ExamServiceTest {
    @Autowired
    private ExamService examService;

    @Before
    public void setUp() {
        this.examService.deleteAll();
    }

    @Test
    public void save_Master() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();

        // when
        this.examService.save(exam);

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
        assertThat(exam.getId(), greaterThan(0l));
    }

    @Test
    public void delete_Master() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam);

        // when
        this.examService.delete(exam);

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));
    }

    @Test
    public void readById_Slave() {
        // given
        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam);

        // when
        Exam result = this.examService.readById(exam.getId());

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));
        assertThat(result.getId(), is(exam.getId()));
    }

    @Test
    public void readAll_Slave() {
        // given
        Exam exam1 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam1);
        Exam exam2 = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam2);

        // then
        List<Exam> result = this.examService.readAll();

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
public class Exam2MasterServiceTest {
    @Autowired
    private Exam2MasterService exam2MasterService;

    @Before
    public void setUp() {
        this.exam2MasterService.deleteAll();
    }

    @Test
    public void save_Master() {
        // given
        Exam2 exam = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();

        // when
        this.exam2MasterService.save(exam);

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));
        assertThat(exam.getId(), greaterThan(0l));
    }

    @Test
    public void delete_Master() {
        // given
        Exam2 exam = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam);

        // when
        this.exam2MasterService.delete(exam);

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class Exam2SlaveServiceTest {
    @Autowired
    private Exam2SlaveService exam2SlaveService;
    @Autowired
    private Exam2MasterService exam2MasterService;

    @Before
    public void setUp() {
        this.exam2MasterService.deleteAll();
    }

    @Test
    public void readById_Slave() {
        // given
        Exam2 exam = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam);

        // when
        Exam2 result = this.exam2SlaveService.readById(exam.getId());

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_2));
        assertThat(result.getId(), is(exam.getId()));
    }

    @Test
    public void readAll_Slave() {
        // given
        Exam2 exam1 = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam1);
        Exam2 exam2 = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam2);

        // when
        List<Exam2> result = this.exam2SlaveService.readAll();

        // then
        EnumDB actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_2));
        assertThat(result, hasSize(2));
    }
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MultipleDataSourceTest {
    @Autowired
    private ExamService examService;
    @Autowired
    private Exam2MasterService exam2MasterService;
    @Autowired
    private Exam2SlaveService exam2SlaveService;

    @Before
    public void setUp() {
        this.examService.deleteAll();
        this.exam2MasterService.deleteAll();
    }

    @Test
    public void insert_select_delete() {
        EnumDB actual;

        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        Exam2 exam2 = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam2);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));

        this.examService.readAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.exam2SlaveService.readAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_2));

        this.examService.deleteAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        this.exam2MasterService.deleteAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));
    }

    @Test
    public void insert_insert_select_select_delete_delete() {
        EnumDB actual;

        Exam exam = Exam.builder()
                .value("a")
                .datetime(LocalDateTime.now())
                .build();
        this.examService.save(exam);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        Exam2 exam2 = Exam2.builder()
                .value2("a")
                .datetime(LocalDateTime.now())
                .build();
        this.exam2MasterService.save(exam2);
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));

        this.examService.readAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.exam2SlaveService.readAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_2));

        this.examService.readById(exam.getId());
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_1));

        this.exam2SlaveService.readById(exam2.getId());
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.SLAVE_DB_2));

        this.examService.deleteAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_1));

        this.exam2MasterService.deleteAll();
        actual = RoutingDataSource.getLatestDB();
        assertThat(actual, is(EnumDB.MASTER_DB_2));
    }
}
```  

---
## Reference
[Use replica database for read-only transactions](https://blog.pchudzik.com/201911/read-from-replica/)  