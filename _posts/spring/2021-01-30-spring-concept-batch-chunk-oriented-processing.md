--- 
layout: single
classes: wide
title: ""
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Job
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

테스트에서는 `numberStep` 에 간단한 `





---
## Reference
[Chunk-oriented Processing](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/step.html#chunkOrientedProcessing)  