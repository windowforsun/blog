--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch Job 설정과 실행 "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Bathc 에서 Job 을 설정하고 실행하는 방법과 그 구성요소에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Job
    - JobRepository
    - JobExplorer
    - JobRegistry
    - JobOperator
toc: true
use_math: true
---  

## Job 설정 및 실행
`Spring Batch` 에서 `Job` 을 설정하고 실행하기 위해서는 다양한 옵션들이 존재한다. 
또한 실행 과정에서 생성되는 메타 데이터들을 관리 및 조작하기 위한 구성요소들도 존재한다. 

### 예제 프로젝트
`Spring Batch` 에서 `Job` 관련 설정하고 실행에 대한 예제 프로젝트를 간단하게 소개하며 시작한다. 

#### 프로젝트 구조

```
│  build.gradle
│  output-data
│  tmp-data
│
└─src
    ├─main
    │  ├─java
    │  │  │
    │  │  └─com
    │  │      └─windowforsun
    │  │          └─jobexam
    │  │              │  JobExamApplication.java
    │  │              │
    │  │              ├─component
    │  │              │      SampleIncrementer.java
    │  │              │      SampleJobListener.java
    │  │              │      SampleJobParametersValidator.java
    │  │              │
    │  │              ├─config
    │  │              │      AsynchronousBatchConfigurer.java
    │  │              │      AsynchronousJobConfig.java
    │  │              │      IncrementerJobConfig.java
    │  │              │      InMemoryBatchConfigurer.java
    │  │              │      InMemoryJobConfig.java
    │  │              │      JobOperatorJobConfig.java
    │  │              │      JobParametersValidatorJobConfig.java
    │  │              │      JobRegistryJobConfig.java
    │  │              │      ListenerJobConfig.java
    │  │              │      RestartabilityJobConfig.java
    │  │              │      SampleJobConfig.java
    │  │              │      StepConfig.java
    │  │              │
    │  │              └─step
    │  │                      PrefixItemProcessor.java
    │  │                      UpperCaseItemProcessor.java
    │  │
    │  └─resources
    │          input-data
    │
    └─test
        └─java
          │
          └─com
              └─windowforsun
                  └─jobexam
                          AsynchronousJobTest.java
                          BatchTestConfig.java
                          IncrementerJobTest.java
                          InMemoryJobTest.java
                          JobParametersValidatorJobTest.java
                          ListenerJobTest.java
                          MetaDataJobExplorerTest.java
                          MetaDataJobOperatorTest.java
                          MetaDataJobRegistryTest.java
                          RestartabilityJobTest.java
                          SampleJobTest.java
                          Util.java

```  

#### build.gradle

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.2.RELEASE'
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    runtimeOnly 'org.hsqldb:hsqldb'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testImplementation 'org.springframework.batch:spring-batch-test'

    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
}

test {
    useJUnitPlatform()
}
```  

#### StepConfig
이후 예제로 등장하는 `Job` 들은 아래의 공통된 `Step` 사용해서 구성된다. 

```java
@Configuration
@RequiredArgsConstructor
public class StepConfig {
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public ItemReader<String> prefixReader() {
        return new FlatFileItemReaderBuilder<String>()
                .name("prefixReader")
                .resource(new ClassPathResource("input-data"))
                .lineMapper(new PassThroughLineMapper())
                .lineTokenizer(new DelimitedLineTokenizer())
                .build();
    }

    @Bean
    public ItemProcessor<String, String> prefixProcessor() {
        return new PrefixItemProcessor();
    }

    @Bean
    public ItemWriter<String> prefixWriter() {
        return new FlatFileItemWriterBuilder<String>()
                .name("prefixWriter")
                .resource(new FileSystemResource("tmp-data"))
                .lineAggregator(new PassThroughLineAggregator<>())
                .build();
    }

    @Bean
    public Step prefixStep() {
        return this.stepBuilderFactory
                .get("prefixStep")
                .<String, String>chunk(2)
                .reader(this.prefixReader())
                .processor(this.prefixProcessor())
                .writer(this.prefixWriter())
                .build();
    }

    @Bean
    public ItemReader<String> upperCaseReader() {
        return new FlatFileItemReaderBuilder<String>()
                .name("upperCaseReader")
                .resource(new FileSystemResource("tmp-data"))
                .lineMapper(new PassThroughLineMapper())
                .lineTokenizer(new DelimitedLineTokenizer())
                .build();
    }

    @Bean
    public ItemProcessor<String, String> upperCaseProcessor() {
        return new UpperCaseItemProcessor();
    }

    @Bean
    public ItemWriter<String> upperCaseWriter() {
        return new FlatFileItemWriterBuilder<String>()
                .name("upperCaseWriter")
                .resource(new FileSystemResource("output-data"))
                .lineAggregator(new PassThroughLineAggregator<>())
                .build();
    }

    @Bean
    public Step upperCaseStep() {
        return this.stepBuilderFactory
                .get("upperCaseStep")
                .<String, String>chunk(2)
                .reader(this.upperCaseReader())
                .processor(this.upperCaseProcessor())
                .writer(this.upperCaseWriter())
                .build();
    }

}
```  

먼저 `prefixStep` 은 `resources` 디렉토리 하위에 있는 `input-data` 를 한줄 한줄 읽어 접두사를 붙이고, 
프로젝트 최상위 경로에 `tmp-data` 이름으로 결과 파일을 쓰는 `Step` 이다. 
`prefixStep` 의 `ItemProcessor` 역할을 하는 `PrefixItemProcessor` 는 아래와 같다. 

```java
@Slf4j
public class PrefixItemProcessor implements ItemProcessor<String, String> {
    public static boolean IS_FAIL = false;
    public static long SLEEP_TIME = 0L;

    @Override
    public String process(String item) throws Exception {
        if(IS_FAIL) {
            IS_FAIL = false;
            throw new Exception("prefixProcess fail");
        }

        Thread.sleep(SLEEP_TIME);

        StringBuilder builder = new StringBuilder("test-")
                .append(item);

        log.info(builder.toString());

        return builder.toString();
    }
}
```  

다음으로 `upperCaseStep` 은 `prefixStep` 에서 쓴 `tmp-data` 파일을 한줄 한줄 읽어 모두 `UpperCase` 처리해서, 
프로젝트 최상위 경로에 `output-data` 이름으로 결과 파일을 쓰는 `Step` 이다. 
`upperCaseStep` 의 `ItemProcessor` 역할을 하는 `UpperCaseItemProcessor` 는 아래와 같다. 

```java
@Slf4j
public class UpperCaseItemProcessor implements ItemProcessor<String, String> {
    public static boolean IS_FAIL = false;
    public static long SLEEP_TIME = 0L;

    @Override
    public String process(String item) throws Exception {
        if(IS_FAIL) {
            IS_FAIL = false;
            throw new Exception("upperCaseProcess fail");
        }

        Thread.sleep(SLEEP_TIME);

        String upperCase = item.toUpperCase();

        log.info(upperCase);

        return upperCase;
    }
}
```  

이후 등장하는 `Job` 설정 및 예제는 위 `Step` 을 사용해서 구성된다.  

#### 기타
- `BatchTestConfig`

```java
@EnableBatchProcessing
@EnableAutoConfiguration
public class BatchTestConfig {
}
```  

- `Util` 

```java
public class Util {
    public static void deleteAllBatchFile() throws Exception{
        String[] files = {"output-data", "tmp-data"};

        for (String file : files) {
            Resource resource = new FileSystemResource(file);
            if (resource.exists()) {
                resource.getFile().delete();
            }
        }
    }
}
```  

- `input-data`

```
a
b
c
d
e
f
g
h
i
```  




### Job 설정
`Job` 은 `JobBuilderFatory` 를 이용해서 아래와 같이 간단하게 빈으로 생성할 수 있다. 

```java
@Bean
public Job sampleJob() {
    return this.jobBuilderFactory
            .get("sampleJob")
            .start(this.prefixStep)
            .next(this.upperCaseStep)
            .build();
}
```  

위 `Job` 설정을 보면 `prefixStep` 과 `upperCaseStep` 으로 구성된 것을 확인 할 수 있다. 
그리고 `prefixStep` 이 실행되고 성공하면 `upperCaseStep` 이 실행된다. 
또한 `Flow` 라는 개념을 사용해서 선행 `Step` 의 성공/실패와 같은 조건에 따라 다른 `Step` 을 실행할 수 있다.  

#### 테스트
- `SampleJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class SampleJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public Job sampleJob() {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .build();
    }
}
```  

- `SampleJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        SampleJobConfig.class
})
public class SampleJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void whenLaunchSampleJob_whenStatusAndLoggingCompleted() throws Exception {
        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "test-a", "test-b", "test-c", "TEST-D", "TEST-E", "TEST-F"
        ));
    }

    @Test(expected = JobInstanceAlreadyCompleteException.class)
    public void givenSameJobParameters_whenTwiceLaunchSampleJob_thenJobInstanceAlreadyCompleteException() throws Exception {
        // given
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("key", 100L)
                .toJobParameters();
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);
    }

    @Test
    public void givenSameJobParameters_whenFirstLaunchFailedAndSecondLaunchSuccess_thenCompletedAndLaunchAfterPrefixProcessor() throws Exception {
        // given
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("key", 100L)
                .toJobParameters();
        UpperCaseItemProcessor.IS_FAIL = true;
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
            "test-a", "test-b", "test-c", "TEST-D", "TEST-E", "TEST-F"
        ));
    }
}
```  

`SampleJob` 이 정상적으로 실행되면 `Job` 을 구성하는 `Step` 의 `ItemProcessor` 에서 출력되는 로그를 나열하면 아래와 같다. 

```
.. PrefixItemProcessor 에서 출력 ..
test-a
test-b
test-c
test-d
test-e
test-f
test-g
test-h
test-i

.. UpperCaseItemProcessor 에서 출력 ..
TEST-A
TEST-B
TEST-C
TEST-D
TEST-E
TEST-F
TEST-G
TEST-I
```  

### Restartability(재시작 여부)
`Job` 실행에 대한 메타 데이터로는 `JobInstance` 와 `JobExecution` 이 있다. 
여기서 `Job` 에 대한 재실행은 `JobInstance` 에 대해 `JobExecution` 이 존재한다면 이를 재시작으로 간주한다. 
재시작이 가능한 경우는 `JobInstance` 에 대해 `JobExecution` 이 실패인 상태인 경우에만 가능하다.  

일반적으로 `Job`이 여러 `Step` 으로 구성됐을 경우 앞선 `Step` 이 실패해서 전체 `Job` 이 실패했을때, 
동일한 `JobParameters` 로 `Job` 을 실행하면 성공한 `Step` 은 실행하지 않고 실패한 `Step` 부터 실행된다. 
하지만 경우에 따라 이러한 `Job` 의 재시작 정책이 `Job` 을 구성하는 비지니스 로직에는 부합하지 않을 수 있다. 
이런 경우 `preventRestart` 옵션을 사용해서 애초에 `JobExecution` 이 실패하더라도, 
동일한 `JobParameters` 로는 `Job` 을 실행하지 못하고 `JobRestartException` 이 발생하게 된다.  

- `RestartabilityJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class RestartabilityJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public Job restartabilityJob() {
        return this.jobBuilderFactory
                .get("restartabilityJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .preventRestart()
                .build();
    }
}
```  

- `RestartabilityJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        RestartabilityJobConfig.class
})
public class RestartabilityJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test(expected = JobRestartException.class)
    public void givenSameJobParameters_whenTwiceLaunch_thenJobRestartException() throws Exception {
        // given
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("key", 100L)
                .toJobParameters();
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);
    }

    @Test(expected = JobRestartException.class)
    public void givenSameJobParameters_whenFirstLaunchFailAndSecondLaunchSuccess_thenJobRestartException() throws Exception {
        // given
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("key", 100L)
                .toJobParameters();
        UpperCaseItemProcessor.IS_FAIL = true;
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);
    }
}
```  

### Interceptor(Listener)
`Job` 의 라이프사이클에서 발생한 이벤트를 받아 원하는 동작을 수행할 수 있다. 
`JobExecutionListener` 인터페이스 구현을 통해 원하는 동작을 정의할 수 있다. 

- `SampleJobListener`

```java
@Slf4j
public class SampleJobListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("beforeJob");
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if(jobExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("afterJob COMPLETED");
        } else if(jobExecution.getStatus() == BatchStatus.FAILED) {
            log.info("afterJob FAILED");
        }
    }
}
```  

`SampleJobListener` 는 `Job` 시작전엔 `beforeJob` 로그를 출력한다. 
그리고 `Job` 이 실패하면 `afterJob FAILED`, 성공하면 `afterJob COMPLTED` 로그를 출력한다.  

- `ListenerJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class ListenerJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public Job listenerJob() {
        return this.jobBuilderFactory
                .get("listenerJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .listener(new SampleJobListener())
                .build();
    }
}
```  

- `ListenerJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        ListenerJobConfig.class
})
public class ListenerJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void givenAllStepSuccess_whenLaunchListenerJob_thenCompletedListenerLogging() throws Exception {
        // given
        PrefixItemProcessor.IS_FAIL = false;
        UpperCaseItemProcessor.IS_FAIL = false;

        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "beforeJob",
                "test-a",
                "test-b",
                "test-c",
                "TEST-D",
                "TEST-E",
                "TEST-F",
                "afterJob COMPLETED"
        ));
    }

    @Test
    public void givenUpperCaseStepFail_whenLaunchListenerJob_thenFailedListenerLogging() throws Exception {
        // given
        PrefixItemProcessor.IS_FAIL = false;
        UpperCaseItemProcessor.IS_FAIL = true;

        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                stringContainsInOrder(
                        "beforeJob",
                        "test-a",
                        "test-b",
                        "test-c",
                        "afterJob FAILED"
                ),
                not(containsString("TEST-C")),
                not(containsString("TEST-D")),
                not(containsString("TEST-F"))
        ));
    }
}
```  

### JobParametersValidator
`Job` 을 실행할때 `JobInstance` 구분 및 실행에 필요한 인자값을 전달하기 위해선 `JobParameters` 를 사용한다. 
여기서 `JobParametersValidator` 를 사용하면 `JobParameters` 에 대한 유효성 검사를 수행할 수 있다. 
`Spring Batch` 에서는 기본적인 유효성 검사를 수행하는 `DefaultJobParametersValidator` 구현체를 제공한다. 

- `SampleJobParametersValidator`

```java
public class SampleJobParametersValidator implements JobParametersValidator {
    @Override
    public void validate(JobParameters parameters) throws JobParametersInvalidException {
        long num = parameters.getLong("num");

        if(num < 100) {
            throw new JobParametersInvalidException("num must be gt 100");
        }
    }
}
```  

`SampleJobParametersValidator` 는 `JobParameters` 에 `num` 키로 `long` 값이 있어야 하고, 
해당 값이 100 보다 큰지 검사한다. 

- `JobParametersValidatorJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class JobParametersValidatorJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public Job jobParametersValidatorJob() {
        return this.jobBuilderFactory
                .get("jobParametersValidatorJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .validator(new SampleJobParametersValidator())
                .build();
    }
}
```  

- `JobParametersValidatorJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        JobParametersValidatorJobConfig.class
})
public class JobParametersValidatorJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test(expected = NullPointerException.class)
    public void givenNotExistNum_whenLaunchJob_thenNullPointException() throws Exception {
        // given
        JobParameters jobParameters = new JobParameters();

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);
    }

    @Test(expected = JobParametersInvalidException.class)
    public void givenNumIsLT100_whenLaunchJob_thenJobParametersInvalidException() throws Exception {
        // given
        long num = 1L;
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("num", num)
                .toJobParameters();

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);
    }

    @Test
    public void givenNumIsGT100_whenLaunchJob_thenCompleted() throws Exception {
        // given
        long num = 101L;
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("num", num)
                .toJobParameters();

        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }
}
```  

### Batch 관련 설정
`Spring Batch` 는 `@EnableBatchProcessing` 어노테이션을 사용하면 모든 기본적인 구성요소들을 자동 설정할 수 있다. 
`Spring Batch` 구성에 필요하면서 자동 설정 대상이 되는 구성요소 클래스와 빈 이름을 나열하면 아래와 같다. 

- `JobRepository` : `jobRepository`
- `JobLauncher` : `jobLauncher`
- `JobRegistry` : `jobRegistry`
- `PlatformTransactionManager` : `transactionManager`
- `JobBuilderFactory`  : `jobBuilders`
- `StepBuilderFactry` : `stepBuilders`

만약 자동 설정이 아닌 커스텀한 설정이 필요하다면 `BatchConfigurer` 인터페이스 구현을 통해 가능하다. 
`BatchConfigurer` 은 기본적으로 `JobRepository` 에 필요한 `DataSource` 에 대한 빈을 설정해 줘야 한다. 
앞서 나열한 모든 구성요소를 모두 설정할 필요가 없다면 `DefaultBatchConfigurer` 클래스를 상속받아 해당하는 메소드를 오버라이딩을 통해 가능하다. 
`BatchConfigurer` 인터페이스의 내용은 아래와 같다. 

```java
public interface BatchConfigurer {

	JobRepository getJobRepository() throws Exception;

	PlatformTransactionManager getTransactionManager() throws Exception;

	JobLauncher getJobLauncher() throws Exception;

	JobExplorer getJobExplorer() throws Exception;
}
```  

### JobRepository 설정
`JobRepository` 는 `JobExecution`, `StepExecution` 와 같은 `Spring Batch` 에서 사용되는 여러 오브젝트에 대한
`CRUD` 를 제공한다. 
그리고 `JobLauncher`, `Job`, `Step` 등과 같은 주요 구성요소에서 사용되어 진다.  

커스텀한 `JobRepository` 설정을 위해서는 `DataSource` 와 `PlatformTransactionManager` 가 필수로 요구 된다. 
다른 속성은 설정하지 않으면 모두 기본 값으로 설정된다. 
외부 저장소를 사용하는 `JobRepository` 는 `DataSource` 를 사용해서 `JobRepositoryFactoryBean` 객체를 사용해서 설정할 수 있다. 
- `setIsolationLevelForCreate("ISOLATION_REPEATABLE_READ")` : 트랜젝션의 `Isolation level` 을 설정한다. 
- `setTablePrefix("TEST_")` : `Spring Batch` 에서 메타 데이터 테이블에 대한 접두사를 설정한다. 
`BATCH_JOB_EXECUTION` 테이블이 `TEST_BATCH_JOB_EXECUTION` 으로 변경된다. 
해당 설정은 `JobRepository` 가 수행하는 쿼리에 대한 테이블 이름만 변경할 뿐 메타 데이터 테이블은 직접 설정해줘야 한다.  
- `setDatabaseType("db2")` : 사용자가 원하는 데이터베이스 타입을 지정할 수 있다. 
지원하지 않는 데이터베이스와 `JobRepository` 를 연동하고 싶다면 가장 비슷한 쿼리를 가진 데이터베이스를 선택하거나, 
`SimpleJobRepository` 가 의존하는 여러 `Dao` 인터페이스를 직접 구현하는 방법이 있다. 

`Job` 이 수행되며 생성되는 메타 데이터르 데이터베이스가 아닌 `In Memory` 에 저장할 수 있는 방법에 대해 알아본다.   

- `InMemoryBatchConfigurer`

```java
public class InMemoryBatchConfigurer extends DefaultBatchConfigurer {
    @Override
    protected JobRepository createJobRepository() throws Exception {
        MapJobRepositoryFactoryBean jobRepositoryFactoryBean = new MapJobRepositoryFactoryBean(getTransactionManager());
        jobRepositoryFactoryBean.afterPropertiesSet();

        return jobRepositoryFactoryBean.getObject();
    }
}
```  

`DefaultBatchConfigurer` 의 `createJobRepository()` 메소드를 오버라이딩해서 `In Memory` 형태로 구성한다. 
`In Memory` 인 만큼 만약 애플리케이션이 재시작된다면 모든 메타 데이터 정보를 사라지게 된다. 
이렇게 휘발성인 특징을 가지므로 여러 `JVM` 을 띄우는 방식으로 이중화는 불가능하다. 
그리고 같은 파라미터로 두 개의 `Job` 인스턴스를 동시에 실행할 수 없고, 
멀티 스레드 `Job` 이나 애플리케이션 내에서 파티셔닝한 `Step` 에는 적합하지 않다. 

- `InMemoryJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class InMemoryJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public BatchConfigurer batchConfigurer () {
        return new InMemoryBatchConfigurer();
    }

    @Bean
    public Job inMemoryJob() {
        return this.jobBuilderFactory
                .get("InMemoryJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .build();
    }
}
```  

- `InMemoryJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        InMemoryJobConfig.class
})
public class InMemoryJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void whenLaunchInMemoryJob_thenCompleted() throws Exception {
        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }
}
```  

### JobLauncher 설정
`JobLauncher` 는 등록된 `Job` 을 실행하는 역할을 수행한다. 
`JobLauncher` 의 기본적인 구현체는 `SimpleJobLauncher` 이고 `JobRepository` 의존성을 통해 생성할 수 있다.  

`JobLauncher` 를 통해 기본적인 `Job` 의 실행 과정은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-config-run-job-1.png)  

`JobLauncher` 가 `Client` 에 의해 실행되면 최종적으로 사용자가 정의한 비지니스 로직이 실행된다. 
그리고 모든 `Batch` 작업이 수행이 완료되면 `Client` 에게 최종적으로 `JobExecution` 이 리턴된다. 
그리고 `JobExecution` 에서 `ExitStatus` 를 통해 `Job` 의 성공여부를 확인할 수 있다.  

아래는 `JobLauncher` 가 비동기로 동작할때 `Job` 의 실행 과정이다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-config-run-job-2.png)  


만약 `JobLauncher` 의 `Trigger` 역할을 `Controller`(HTTP 요청) 가 한다면 오랜 시간이 소요되는 `Batch` 작업의 특성에 따라 
`HTTP` 타임아웃으로 인해 `Client` 는 해당 `Job` 에 대한 결과르 받지 못하고 `HTTP` 세션이 끊길 수 있다. 
위와 같이 비동기를 사용하면 우선 `HTTP` 를 사용해서 `Job` 을 실행하고 이후 결과를 요청해서 받아 볼 수 있게 된다. 
아래 예제를 통해 `JobLauncher` 를 비동기로 구성해서 사용할 수 있다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-config-run-job-3.png)  

- `AsynchronousBatchConfigurer`

```java
public class AsynchronousBatchConfigurer extends DefaultBatchConfigurer {
    @Override
    protected JobLauncher createJobLauncher() throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
        jobLauncher.setJobRepository(getJobRepository());
        jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());
        jobLauncher.afterPropertiesSet();

        return jobLauncher;
    }
}
```  

`JobLauncher` 의 `TaskExecutor` 에 `SimpleAsyncTaskExecutor` 객체를 설정하면 비동기 설정이 가능하다.  

- `AsynchronousJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class AsynchronousJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public BatchConfigurer batchConfigurer() {
        return new AsynchronousBatchConfigurer();
    }

    @Bean
    public Job asynchronousJob() {
        return this.jobBuilderFactory
                .get("asynchronousJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .build();
    }
}
```  

- `AsynchronousJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        AsynchronousJobConfig.class
})
public class AsynchronousJobTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobOperator jobOperator;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void givenNoSleep_whenLaunchAsynchronousJob_thenUnknown() throws Exception {
        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.UNKNOWN.getExitCode()));
        actual.stop();
    }

    @Test
    public void givenSleepEnoughTime_whenLaunchAsynchronousJob_thenCompleted() throws Exception {
        // given
        long sleepTime = 1000L;

        // when
        JobExecution actual = this.jobLauncherTestUtils.launchJob();

        // then
        Thread.sleep(sleepTime);
        assertThat(actual.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }
}
```  

비동기 `JobLauncher` 는 실행하게 되면 `JobExecution` 을 반환하고 `ExitStatus` 는 `UNKNOWN` 이다. 
그리고 `Job` 실행이 완료되면 `JobExecution` 의 `ExitStatus` 가 `COMPLETED` 로 변하는 것을 확인 할 수 있다.  

### Meta Data(메타 데이터)
앞서 설명만 `JobLauncher` 와 `JobRepository` 모두 `Job` 을 조작하고 라이프사이클에 대한 메이터 데이터의 `CRUD` 를 제공한다.  

![그림 1]({{site.baseurl}}/img/spring/concept-batch-config-run-job-4.png)  

`JobLauncher` 는 `JobExecution` 객체를 생성하고 메타 데이터를 위해 `JobRepository` 를 사용한다. 
그리고 이후 `Job` 의 라이프사이클에서 `Job`, `Step` 에 대한 `Execution` 을 생성 및 업데이트를 위해서도 `JobRepository` 를 사용한다.  

![그림 1]({{site.baseurl}}/img/spring/concept-batch-config-run-job-5.png)  

### JobExplorer
`JobExplorer` 는 `JobInstance`, `JobExecution`, `StepExecution` 등을 조회 할 수 있는 인터페이스이다. 
그리고 `JobExplorer` 는 아래와 같이 `JobRepository` 에서 읽기 관련 동작만 있다. 

```java
public interface JobExplorer {

	List<JobInstance> getJobInstances(String jobName, int start, int count);

	@Nullable
	default JobInstance getLastJobInstance(String jobName) {
		throw new UnsupportedOperationException();
	}

	@Nullable
	JobExecution getJobExecution(@Nullable Long executionId);

	@Nullable
	StepExecution getStepExecution(@Nullable Long jobExecutionId, @Nullable Long stepExecutionId);

	@Nullable
	JobInstance getJobInstance(@Nullable Long instanceId);

	List<JobExecution> getJobExecutions(JobInstance jobInstance);

	@Nullable
	default JobExecution getLastJobExecution(JobInstance jobInstance) {
		throw new UnsupportedOperationException();
	}

	Set<JobExecution> findRunningJobExecutions(@Nullable String jobName);

	List<String> getJobNames();
	
	List<JobInstance> findJobInstancesByJobName(String jobName, int start, int count);

	int getJobInstanceCount(@Nullable String jobName) throws NoSuchJobException;

}
```  

커스텀한 `JobExplorer` 를 설정하기 위해서는 `DataSource` 빈만 있으면 가능하다. 
그리고 만약 `JobRepository` 에서 메타 데이터 테이블의 접두사를 설정했다면 `JobExplorer` 에서도 설정이 필요하다. 

```java
@Override
public JobExplorer getJobExplorer() throws Exception {
	JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();
	factoryBean.setDataSource(this.dataSource);
	factoryBean.setTablePrefix("TEST_");
	return factoryBean.getObject();
}
```  

- `MetaDataJobExplorerTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        SampleJobConfig.class
})
public class MetaDataJobExplorerTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobExplorer jobExplorer;
    @Autowired
    private JobRepository jobRepository;


    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void getJobInstance() throws Exception {
        // given
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // when
        JobInstance actual = this.jobExplorer.getJobInstance(jobExecution.getJobInstance().getInstanceId());

        // then
        assertThat(actual, is(jobExecution.getJobInstance()));
    }

    @Test
    public void getJobInstanceCount() throws Exception {
        // given
        this.jobLauncherTestUtils.launchJob();
        this.jobLauncherTestUtils.launchJob();

        // when
        int actual = this.jobExplorer.getJobInstanceCount("sampleJob");

        // then
        assertThat(actual, is(2));
    }

    @Test
    public void getLastJobInstance() throws Exception {
        // given
        this.jobLauncherTestUtils.launchJob();
        this.jobLauncherTestUtils.launchJob();
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // when
        JobInstance actual = this.jobExplorer.getLastJobInstance("sampleJob");

        // then
        assertThat(actual, is(jobExecution.getJobInstance()));
    }

    @Test
    public void getJobExecutions() throws Exception {
        // given
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        JobParameters jobParameters = jobParametersBuilder
                .addLong("key", 1L)
                .toJobParameters();
        UpperCaseItemProcessor.IS_FAIL = true;
        this.jobLauncherTestUtils.launchJob(jobParameters);
        UpperCaseItemProcessor.IS_FAIL = false;
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        List<JobExecution> actual = this.jobExplorer.getJobExecutions(jobExecution.getJobInstance());

        // then
        assertThat(actual, hasSize(2));
    }

    @Test
    public void getStepExecution() throws Exception {
        // given
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();
        StepExecution[] stepExecutions = jobExecution.getStepExecutions().toArray(new StepExecution[0]);

        // when
        StepExecution actual1 = this.jobExplorer.getStepExecution(jobExecution.getId(), stepExecutions[0].getId());
        StepExecution actual2 = this.jobExplorer.getStepExecution(jobExecution.getId(), stepExecutions[1].getId());

        // then
        assertThat(actual1, is(stepExecutions[0]));
        assertThat(actual1.getStepName(), is("prefixStep"));
        assertThat(actual2, is(stepExecutions[1]));
        assertThat(actual2.getStepName(), is("upperCaseStep"));
    }

    @Test
    public void findRunningJobExecutions() throws Exception {
        // given
        JobLauncher origin = this.jobLauncherTestUtils.getJobLauncher();
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
        jobLauncher.setJobRepository(this.jobRepository);
        jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());
        this.jobLauncherTestUtils.setJobLauncher(jobLauncher);
        UpperCaseItemProcessor.SLEEP_TIME = 100000L;
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();
        Thread.sleep(100);

        // when
        Set<JobExecution> actual = this.jobExplorer.findRunningJobExecutions("sampleJob");

        // then
        assertThat(actual, hasSize(1));
        assertThat(actual, hasItem(jobExecution));
        this.jobLauncherTestUtils.setJobLauncher(origin);
        UpperCaseItemProcessor.SLEEP_TIME = 0L;
        jobExecution.stop();
    }
}
```  

### JobRegistry
`JobRegistry` 는 `ApplicationContext` 내에 있는 `Job` 을 추적하거나 조작할때 유용한 객체이다. 
`Spring Batch` 사용에 있어 필수가 되는 객체는 아니지만, 
동적으로 `Job` 을 생성하는 구조에서 전체적인 `Job` 을 관리하고 조작하는데 사용할 수 있다. 
이후 언급될 `JobOperator` 이나 `JobLocator` 를 통해 `Job` 을 조작할때 `JobRegistry` 에 `Job` 객체가 등록돼야 한다.  

특정 `Job` 만 `JobRegistry` 에 등록하고 싶다면 아래와 같이 `Job` 생성과정에 등록하는 방식을 사용할 수 있다. 
그리고 애플리케이션의 `Context` 에 빈으로 생성된 `Job` 을 모두 자동으로 등록하고 싶을 경우 아래와 같이 `JobRegistryBeanPostProcessor` 를 사용할 수 있다. 

```java
@Bean
public JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor() {
    JobRegistryBeanPostProcessor postProcessor = new JobRegistryBeanPostProcessor();
    postProcessor.setJobRegistry(jobRegistry());
    return postProcessor;
}
```  

그리고 `AutomaticJobRegistry` 를 사용하면 `Job` 을 자식 컨텍스트에 등록하는 경우, 
자식 컨텍스트에 있는 `Job` 이름은 `JobRegistry` 전역에서 유일해야 하지만 `Job` 이 의존성을 갖는(`ItemReader`, `ItemWriter`, ..)
빈이름은 유일성을 가질 필요가 없다. 
2개의 `Job` 설정 파일이 있을때 설정 파일 모두에 `ItemReader` 빈인 `reader` 가 동일한 이름으로 있더라도 충돌되거나 덮어쓰지 않도록 방지해 준다. 

```java
@Bean
public AutomaticJobRegistrar registrar() {

    AutomaticJobRegistrar registrar = new AutomaticJobRegistrar();
    registrar.setJobLoader(jobLoader());
    registrar.setApplicationContextFactories(applicationContextFactories());
    registrar.afterPropertiesSet();
    return registrar;

}
```  

필요에 따라 `JobRegistryBeanPostProcessor`, `AutomaticJobRegistrar` 동시에 사용하는 것도 가능하다. 

- `JobRegistryJobConfig`

```java
{
    private final JobBuilderFactory jobBuilderFactory;
    private final JobRegistry jobRegistry;
    private final Step prefixStep;
    private final Step upperCaseStep;

    @Bean
    public Job registeredJob() throws Exception{
        Job job = this.jobBuilderFactory
                .get("registeredJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .build();

        ReferenceJobFactory factory = new ReferenceJobFactory(job);
        this.jobRegistry.register(factory);

        return job;
    }
}
```  

- ``

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        JobRegistryJobConfig.class
})
public class MetaDataJobRegistryTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobRegistry jobRegistry;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void getJob_Success() throws Exception {
        Job actual = this.jobRegistry.getJob("registeredJob");

        assertThat(actual, notNullValue());
    }

    @Test(expected = NoSuchJobException.class)
    public void getJob_Fail() throws Exception {
        this.jobRegistry.getJob("notRegisteredJob");
    }

    @Test
    public void getJobNames() {
        Collection<String> actual = this.jobRegistry.getJobNames();

        assertThat(actual, hasSize(1));
        assertThat(actual, hasItem("registeredJob"));
    }
}
```  

### JobOperator
`JobOperator` 는 메터 데이터의 `CRUD` 를 제공하는 `JobRepository` 와 읽기 관련 동작을 지원하는 `JobExplorer` 의 모든 동작을 제공하는 인터페이스이다. 
`JobOperator` 를 사용하게되면 `Job` 에 대한 다양한 동작을 컨트롤 할 수 있어 실행, 재시작, 중단, 모니터링과 같은 동작이 가능하다. 

```java
public interface JobOperator {

	List<Long> getExecutions(long instanceId) throws NoSuchJobInstanceException;

	List<Long> getJobInstances(String jobName, int start, int count) throws NoSuchJobException;

	Set<Long> getRunningExecutions(String jobName) throws NoSuchJobException;

	String getParameters(long executionId) throws NoSuchJobExecutionException;

	Long start(String jobName, String parameters) throws NoSuchJobException, JobInstanceAlreadyExistsException, JobParametersInvalidException;

	Long restart(long executionId) throws JobInstanceAlreadyCompleteException, NoSuchJobExecutionException,
			NoSuchJobException, JobRestartException, JobParametersInvalidException;

	Long startNextInstance(String jobName) throws NoSuchJobException, JobParametersNotFoundException,
			JobRestartException, JobExecutionAlreadyRunningException, JobInstanceAlreadyCompleteException, UnexpectedJobExecutionException, JobParametersInvalidException;

	boolean stop(long executionId) throws NoSuchJobExecutionException, JobExecutionNotRunningException;

	String getSummary(long executionId) throws NoSuchJobExecutionException;

	Map<Long, String> getStepExecutionSummaries(long executionId) throws NoSuchJobExecutionException;

	Set<String> getJobNames();

	JobExecution abandon(long jobExecutionId) throws NoSuchJobExecutionException, JobExecutionAlreadyRunningException;
}
```  

`JobOperator` 가 다양한 동작을 제공하는 만큼 `JobRepository`, `JobRegistry`, `JobExplorer` 등 많은 의존성을 가진다. 

- `JobOperatorJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class JobOperatorJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final JobRegistry jobRegistry;
    private final Step prefixStep;
    private final Step upperCaseStep;


    @Bean
    public JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor() throws Exception {
        JobRegistryBeanPostProcessor postProcessor = new JobRegistryBeanPostProcessor();
        postProcessor.setJobRegistry(this.jobRegistry);
        return postProcessor;
    }

    @Bean
    public BatchConfigurer batchConfigurer() {
        return new AsynchronousBatchConfigurer();
    }

    @Bean
    public Job operatorJob() {
        return this.jobBuilderFactory
                .get("operatorJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .build();
    }
}
```  

- `MetaDataJobOperatorTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        JobOperatorJobConfig.class
})
public class MetaDataJobOperatorTest {
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobOperator jobOperator;
    @Autowired
    private JobExplorer jobExplorer;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void getJobNames() throws Exception {
        // when
        Set<String> actual = this.jobOperator.getJobNames();

        // then
        assertThat(actual, hasItem("operatorJob"));
    }

    @Test
    public void start() throws Exception {
        // when
        long actual = this.jobOperator.start("operatorJob", "key");

        // then
        Thread.sleep(1000);
        JobExecution jobExecution = this.jobExplorer.getJobExecution(actual);
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }

    @Test
    public void getParameters() throws Exception {
        // given
        JobParameters jobParameters = new JobParametersBuilder()
                .addLong("key", 100L)
                .toJobParameters();
        long executionId = this.jobOperator.start("operatorJob", jobParameters.toString());

        // when
        String actual = this.jobOperator.getParameters(executionId);

        // then
        assertThat(actual, is("{key=100}"));
        this.jobOperator.stop(executionId);
    }

    @Test
    public void stop() throws Exception {
        // given
        UpperCaseItemProcessor.SLEEP_TIME = 1000000L;
        long executionId = this.jobOperator.start("operatorJob", "key");

        // when
        boolean actual = this.jobOperator.stop(executionId);

        // then
        Thread.sleep(1000);
        assertThat(actual, is(true));
        JobExecution jobExecution = this.jobExplorer.getJobExecution(executionId);
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.STOPPED.getExitCode()));
        UpperCaseItemProcessor.SLEEP_TIME = 0L;
    }

    @Test
    public void getRunningExecutions() throws Exception {
        // given
        UpperCaseItemProcessor.SLEEP_TIME = 1000000L;
        long executionId = this.jobOperator.start("operatorJob", "key");
        Thread.sleep(100);

        // when
        Set<Long> actual = this.jobOperator.getRunningExecutions("operatorJob");

        // then
        assertThat(actual, hasSize(1));
        assertThat(actual, hasItem(executionId));
        UpperCaseItemProcessor.SLEEP_TIME = 0L;
        this.jobOperator.stop(executionId);
    }

}
```  

### JobParametersIncrementer
`JobParametersIncrementer` 는 `JobOperator` 에서 `startNextInstance()` 메소드를 사용해서 항상 새로운 `Job` 을 실행할 수 있다. 
실패하더라도 매번 새로운 `JobInstance` 를 생성해서 실행해야 하는 경우 사용할 수 있다. 

```java
public interface JobParametersIncrementer {

    JobParameters getNext(JobParameters parameters);

}
```  

`getNext()` 메소드에서 이전 `JobParameters` 를 기준으로 중복되지 않는 다음 `JobParameters` 를 생성하면 로직을 작성해주면 된다. 

- `SampleIncrementer`

```java
public class SampleIncrementer implements JobParametersIncrementer {
    @Override
    public JobParameters getNext(JobParameters parameters) {
        if(parameters == null || parameters.isEmpty()) {
            return new JobParametersBuilder().addLong("num", 1L).toJobParameters();
        }

        long num = parameters.getLong("num", Long.valueOf(1)) + 1;
        return new JobParametersBuilder().addLong("num", num).toJobParameters();
    }
}
```  

- `IncrementerJobConfig`

```java
@Configuration
@Import(StepConfig.class)
@RequiredArgsConstructor
public class IncrementerJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final Step prefixStep;
    private final Step upperCaseStep;
    private final JobRegistry jobRegistry;

    @Bean
    public JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor() {
        JobRegistryBeanPostProcessor postProcessor = new JobRegistryBeanPostProcessor();
        postProcessor.setJobRegistry(this.jobRegistry);
        return postProcessor;
    }

    @Bean
    public Job incrementerJob() {
        return this.jobBuilderFactory
                .get("incrementerJob")
                .start(this.prefixStep)
                .next(this.upperCaseStep)
                .incrementer(new SampleIncrementer())
                .build();
    }
}
```  

`Job` 설정에서 `incremnt()` 메소드에 앞서 정의한 `SampleIncrementer` 객체를 설정해주면 된다.  

- `IncrementerJobTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        IncrementerJobConfig.class
})
public class IncrementerJobTest {
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobOperator jobOperator;

    @Before
    public void setUp() throws Exception {
        this.jobRepositoryTestUtils.removeJobExecutions();
        Util.deleteAllBatchFile();
    }

    @Test
    public void startNextInstance_First() throws Exception {
        // given
        long executionId  = this.jobOperator.startNextInstance("incrementerJob");

        // when
        String actual = this.jobOperator.getParameters(executionId);

        // then
        assertThat(actual, is("num(long)=1"));
    }

    @Test
    public void startNextInstance_Twice() throws Exception {
        // given
        this.jobOperator.startNextInstance("incrementerJob");
        long executionId  = this.jobOperator.startNextInstance("incrementerJob");

        // when
        String actual = this.jobOperator.getParameters(executionId);

        // then
        assertThat(actual, is("num(long)=2"));
    }
}
```  

### Stopping Job
`JobOperator` 의 `stop()` 메소드를 사용하면 실행 중인 `Job` 을 `Gracefully` 하게 중지 가능하다. 
하지만 `stop()` 메소드는 `Job` 을 바로 종료하지는 않는다. 
비지니스 로직을 처리중인 `Job` 에서 `Spring Batch` 가 제어권을 가져오게 될때 `StepExecution` 의 상태를 `BatchStatus.STOPPED` 로 변경하고, 
`JobExecution` 에서도 동일한 처리를 하게 된다.  

### Aborting Job
만약 `Job` 의 상태가 `ABANDONED` 라면 재시작이 가능한 설정이라도 재실행이 불가능하다. 
`Job` 재실행 시 특정 `Step` 을 스킵하고 싶을 때도 `ABANDONED` 상태를 사용할 수 있다. 
만약 이전에 실패한 `Job` 이 실행 중일 때 `ANBANDONED` 인 `Step` 을 만나게 되면 해당 `Step` 은 넘어가고 다음 `Step` 을 실행하게 된다.  


---
## Reference
[Configuring and Running a Job](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/job.html#configureJob)  