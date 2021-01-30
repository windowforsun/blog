--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch Item기반(Chunk) Step 설정과 실행"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Batch 에서 Chunk-oriented Processing 기반 Step 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Job
    - Chunk
    - Listener
    - Step
toc: true
use_math: true
---  

## Chunk-oriented Processing
`Spring Batch` 에서 `Step` 은 `Job` 이 가질 수 있는 독립직인 흐름을 캡슐화한 객체라고 할 수 있다. 
복잡한 `Job` 을 세분화된 여러 `Step` 으로 분리해서 `Step` 단위로 모호한 부분들 보다 정확하게 풀어 낼 수 있다. 
`Step` 에서 처리되는 로직은 타겟이 되는 비지니스에 크게 의존하기 때문에 굉장히 복잡해 질 수도 아주 간단할 수도 있다. 
다시 한번 `Step` 이 가지는 구성요소를 나타내면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-chunk-oriented-processing-1.png)  

위 그림 처럼 `Step` 이 가지는 구성요소인 `ItemReader`, `ItemProcessor`, `ItemWriter` 는 
`Chunk-oriented`(청크 지향)을 기반으로 한다. 
여기서 청크 지향이란 `Step` 에서 처리 단위인 `item` 의 묶음이라고 할 수 있다.  

`ItemReader` 가 `item` 하나를 읽으면 데이터는 `ItemProcessor` 로 넘겨지게 된다. 
그리고 `ItemWriter` 로 넘어올때는 수행된 `item` 의 수가 `commit-inverval` 의 수가 됐을 때 한번에 넘어온다. 
그러면 `ItemWriter` 는 넘어온 `chunk`(`item` 의 묶음) 전체에 대해서 쓰기를 수행하고 트랜잭션이 커밋 된다. 
위와 같은 과정을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/concept-batch-chunk-oriented-processing-2.png.png)  

## 예제 프로젝트
`Spring Batch` 에서 `Chunk-oriented Processing` 을 기반으로 하는 `Step` 을 설정하고 실행하는 예제 프로젝트를 간단히 소개하며 시작한다.  

### 프로젝트 구조

```
│  build.gradle
└─src
    ├─main
    │  ├─java
    │  │  │
    │  │  └─com
    │  │      └─windowforsun
    │  │          └─stepexam
    │  │              │  StepExamApplication.java
    │  │              │
    │  │              ├─component
    │  │              │      SampleChunkListener.java
    │  │              │      SampleItemProcessorListener.java
    │  │              │      SampleItemReadListener.java
    │  │              │      SampleItemWriteListener.java
    │  │              │      SampleJobListener.java
    │  │              │      SampleSkipListener.java
    │  │              │      SampleStepListener.java
    │  │              │
    │  │              ├─config
    │  │              │      InterceptingConfig.java
    │  │              │      ItemConfig.java
    │  │              │      ItemStreamConfig.java
    │  │              │      NoRollbackConfig.java
    │  │              │      RestartCompleteStepConfig.java
    │  │              │      RetryLogicConfig.java
    │  │              │      SampleJobConfig.java
    │  │              │      SkipLogicConfig.java
    │  │              │      StartLimitConfig.java
    │  │              │      StartLimitDuplicatedStepConfig.java
    │  │              │
    │  │              ├─domain
    │  │              │      NumberData.java
    │  │              │      PrefixStringData.java
    │  │              │      StringData.java
    │  │              │
    │  │              ├─exception
    │  │              │      CustomException.java
    │  │              │      MySkipException.java
    │  │              │
    │  │              ├─item
    │  │              │      CustomNumberDataReader.java
    │  │              │      CustomStringDataReader.java
    │  │              │      FailCountNumberDataProcessor.java
    │  │              │      FailCountStringDataProcessor.java
    │  │              │      LogItemProcessor.java
    │  │              │      LogItemReader.java
    │  │              │      LogItemWriter.java
    │  │              │      NumberDataItemStream.java
    │  │              │      NumberDataProcessor.java
    │  │              │      SampleStream.java
    │  │              │      StringDataProcessor.java
    │  │              │
    │  │              └─tmp
    │  └─resources
    │          application.yaml
    │          number-data.csv
    │          schema-all.sql
    │          string-data.csv
    │
    └─test
        └─java
          │
          └─com
              └─windowforsun
                  └─stepexam
                          BatchTestConfig.java
                          CommitIntervalTest.java
                          InterceptingTest.java
                          ItemStreamTest.java
                          NoRollbackTest.java
                          RestartCompleteStepTest.java
                          RetryLogicTest.java
                          SkipLogicTest.java
                          StartLimitDuplicatedStepTest.java
                          StartLimitTest.java

```  

### build.gradle

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
    runtimeOnly 'com.h2database:h2'
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

### StepExamApplication

```java
@SpringBootApplication
@EnableBatchProcessing
public class StepExamApplication {
    public static void main(String[] args) {
        System.exit(
                SpringApplication.exit(
                        SpringApplication.run(StepExamApplication.class, args)
                )
        );
    }
}
```  

### ItemConfig

```java
@Configuration
@RequiredArgsConstructor
public class ItemConfig {
    private final DataSource dataSource;

    @Bean
    public ItemReader<NumberData> numberReader() {
        return new FlatFileItemReaderBuilder<NumberData>()
                .name("numberReader")
                .resource(new ClassPathResource("number-data.csv"))
                .delimited()
                .names("numVal")
                .fieldSetMapper(new BeanWrapperFieldSetMapper<NumberData>() {{
                    setTargetType(NumberData.class);
                }})
                .build();
    }

    @Bean
    public ItemProcessor<NumberData, NumberData> numberProcessor() {
        return new NumberDataProcessor();
    }

    @Bean
    public ItemWriter<NumberData> numberWriter() {
        return new JdbcBatchItemWriterBuilder<NumberData>()
                .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
                .sql("INSERT INTO number_data (num_val) VALUES (:numVal)")
                .dataSource(this.dataSource)
                .build();
    }

    @Bean
    public ItemReader<StringData> stringReader() {
        return new FlatFileItemReaderBuilder<StringData>()
                .name("stringReader")
                .resource(new ClassPathResource("string-data.csv"))
                .delimited()
                .names("strVal ")
                .fieldSetMapper(new BeanWrapperFieldSetMapper<StringData>() {{
                    setTargetType(StringData.class);
                }})
                .build();
    }

    @Bean
    public ItemProcessor<StringData, StringData> stringProcessor() {
        return new StringDataProcessor();
    }

    @Bean
    public ItemWriter<StringData> stringWriter() {
        return new JdbcBatchItemWriterBuilder<StringData>()
                .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
                .sql("INSERT INTO string_data (str_val) VALUES (:strVal)")
                .dataSource(this.dataSource)
                .build();
    }
}
```  

### BatchTestConfig

```java
@EnableBatchProcessing
@EnableAutoConfiguration
public class BatchTestConfig {
}
```  

### resources
- `number-data.csv`

```
1000
2000
3000
4000
5000
6000
7000
8000
9000
10000
```  

- `string-data.csv`

```
aa
bb
cc
dd
ee
ff
gg
hh
ii
jj
```  

- `schema-all.sql`

```sql
DROP TABLE IF EXISTS `number_data`;

CREATE TABLE number_data  (
    id BIGINT auto_increment NOT NULL PRIMARY KEY,
    num_val int
);

DROP TABLE IF EXISTS `string_data`;

CREATE TABLE string_data  (
    id BIGINT auto_increment NOT NULL PRIMARY KEY,
    str_val varchar
);

DROP TABLE IF EXISTS `prefix_string_data`;

CREATE TABLE prefix_string_data  (
    id BIGINT auto_increment NOT NULL PRIMARY KEY,
    prefix_str_val varchar
);
```  






## Step 구성하기
`Step` 은 `Spring Batch` 에서 기본으로 제공하는 `SimpleStepBuilder` 를 사용해서 구성할 수 있다. 
실제로 사용할때는 `StepBuilderFactory` 를 통해 간편하게 생성할 수 있다. 

- `SampleJobConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class SampleJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step numberStep(ItemReader<NumberData> numberReader,
                           ItemProcessor<NumberData, NumberData> numberProcessor,
                           ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(numberReader)
                .processor(numberProcessor)
                .writer(numberWriter)
                .build();
    }

    @Bean
    public Step stringStep(ItemReader<StringData> stringReader,
                           ItemProcessor<StringData, StringData> stringProcessor,
                           ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(stringProcessor)
                .writer(stringWriter)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStep, Step stringStep) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStep)
                .next(stringStep)
                .build();
    }
}
```  

`Step` 을 구성할때 필수적인 의존성은 아래와 같다. 
- `reader` : 처리할 아이템을 제공하는 `ItemReader`
- `writer` : `ItemReader` 가 제공하는 아이템에 대한 쓰기 동작을 수행하는 `ItemWriter`
- `transactionManager` : 트랜잭션을 시작하고 커밋하는 `Spring` 의 `PlatformTransactionManager`
- `repository` : `StepExecution` 과 `ExecutionContext` 를 주기적으로(커밋 직전) 저장하는 `JobRepository`
- `chunk` : 트랜잭션을 커밋하기 전까지 처리할 아이템 수 

위 자바 설정 코드를 보면 `transactionManager` 와 `repository` 에 대한 의존성이 보이지 않는다. 
`@EnableBatchProcessing` 을 선언한 경우 기본으로 두 의존성에 대한 빈이 제공되기 때문이다. 
만약 커스컴한 의존성을 설정하고 싶다면 별도로 설정을 통해 가능하다. 
그리고 `Job` 의 목적이 단순히 특정 부분을 읽고 다른 곳에 쓰는 동작을 수행할 수도 있기 때문에, 
`ItemProcessor` 는 옵션이다. 


## The Commit Interval
앞서 언급한 것과 같이 `Step` 은 아이템을 읽고, 쓰는 동안 설정된 `PlatformTransacitonManager` 를 사용해서 주기적으로 커밋하게 된다. 
만약 `comiit-interval` 가 `1`이면 아아템 한개단위로 읽고, 쓰고, 커밋을 수행한다. 
매번 아이템 한개 단위로 커밋을 반복하는 것은 비효율적이기 때문에, 
수행하는 `Step` 의 전체적인 상황에 맞춰 `commit-interval` 에 대한 설정이 필요하다.  

`SampleJobConfig` 에서 `numberStep`, `stringStep` 은 모두 `commit-interval` 가 2인 설정이다. 
설정 파일을보면 `chunk(2)` 라고 설정된 것을 확인 할 수 있다. 
이는 아이템을 2개 단위로 처리한다는 의미로 2개의 아이템을 하나의 청크로 묶어 전체 `Step` 이 처리된다. 
`ItemReader` 에서 `read` 메소드가 호출 될때마다 카운트가 증가하고, `ItemProcessor,` `ItemWriter` 까지 
청크단위로 전달되고 수행된다. 
최종적으로 `ItemWriter` 에서 청크에 대한 모든 수행이 완료되면 커밋되고, 이는 `ItemReader`, `ItemProcessor`, `ItemWriter` 의 
청크 하나의 처리가 완료 된것을 의미할 수 있다.  

## Configuring a Step for Restart

### Setting a Start Limit
`startLimit` 설정을 사요앟면 `Job` 내에서 `Step` 의 실행 횟수를 지정할 수 있다. 
여기서 `Step` 의 실행 횟수란 `JobInstance` 단위에서 횟수를 의미한다. 
만약 `JobInstane` 단위에서 `Step` 에 설정된 `startLimit` 를 넘는 실행횟수가 수행되면 `StartLimitExceededException` 예외가 발생한다. 
`startLimit` 의 기본값은 `Integer.MAX_VALUE` 이다.  

테스트는 하나의 `Job` 이 중복되지 않은 `Step` 을 하나씩 실행할때 `Job` 을 동일 `JobParameters` 로 여러번 수행하는 것과 
하나의 `Job` 에서 중복되는 `Step` 을 여러번 실행할 떄의 상황으로 진행한다.  

- `StartLimitConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class StartLimitConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step numberStepStartLimit1(ItemReader<NumberData> numberReader, ItemProcessor<NumberData, NumberData> numberProcessor, ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(numberReader)
                .processor(numberProcessor)
                .writer(numberWriter)
                .startLimit(1)
                .build();
    }

    @Bean
    public Step stringStepStartLimit2(ItemReader<StringData> stringReader, ItemProcessor<StringData, StringData> stringProcessor, ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(stringProcessor)
                .writer(stringWriter)
                .startLimit(2)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStepStartLimit1, Step stringStepStartLimit2) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStepStartLimit1)
                .next(stringStepStartLimit2)
                .build();
    }
}
```  
  - `numberStepStartLimit1` 은 `startLimit` 가 1로 설정되었기 때문에, 동일한 `JobExecution` 에서 2번째 실행에서 예외가 발생한다. 
  - `stringStepStartLimit2` 은 `startLimit` 가 2로 설정되었기 때문에, 동일한 `JobExecution` 에서 3번째 실행에서 예외가 발생한다. 

- `StartLimitTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        StartLimitConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_CLASS)
public class StartLimitTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Before
    public void setUp() {
        NumberDataProcessor.FAIL_NUMBER = 0;
        StringDataProcessor.FAIL_STRING = "";
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenSameJobParameters_whenMultipleLaunchJob_thenNumberDataRowCount10_StringDataRowCount2_ThrowStartLimitExceededException() throws Exception {
        // given
        JobParameters jobParameters = new JobParametersBuilder().addLong("key", 11L).toJobParameters();
        StringDataProcessor.FAIL_STRING = "cc";
        this.jobLauncherTestUtils.launchJob(jobParameters);
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        StringDataProcessor.FAIL_STRING = "";
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(2));
        assertThat(this.capture.getOut(), containsString("org.springframework.batch.core.StartLimitExceededException: Maximum start limit exceeded for step: stringStepStartMax: 2"));
    }
}
```  
  - 첫 번재, 두번 째 실행에서는 `cc` 에서 실패하고, 세번째 실행에서는 `StartLimitExceededException` 로 실패하기 때문에 `stringData` 는 2개 밖에 들어가지 않는다. 
  - `numberStepStartLimit1` 스텝은 첫번째 시행에서 성공하기 때문에 이후 `Job` 실행에서는 수행되지 않는다. 

- `StartLimitDuplicatedStepConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class StartLimitDuplicatedStepConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step numberStepStartLimit1(ItemReader<NumberData> numberReader, ItemProcessor<NumberData, NumberData> numberProcessor, ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(numberReader)
                .processor(numberProcessor)
                .writer(numberWriter)
                .startLimit(1)
                .build();
    }

    @Bean
    public Step stringStepStartLimit2(ItemReader<StringData> stringReader, ItemProcessor<StringData, StringData> stringProcessor, ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(stringProcessor)
                .writer(stringWriter)
                .startLimit(2)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStepStartLimit1, Step stringStepStartLimit2) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStepStartLimit1)
                .next(stringStepStartLimit2)
                .next(stringStepStartLimit2)
                .next(stringStepStartLimit2)
                .build();
    }
}
```  
  - `sampleJob` 은 동일한 `stringStepStartLimit2` 가 3번 실행 하도록 구성돼있다. 

- `StartLimitDuplicatedStepTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        StartLimitDuplicatedStepConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_CLASS)
public class StartLimitDuplicatedStepTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Before
    public void setUp() {
        NumberDataProcessor.FAIL_NUMBER = 0;
        StringDataProcessor.FAIL_STRING = "";
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenJobDuplicatedStep_whenLaunchJob_thenNumberDataRowCount10_StringDataRowCount20_ThrowStartLimitExceededException() throws Exception {
        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(20));
        assertThat(this.capture.getOut(), containsString("org.springframework.batch.core.StartLimitExceededException: Maximum start limit exceeded for step: stringStepStartMax: 2"));
    }
}
```  

### Restarting a Completed Step
만약 `Job` 이 재시작 가능할때, 재시작에 대한 부분을 `Step` 단위로 본다면 매번 실행이 필요한 `Step` 이 존재할 수 있다. 
이전 실행에서 성공해서 `Step` 상태가 `COMPLETED` 라도 `allowStartIfComplet` 를 사용하면 해당 `Step` 은 항상 재시작 된다. 

- `RestartCompleteStepConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class RestartCompleteStepConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step numberStepAllowStartIfCompleteTrue(ItemReader<NumberData> numberReader, ItemProcessor<NumberData, NumberData> numberProcessor, ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(numberReader)
                .processor(numberProcessor)
                .writer(numberWriter)
                .allowStartIfComplete(true)
                .build();
    }

    @Bean
    public Step stringStep(ItemReader<StringData> stringReader, ItemProcessor<StringData, StringData> stringProcessor, ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(stringProcessor)
                .writer(stringWriter)
                .allowStartIfComplete(true)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStepAllowStartIfCompleteTrue, Step stringStep) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStepAllowStartIfCompleteTrue)
                .next(stringStep)
                .build();
    }
}
```  

- `RestartCompleteStepTest`
 
```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        RestartCompleteStepConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_CLASS)
public class RestartCompleteStepTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Before
    public void setUp() {
        NumberDataProcessor.FAIL_NUMBER = 0;
        StringDataProcessor.FAIL_STRING = "";
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenSameJobParametersProcessorFailLaunchJob_whenNextLaunchJobSuccess_thenNumberDataRow20_StringDataRow10() throws Exception {
        // given
        JobParameters jobParameters = new JobParametersBuilder().addLong("my", 100L).toJobParameters();
        StringDataProcessor.FAIL_STRING = "aa";
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // when
        StringDataProcessor.FAIL_STRING = "";
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(20));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(10));
    }
}
```  

## Configuring Skip Logic
`Step` 에서 지정된 로직을 처리하던 중 특정 예외가 발생하더라도 무시하고 진행해야 하는 경우가 발생할 수 있다. 
이때 `faultTolerant` 에서 `skip` 과 `skipLimit` 를 사용하면 이러한 상황에 대한 대처가 가능하다.  

그리고 특정 예외를 스킵하는 것이 아니라, 특정 예외가 발생했을 떄만 `Step` 을 실패하도록 설정이 필요할 수 있다. 
이때는 `skip` 설정과 함께 `noSkip` 설정을 사용하면 대처가 가능하다.  

- `SkipLogicConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class SkipLogicConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public ItemReader<NumberData> customNumberReader() {
        return new CustomNumberDataReader();
    }

    @Bean
    public ItemReader<StringData> customStringReader() {
        return new CustomStringDataReader();
    }

    @Bean
    public Step numberStepSkipCustomException2(ItemProcessor<NumberData, NumberData> numberProcessor,
                           ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(this.customNumberReader())
                .processor(numberProcessor)
                .writer(numberWriter)
                .faultTolerant()
                .skipLimit(2)
                .skip(CustomException.class)
                .build();
    }

    @Bean
    public Step stringStepSkipExceptionNoSkipCustomException(ItemProcessor<StringData, StringData> stringProcessor,
                           ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(this.customStringReader())
                .processor(stringProcessor)
                .writer(stringWriter)
                .faultTolerant()
                .skipLimit(1)
                .skip(RuntimeException.class)
                .noSkip(CustomException.class)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStepSkipCustomException2, Step stringStepSkipExceptionNoSkipCustomException) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStepSkipCustomException2)
                .next(stringStepSkipExceptionNoSkipCustomException)
                .build();
    }
}
```  
  - `numberStepSkipCustomException2` 스텝은 `CustomException` 이 발생하더라도 2번까지는 `skip` 한다. 
  - `stringStepSkipExceptionNoSkipCustomException` 스텝은 `RuntimeException` 발생은 1번까지 `skip` 하고, `CustomException` 발생하면 스텝은 실패한다. 

- `SkipLogicTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        SkipLogicConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class SkipLogicTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Before
    public void setUp() {
        CustomNumberDataReader.INDEX = 0;
        CustomStringDataReader.INDEX = 0;
        CustomNumberDataReader.DATAS = new int[] {
                1000, 2000, 3000, 4000, 5000, 6000, 7000, 8000, 9000, 10000
        };
        CustomStringDataReader.DATAS = new String[] {
                "aa", "bb", "cc", "dd", "ee", "ff", "gg", "hh", "ii", "jj"
        };
        NumberDataProcessor.FAIL_NUMBER = 0;
        StringDataProcessor.FAIL_STRING = "";
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenFailNumber1000AndOneCustomException_whenLaunchJob_thenCompletedAndNumberDataRow9_StringDataRow10() throws Exception {
        // given
        NumberDataProcessor.FAIL_NUMBER = 1000;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(9));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(10));
    }

    @Test
    public void givenFailNumber1000AndTowCustomException_whenLaunchJob_thenCompletedAndNumberDataRow8_StringDataRow10() throws Exception {
        // given
        NumberDataProcessor.FAIL_NUMBER = 1000;
        CustomNumberDataReader.DATAS = new int[]{
                1000, 1000, 2000, 3000, 4000, 5000, 6000, 7000, 8000, 9000
        };

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(8));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(10));
    }

    @Test
    public void givenFailNumber1000AndThreeCustomException_whenLaunchJob_thenFailedAndNumberDataRow0_StringDataRow0() throws Exception {
        // given
        NumberDataProcessor.FAIL_NUMBER = 1000;
        CustomNumberDataReader.DATAS = new int[]{
                1000, 1000, 1000, 2000, 3000, 4000, 5000, 6000, 7000, 8000
        };

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(0));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(0));
    }

    @Test
    public void givenFailStringaaAndOneCustomException_whenLaunchJob_thenFailedAndNumberDataRow10_StringDataRow0() throws Exception {
        // given
        StringDataProcessor.FAIL_STRING = "aa";
        CustomStringDataReader.DATAS = new String[] {
                "aa", "bb", "cc", "dd", "ee", "ff", "gg", "hh", "ii", "jj"
        };

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(0));
    }

    @Test
    public void giveOneNullPointException_whenLaunchJob_thenCompletedAndNumberDataRow10_StringDataRow9() throws Exception {
        // given
        CustomStringDataReader.DATAS = new String[] {
                null, "bb", "cc", "dd", "ee", "ff", "gg", "hh", "ii", "jj"
        };

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(9));

    }
}
```  

## Configuring Retry Logic
예외가 발생하는 경우 이를 무시하는 동작으로 대부분 로직에 대한 대처는 가능할 것이다. 
하지만 대표적으로 리소스에 대한 예외가 발생한 경우, 이는 리소스 사용량이 다시 좋아진다면 이후 재실행을 할때 성공할 수 도 있다. 
이런 상황에서 `faulTolerant` 에서 `retry` 와 `retryLimit` 를 사용하면 재시작에 대한 횟수 지정과 재시작을 수행할 예외를 지정 할 수 있다. 
만약 `retryLimit` 를 넘어가게 되면 `RetryException` 이 발생한다.  

- `RetryLogicConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class RetryLogicConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public ItemProcessor<NumberData, NumberData> failCountNumberProcessor() {
        return new FailCountNumberDataProcessor();
    }

    @Bean
    public ItemProcessor<StringData, StringData> failCountStringProcessor() {
        return new FailCountStringDataProcessor();
    }

    @Bean
    public Step numberStep(ItemReader<NumberData> numberReader,
                           ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(numberReader)
                .processor(this.failCountNumberProcessor())
                .writer(numberWriter)
                .faultTolerant()
                .retryLimit(2)
                .retry(CustomException.class)
                .build();
    }

    @Bean
    public Step stringStep(ItemReader<StringData> stringReader,
                           ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(this.failCountStringProcessor())
                .writer(stringWriter)
                .faultTolerant()
                .retryLimit(1)
                .retry(CustomException.class)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStep, Step stringStep) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStep)
                .next(stringStep)
                .build();
    }
}
```  

- `FailCountNumberDataProcessor`

```java
public class FailCountNumberDataProcessor implements ItemProcessor<NumberData, NumberData> {
    public static int FAIL_COUNT = 0;

    @Override
    public NumberData process(NumberData item) throws Exception {
        if(FAIL_COUNT > 0) {
            FAIL_COUNT--;
            throw new CustomException("number processor fail");
        }

        return item;
    }
}
```  

- `FailCountStringDataProcessor`

```java
public class FailCountStringDataProcessor implements ItemProcessor<StringData, StringData> {
    public static int FAIL_COUNT = 0;

    @Override
    public StringData process(StringData item) throws Exception {
        if(FAIL_COUNT > 0) {
            FAIL_COUNT--;
            throw new CustomException("string processor fail");
        }

        return item;
    }
}
```

- `RetryLogicTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        RetryLogicConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class RetryLogicTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        FailCountNumberDataProcessor.FAIL_COUNT = 0;
        FailCountStringDataProcessor.FAIL_COUNT = 0;
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenFailCountNumberProcessorFailCount1_whenLaunchJob_thenCompletedAndNumberDataRow10_StringDataRow10() throws Exception {
        // given
        FailCountNumberDataProcessor.FAIL_COUNT = 1;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(10));
    }

    @Test
    public void givenFailCountNumberProcessorFailCount2_whenLaunchJob_thenFailedAndNumberDataRow0_StringDataRow0_ThrowRetryException() throws Exception {
        // given
        FailCountNumberDataProcessor.FAIL_COUNT = 2;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(0));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(0));
        assertThat(this.capture.getOut(), containsString(
                "org.springframework.retry.RetryException"
        ));
    }

    @Test
    public void givenFailCountStringProcessorFailCount1_whenLaunchJob_thenFailedAndNumberDataRow10_StringDataRow0_ThrowRetryException() throws Exception {
        // given
        FailCountStringDataProcessor.FAIL_COUNT = 1;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(0));
        assertThat(this.capture.getOut(), containsString(
                "org.springframework.retry.RetryException"
        ));
    }
}
```  

`retry` 에 대한 더 자세한 설명은 [여기](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/retry.html#retry)
에서 확인 할 수 있다.  

## Controlling Rollback
재시도와 상관없이 `ItemWriter` 에서 발생하는 예외는 기본적으로 `Step` 에서 관리하는 트랜잭션을 롤백 시킨다. 
스킵 설정또한 이러한 트랜잭션을 회피할 수 있는 방법중 하나이다. 
하지만 별도의 설정으로 트랜잭션을 롤백을 회피해야 하는 경우 `faulTolerant` 에서 `noRollback` 에 특정 예외를 지정해서 가능하다.  

### Transactional Readers
ItemReader는 데이터를 읽을 때 기본적으로 앞에서 뒤로만 읽고 역행하지 않는다 (forward only). step은 데이터를 읽고 나면 버퍼에 넣어두기 때문에 롤백되었을 때 한 번 읽어들인 데이터를 다시 읽어올 필요는 없다. 하지만 어떤 경우에는 reader가 트랜잭션 리소스보다 상위 레벨에 있을 수도 있다 (e.g JMS 큐). 이런 경우 큐가 롤백되는 트랜잭션과 엮여있기 때문에 큐로부터 읽어온 메세지는 여전히 큐에 남아있다. 이런 이유로 아래 예제처럼 step이 버퍼를 사용하지 않도록 설정할 수 있다:

```java
@Bean
public Step step1() {
	return this.stepBuilderFactory.get("step1")
				.<String, String>chunk(2)
				.reader(itemReader())
				.writer(itemWriter())
				.readerIsTransactionalQueue()
				.build();
}
```  

## Transaction Attributes
트랜잭션 속성값으로 트랜잭션 고립 수준(isolation), 전파(propagation), 타임아웃을 설정할 수 있다. 트랜잭션 속성에 대한 자세한 설명은 스프링 코어 문서을 참고하라. 아래 예제에서는 고립 수준, 전파, 타임아웃을 설정한다:

```java
@Bean
public Step step1() {
	DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();
	attribute.setPropagationBehavior(Propagation.REQUIRED.value());
	attribute.setIsolationLevel(Isolation.DEFAULT.value());
	attribute.setTimeout(30);

	return this.stepBuilderFactory.get("step1")
				.<String, String>chunk(2)
				.reader(itemReader())
				.writer(itemWriter())
				.transactionAttribute(attribute)
				.build();
}
```  

## Registering ItemStream with a Step
`ItemStream` 은 `Step` 의 생명주기 동안 정의된 지점에서 콜백을 처리할 수 있다. 
한가지 예로 커스텀한 `ItemReader` 또는 `ItemWriter` 를 구현해야 할때, 
`ItemStream` 에 있는 `open()`, `update()`, `close()` 메소드를 사용해서 리소스 접근에(`Connection`) 대한 전처리, 중간처리, 후처리를 수행할 수 있다. 
그리고 커스텀한 재시작 룰을 적용하고 싶은 경우 `ItemStream` 에서 `ExecutionContext` 를 활용해서 처리 가능하다.  

`ItemStream` 을 구성하는 3개의 메소드의 간단한 설명은 아래와 같다. 
- `open()` : `Step` 이 시작할때 한번 호출 된다. 
- `update()` : `chunk` 단위가 시작할 떄마다 호출 된다. 
- `close()` : `Step` 종료 될떄 한번 호출 된다. 

`ItemStream` 에 대한 더 자세한 내용은 [여기](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/readersAndWriters.html#itemStream)
에서 확인 할 수 있다.  

테스트에서는 `numberStep` 에 커스텀한 `ItemReader`(`CustomNumberDataReader`) 를 사용할떄, 
`ItemStream` 을 사용해서 재시작에 대한 처리를 적용해 본다.  

- `ItemStreamConfig`

```java
@Configuration
@Import(ItemConfig.class)
@RequiredArgsConstructor
public class ItemStreamConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step numberStep(ItemProcessor<NumberData, NumberData> numberProcessor,
                           ItemWriter<NumberData> numberWriter) {
        return this.stepBuilderFactory
                .get("numberStep")
                .<NumberData, NumberData>chunk(2)
                .reader(new CustomNumberDataReader())
                .processor(numberProcessor)
                .writer(numberWriter)
                .stream(new NumberDataItemStream())
                .build();
    }

    @Bean
    public Step stringStep(ItemReader<StringData> stringReader,
                           ItemProcessor<StringData, StringData> stringProcessor,
                           ItemWriter<StringData> stringWriter) {
        return this.stepBuilderFactory
                .get("stringStep")
                .<StringData, StringData>chunk(2)
                .reader(stringReader)
                .processor(stringProcessor)
                .writer(stringWriter)
                .build();
    }

    @Bean
    public Job sampleJob(Step numberStep, Step stringStep) {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(numberStep)
                .next(stringStep)
                .build();
    }
}
```  

- `NumberDataItemStream`

```java
@Slf4j
public class NumberDataItemStream implements ItemStream {
    private int index;

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        if(executionContext.containsKey("index")) {
            this.index = executionContext.getInt("index");
        } else {
            this.index = 0;
        }

        CustomNumberDataReader.INDEX = this.index;
        log.info("open index : " + this.index);
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        this.index = CustomNumberDataReader.INDEX;
        executionContext.putInt("index", this.index);
        log.info("update index : " + this.index);
    }

    @Override
    public void close() throws ItemStreamException {
        log.info("close");
    }
}
```  
  - `open()` 이 호출됐을 떄 `ExecutionContext` 에서 인덱스 정보가 있으면 `ItemReader` 에 설정하고, 아니면 초기화 한다. 
  - `update()` 가 호출 될 때마다 `ExecutionContext` 에 인덱스 정보를 갱신한다. 
  
- `ItemStreamTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        ItemStreamConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class ItemStreamTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Autowired
    private JobRepository jobRepository;
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        CustomNumberDataReader.INDEX = 0;
        NumberDataProcessor.FAIL_NUMBER = -1;
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void givenSameJobParametersFirstFailed_whenSuccessLaunchJob_thenRestartSuccess() throws Exception {
        // given
        NumberDataProcessor.FAIL_NUMBER = 5000;
        JobParameters jobParameters = new JobParametersBuilder().addLong("key", 100L).toJobParameters();
        this.jobLauncherTestUtils.launchJob(jobParameters);
        NumberDataProcessor.FAIL_NUMBER = -1;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        int actualNumberData = this.jdbcTemplate.queryForObject("select count(1) from number_data", Integer.class);
        assertThat(actualNumberData, is(10));
        int actualStringData = this.jdbcTemplate.queryForObject("select count(1) from string_data", Integer.class);
        assertThat(actualStringData, is(10));
        StepExecution stepExecution = this.jobRepository.getLastStepExecution(jobExecution.getJobInstance(), "numberStep");
        ExecutionContext executionContext = stepExecution.getExecutionContext();
        assertThat(executionContext.getInt("index"), is(10));
    }

    @Test
    public void givenSameJobParametersFirstFailed_whenSuccessLaunchJob_thenCheckLogging() throws Exception {
        // given
        CustomNumberDataReader.DATAS = new int[]{
                1000, 2000, 3000, 4000, 5000
        };
        NumberDataProcessor.FAIL_NUMBER = 3000;
        JobParameters jobParameters = new JobParametersBuilder().addLong("key", 100L).toJobParameters();
        this.jobLauncherTestUtils.launchJob(jobParameters);
        NumberDataProcessor.FAIL_NUMBER = -1;

        // when
        this.jobLauncherTestUtils.launchJob(jobParameters);

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "NumberDataItemStream.open index : 0",
                "NumberDataItemStream.update index : 0",
                "CustomNumberDataReader.read : 1000",
                "CustomNumberDataReader.read : 2000",
                "NumberDataProcessor.process : 1000",
                "NumberDataProcessor.process : 2000",
                "NumberDataItemStream.update index : 2",
                "CustomNumberDataReader.read : 3000",
                "CustomNumberDataReader.read : 4000",
                // when process 3000 throw exception
                "CustomException: number processor fail",
                "NumberDataItemStream.close",
                // restart jobinstance
                "NumberDataItemStream.open index : 2",
                "NumberDataItemStream.update index : 2",
                "CustomNumberDataReader.read : 3000",
                "CustomNumberDataReader.read : 4000",
                "NumberDataProcessor.process : 3000",
                "NumberDataProcessor.process : 4000",
                "NumberDataItemStream.update index : 4",
                "CustomNumberDataReader.read : 5000",
                "NumberDataProcessor.process : 5000",
                "NumberDataItemStream.close"
        ));
    }
}
```  


## Intercepting Step Execution
`Job` 과 동일하게 `Step` 또한 라이프 사이클 중 특정 이벤트에 따라 처리를 수행할 수 있다. 
`Step` 뿐만 아니라 구성요소인 `ItemReader`, `ItemProcessor`, `ItemWriter`, `Chunk`, `Skip` 등 
다양한 `Listener` 를 구현해서 `Step` 구성에 사용하는 `StepBuilderFactory` 를 통해 등록 가능하다.  

`Listener` 에 대한 구현은 각 인터페이스를 구현하고 등록하는 방법도 가능하고, 
어노테이션을 사용해서 구현하는 방법도 있다.  

### StepExecutionListener
`Step` 실행 전, 후에 대한 처리를 수행할 수 있는 리스너의 인터페이스는 아래와 같다. 

```java
public interface StepExecutionListener extends StepListener {

    void beforeStep(StepExecution stepExecution);

    ExitStatus afterStep(StepExecution stepExecution);

}
```  

두 메소드는 `Step` 의 성공, 실패 여부에 상관하지 않고 한번씩 실행된다. 
`afterStep` 에서는 `Step` 이 처리완료 되었지만 다시한번 검증 후 실패 `ExitStatus` 를 리턴하는 동작이 가능하다.  

`StepExecutionListener` 에 해당하는 어노테이션은 아래와 같다. 
- `@BeforeStep`
- `@AfterStep`

### ChunkListener
`Chunk` 실행 전, 후등에 처리를 수행할 수 있는 리스너의 인터페이스는 아래와 같다. 

```java
public interface ChunkListener extends StepListener {

    void beforeChunk(ChunkContext context);
    void afterChunk(ChunkContext context);
    void afterChunkError(ChunkContext context);

}
```  

- `beforeChunk` : 트랜잭션 시작된 후 이면서, `ItemReader` 의 `read` 메소드 호출 전에 호출된다. 
- `afterChunk` : 트랜잭션 커밋 후 호출된다. 만약 롤백된 경우에는 호출되지 않는다. 
- `afterChunkError` : `Chunk` 단위에서 예외가 발생한 경우 롤백 후 호출 된다. 

`ChunkListener` 는 `Tasklet` 기반 `Step` 에서도 사용될 수 있다. 
`Tasklet` 에서 사용될 때는 실행 전, 후에 호출 된다.  

`ChunkListener` 에 해당하는 어노테이션은 아래와 같다. 
- `@BeforeChunk`
- `@AfterChunk`
- `@AfterChunkError`

### ItemReadListener
`ItemReader` 실행 전, 후등에 처리를 수행할 수 있는 리스너의 인터페이스는 아래와 같다. 

```java
public interface ItemReadListener<T> extends StepListener {

    void beforeRead();
    void afterRead(T item);
    void onReadError(Exception ex);

}
```  

- `beforeReader` : `read` 메소드 호출 전에 호출된다. 
- `afterReader` : `read` 메소드 호출이 성공하고 나서 호출 된다. 
- `onReadError` : `read` 메소드에서 예외가 발생하면 호출 된다. 

`ItemReadListner` 에 해당하는 어노테이션은 아래와 같다. 
- `@BeforeRead`
- `@AfterRead`
- `@OnReadError`

### ItemProcessListener
`ItemProcessor` 실행 전, 후등에 처리를 수행할 수 있는 리스너의 인터페이스는 아래와 같다. 

```java
public interface ItemProcessListener<T, S> extends StepListener {

    void beforeProcess(T item);
    void afterProcess(T item, S result);
    void onProcessError(T item, Exception e);

}
```  

- `beforeProcess` : `process` 메소드 호출 전에 호출 된다. 
- `afterProcess` : `process` 메소드가 성공하고 나서 호출 된다. 
- `onProcessError` : `process` 메소드에서 예외가 발생하면 호출 된다. 

`ItemProcessListener` 에 해당하는 어노테이션은 아래와 같다. 
- `@BeforeProcess`
- `@AfterProcess`
- `@OnProcessError`

### ItemWriteListener
`ItemWriter` 실행 전, 후등에 처리를 수행할 수 있는 리스너의 인터페이스는 아래와 같다. 

```java
public interface ItemWriteListener<S> extends StepListener {

    void beforeWrite(List<? extends S> items);
    void afterWrite(List<? extends S> items);
    void onWriteError(Exception exception, List<? extends S> items);

}
```  

- `beforeWrite` : `write` 메소드 호출 전에 호출 된다. 
- `afterWrite` : `write` 메소드가 성공하고 나서 호출 된다. 
- `onWriteError` : `write` 메소드에서 예외가 발생하면 호출 된다. 

`ItemWriteListener` 에 해당하는 어노테이션은 아래와 같다. 
- `@BeforeWrite`
- `@AfterWrite`
- `@OnWriteError`

### SkipListener
앞서 살펴본 `skip` 설정을 하게 되면 `ItemReader`, `ItemProcessor`, `ItemWriter` 의 리스너에서는 스킵 여부는 알 수 없다. 
스킵 여부는 `SkipListener` 를 통해서만 여부를 판별 할 수 있고 그 인터페이스는 아래와 같다. 

```java
public interface SkipListener<T,S> extends StepListener {

    void onSkipInRead(Throwable t);
    void onSkipInProcess(T item, Throwable t);
    void onSkipInWrite(S item, Throwable t);

}
```  

- `onSkipInRead` : `ItemReader` 에서 스킵이 발생할 때마다 호출 된다. 
- `onSkipInProcess` : `ItemProcessor` 에서 스킵이 발생할 때마다 호출 된다. 
- `onSkipInWrite` : `ItemWriter` 에서 스킵이 발생할 때마다 호출 된다. 

`SkipListener` 는 아래 두가지 동작을 보장한다. 
- 하나의 `item` 스킵에 대해서 해당하는 하나의 `SkipListener` 메소드만 호출 된다. 
- `SkipListener` 는 트랜잭션 커밋 직전에 호출 된다. 
만약 `ItemWriter` 에서 예외 발생하더라도 리스너에서 호출하는 트랜잭션까지 롤백되진 않는다. 

`SkipListener` 에 해당하는 어노테이션은 아래와 같다. 
- `@OnSkipRead`
- `@OnSkipProcess`
- `@OnSkipWrite`

### 리스너 등록
앞서 설명한 것처럼 `Listener` 는 `StepBuilderFactory` 의 `listener` 메소드를 사용해서 `Step` 에 등록하게 된다. 
`listener` 의 메소드의 리턴 타입을 기준으로 `Listener` 를 3가지로 분류 할 수 있다. 
1. `SimpleStepBuilder`
    - `ItemReadListener`
    - `ItemProcessListener`
    - `ItemWriteListener`
1. `FaultTolerantStepBuilder`
    - `SkipListener`
    - `ChunkListener`
1. `AbstractTaskletStepBuilder`
    - `StepExecutionListener`
    - `ChunkListener`
    
만약 사용할 수 있는 모든 `Listener` 를 등록한다면 등록 순서에 따라 특정 `Listener` 는 동작이 불가할 수 있다. 
모든 `Listener` 를 동작한다면 아래와 같은 순서로 등록하는게 모든 동작에 대한 보장이 될 수 있을 것같다. 
`ChunkListener` 는 두 위치 중 구현 내용에 적절한 위치에 등록한다. 
1. `ItemReadListener`, `ItemProcessListener`, `ItemWriteListener`
1. `faultTolerant` 설정 후, `SkipListener`, `ChunkListener`
1. `StepExecutionListener`, `ChunkListener`

### 테스트 

- `InterceptingConfig`

```java
@Configuration
@RequiredArgsConstructor
public class InterceptingConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step sampleStep() {
        return this.stepBuilderFactory
                .get("sampleStep")
                .<String, String>chunk(2)
                .reader(new LogItemReader())
                .processor(new LogItemProcessor())
                .writer(new LogItemWriter())
                .listener(new SampleItemReadListener())
                .listener(new SampleItemProcessorListener())
                .listener(new SampleItemWriteListener())
                .faultTolerant()
                .skipLimit(1)
                .skip(MySkipException.class)
                .listener(new SampleSkipListener())
                .listener(new SampleStepListener())
                .listener(new SampleChunkListener())
                .build();
    }

    @Bean
    public Job sampleJob() {
        return this.jobBuilderFactory
                .get("sampleJob")
                .start(this.sampleStep())
                .build();
    }
}
```  

- `SampleStepListener`

```java
@Slf4j
public class SampleStepListener implements StepExecutionListener {
    @Override
    public void beforeStep(StepExecution stepExecution) {
        log.info("SampleStepListener.beforeStep");
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        log.info("SampleStepListener.afterStep : " + stepExecution.getExitStatus().getExitCode());
        return stepExecution.getExitStatus();
    }
}
```  

- `SampleChunkListener`

```java
@Slf4j
public class SampleChunkListener implements ChunkListener {
    @Override
    public void beforeChunk(ChunkContext context) {
        log.info("SampleChunkListener.beforeChunk");
    }

    @Override
    public void afterChunk(ChunkContext context) {
        log.info("SampleChunkListener.afterChunk");
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        log.info("SampleChunkListener.afterChunkError");
    }
}
```  

- `SampleSkipListener`

```java
@Slf4j
public class SampleSkipListener implements SkipListener<String, String> {
    @Override
    public void onSkipInRead(Throwable t) {
        log.info("SampleSkipListener.onSkipInRead");
    }

    @Override
    public void onSkipInWrite(String item, Throwable t) {
        log.info("SampleSkipListener.onSkipInWrite : " + item);
    }

    @Override
    public void onSkipInProcess(String item, Throwable t) {
        log.info("SampleSkipListener.onSkipInProcess : " + item);
    }
}
```  

- `SampleItemReadListener`

```java
@Slf4j
public class SampleItemReadListener implements ItemReadListener<String> {
    @Override
    public void beforeRead() {
        log.info("SampleItemReadListener.beforeRead");
    }

    @Override
    public void afterRead(String item) {
        log.info("SampleItemReadListener.afterReader : " + item);
    }

    @Override
    public void onReadError(Exception ex) {
        log.info("SampleItemReadListener.onReadError");
    }
}
```  

- `SampleItemProcessorListener`

```java
@Slf4j
public class SampleItemProcessorListener implements ItemProcessListener<String, String> {
    @Override
    public void beforeProcess(String item) {
        log.info("SampleItemProcessorListener.beforeProcess : " + item);
    }

    @Override
    public void afterProcess(String item, String result) {
        log.info("SampleItemProcessorListener.afterProcess : " + item + " -> " + result);
    }

    @Override
    public void onProcessError(String item, Exception e) {
        log.info("SampleItemProcessorListener.onProcessError : " + item);
    }
}
```  

- `SampleItemWriteListener`

```java
@Slf4j
public class SampleItemWriteListener implements ItemWriteListener<String> {
    @Override
    public void beforeWrite(List<? extends String> items) {
        log.info("SampleWriteListener.beforeWrite : " + Arrays.toString(items.toArray()));
    }

    @Override
    public void afterWrite(List<? extends String> items) {
        log.info("SampleWriteListener.afterWrite : " + Arrays.toString(items.toArray()));
    }

    @Override
    public void onWriteError(Exception exception, List<? extends String> items) {
        log.info("SampleWriteListener.onWriteError : " + Arrays.toString(items.toArray()));
    }
}
```  

- `LogItemReader`

```java
@Slf4j
public class LogItemReader implements ItemReader<String> {
    public static String[] DATAS = new String[]{
            "aa", "bb", "cc", "dd"
    };
    public static int INDEX = 0;
    public static String FAIL_STR = "";
    public static boolean IS_SKIP = false;
    
    @Override
    public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if(INDEX < DATAS.length) {
            String data = DATAS[INDEX++];

            if(FAIL_STR.equals(data)) {
                if(IS_SKIP) {
                    throw new MySkipException("skip reader exception");
                } else {
                    throw new CustomException("log reader fail");
                }
            }

            log.info("LogItemReader.read : " + data);
            return data;
        }

        return null;
    }
}
```  

- `LogItemProcessor`

```java
@Slf4j
public class LogItemProcessor implements ItemProcessor<String, String> {
    public static String FAIL_STR = "";
    public static boolean IS_SKIP = false;

    @Override
    public String process(String item) throws Exception {
        if(FAIL_STR.equals(item)) {
            if(IS_SKIP) {
                throw new MySkipException("skip processor exception");
            } else {
                throw new CustomException("log process fail");
            }
        }

        String after = item + "~";

        log.info("LogItemProcessor.process : " + after);

        return after;
    }
}
```  

- `LogItemWriter`

```java
@Slf4j
public class LogItemWriter implements ItemWriter<String> {
    public static String FAIL_STR = "";
    public static boolean IS_SKIP = false;

    @Override
    public void write(List<? extends String> items) throws Exception {
        for (String item : items) {
            if(FAIL_STR.equals(item)) {
                if(IS_SKIP) {
                    throw new MySkipException("skip writer exception");
                } else {
                    throw new CustomException("log writer fail");
                }
            }

            log.info("LogItemWriter.write : " + item);
        }
    }
}
```  

- `InterceptingTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        InterceptingConfig.class
})
public class InterceptingTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        LogItemReader.INDEX = 0;
        LogItemReader.FAIL_STR = "";
        LogItemReader.IS_SKIP = false;
        LogItemProcessor.FAIL_STR = "";
        LogItemProcessor.IS_SKIP = false;
        LogItemWriter.FAIL_STR = "";
        LogItemWriter.IS_SKIP = false;
        this.jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    public void whenLaunchJob_thenAllCompletedLog() throws Exception {
        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "LogItemReader.read : aa",
                "SampleItemReadListener.afterReader : aa",
                "SampleItemReadListener.beforeRead",
                "LogItemReader.read : bb",
                "SampleItemReadListener.afterReader : bb",
                "SampleItemProcessorListener.beforeProcess : aa",
                "LogItemProcessor.process : aa~",
                "SampleItemProcessorListener.afterProcess : aa -> aa~",
                "SampleItemProcessorListener.beforeProcess : bb",
                "LogItemProcessor.process : bb~",
                "SampleItemProcessorListener.afterProcess : bb -> bb~",
                "SampleWriteListener.beforeWrite : [aa~, bb~]",
                "LogItemWriter.write : aa~",
                "LogItemWriter.write : bb~",
                "SampleWriteListener.afterWrite : [aa~, bb~]",
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                // skip next chunk log (cc, dd)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "SampleChunkListener.afterChunk",
                "SampleStepListener.afterStep : COMPLETED"

        ));
    }

    @Test
    public void givenReaderException_whenLaunchJob_thenReaderErrorLog() throws Exception {
        // given
        LogItemReader.FAIL_STR = "dd";

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "LogItemReader.read : cc",
                "SampleItemReadListener.afterReader : cc",
                "SampleItemReadListener.beforeRead",
                // when reader dd throw exception
                "SampleItemReadListener.onReadError",
                "SampleChunkListener.afterChunkError",
                "SampleStepListener.afterStep : FAILED"
        ));
    }

    @Test
    public void givenProcessorException_whenLaunchJob_thenProcessorErrorLog() throws Exception {
        // given
        LogItemProcessor.FAIL_STR = "dd";

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                // skip read cc, dd
                "SampleItemProcessorListener.beforeProcess : cc",
                "LogItemProcessor.process : cc~",
                "SampleItemProcessorListener.afterProcess : cc -> cc~",
                "SampleItemProcessorListener.beforeProcess : dd",
                // when process dd throw exception
                "SampleItemProcessorListener.onProcessError : dd",
                "SampleChunkListener.afterChunkError",
                "SampleStepListener.afterStep : FAILED"
        ));
    }

    @Test
    public void givenWriterException_whenLaunchJob_thenWriteErrorLog() throws Exception {
        // given
        LogItemWriter.FAIL_STR = "dd~";

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                // skip read, process cc, dd
                "SampleWriteListener.beforeWrite : [cc~, dd~]",
                "LogItemWriter.write : cc~",
                // when write dd~ throw exception
                "SampleWriteListener.onWriteError : [cc~, dd~]",
                "SampleChunkListener.afterChunkError",
                "SampleStepListener.afterStep : FAILED"
        ));
    }

    @Test
    public void givenReaderSkipFail_whenLaunchJob_thenReaderSkipLog() throws Exception {
        // given
        LogItemReader.FAIL_STR = "dd";
        LogItemReader.IS_SKIP = true;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "LogItemReader.read : cc",
                "SampleItemReadListener.afterReader : cc",
                "SampleItemReadListener.beforeRead",
                // when reader dd throw skip exception
                "SampleItemReadListener.onReadError",
                "SampleItemReadListener.beforeRead",
                "SampleItemProcessorListener.beforeProcess : cc",
                "LogItemProcessor.process : cc~",
                "SampleItemProcessorListener.afterProcess : cc -> cc~",
                "SampleWriteListener.beforeWrite : [cc~]",
                "LogItemWriter.write : cc~",
                "SampleWriteListener.afterWrite : [cc~]",
                // logging skip reader
                "SampleSkipListener.onSkipInRead",
                "SampleChunkListener.afterChunk",
                "SampleStepListener.afterStep : COMPLETED"
        ));
    }

    @Test
    public void givenProcessorSkipFail_whenLaunchJob_thenProcessorSkipLog() throws Exception {
        // given
        LogItemProcessor.FAIL_STR = "dd";
        LogItemProcessor.IS_SKIP = true;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                // skip read cc, dd
                "SampleItemProcessorListener.beforeProcess : cc",
                "LogItemProcessor.process : cc~",
                "SampleItemProcessorListener.afterProcess : cc -> cc~",
                "SampleItemProcessorListener.beforeProcess : dd",
                // when process dd throw skip exception
                "SampleItemProcessorListener.onProcessError : dd",
                "SampleChunkListener.afterChunkError",
                // restart chunk
                "SampleChunkListener.beforeChunk",
                "SampleItemProcessorListener.beforeProcess : cc",
                "LogItemProcessor.process : cc~",
                "SampleItemProcessorListener.afterProcess : cc -> cc~",
                "SampleWriteListener.beforeWrite : [cc~]",
                "LogItemWriter.write : cc~",
                "SampleWriteListener.afterWrite : [cc~]",
                // logging skip processor
                "SampleSkipListener.onSkipInProcess : dd",
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "SampleChunkListener.afterChunk",
                "SampleStepListener.afterStep : COMPLETED"
        ));
    }

    @Test
    public void givenWriterSkipFail_whenLaunchJob_thenWriterSkipLog() throws Exception {
        // given
        LogItemWriter.FAIL_STR = "dd~";
        LogItemWriter.IS_SKIP = true;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SampleStepListener.beforeStep",
                "SampleChunkListener.beforeChunk",
                // skip log completed first chunk (aa, bb)
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                // skip read, process cc, dd
                "SampleWriteListener.beforeWrite : [cc~, dd~]",
                "LogItemWriter.write : cc~",
                // when write dd~ throw skip exception
                "SampleWriteListener.onWriteError : [cc~, dd~]",
                "SampleChunkListener.afterChunkError",
                // restart chunk
                "SampleChunkListener.beforeChunk",
                "SampleItemProcessorListener.beforeProcess : cc",
                "LogItemProcessor.process : cc~",
                "SampleItemProcessorListener.afterProcess : cc -> cc~",
                "LogItemWriter.write : cc~",
                "SampleWriteListener.afterWrite : [cc~]",
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemProcessorListener.beforeProcess : dd",
                "LogItemProcessor.process : dd~",
                "SampleItemProcessorListener.afterProcess : dd -> dd~",
                "SampleWriteListener.onWriteError : [dd~]",
                "SampleChunkListener.afterChunkError",
                "SampleChunkListener.beforeChunk",
                // logging skip writer
                "SampleSkipListener.onSkipInWrite : dd~",
                "SampleChunkListener.afterChunk",
                "SampleChunkListener.beforeChunk",
                "SampleItemReadListener.beforeRead",
                "SampleChunkListener.afterChunk",
                "SampleStepListener.afterStep : COMPLETED"
        ));
    }
}
```  

---
## Reference
[Chunk-oriented Processing](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/step.html#chunkOrientedProcessing)  