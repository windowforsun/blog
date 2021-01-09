--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch Unit Test"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Batch 에서 Unit Test 를 사용해서 Job을 테스트하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - UnitTest
    - Test
toc: true
use_math: true
---  

## Spring Batch Unit Test
`Spring Batch` 에 대한 `Unit Test` 또한 기존 `Spring` 프로젝트 테스트 방법과 비슷하게 진행할 수 있다. 
`Spring Batch` 에서 제공해주는 다양한 클래스를 사용해서 단위 테스트를 수행하는 방법에 대해 알아본다. 

### Unit Test Class 설정
단위 테스트에서 배치 작업을 실행하기 위해서는 `Spring Framework` 에서 배치 작업이 설정된 `Application Context` 를 등록해야 한다. 
이 등록작업을 위해서 아래와 같은 2개의 `Annotation` 을 사용한다. 
- `@RunWith(SpringRunner.class)` : 해당 단위 테스트가 `Spring` 을 사용한 테스트임을 나타낸다. 
- `@ContextConfigureation(classes = {...}` : `Application Context` 에 등록할 설정 파일들을 등록한다. 
    - 테스트 환경에따라 `@SpringBootTest(classes = {...})` 와 같이 사용해서 `Spring` 에 대한 통합테스트 설정까지 함께 할 수 있다. 
- `@SpringBatchTest` : 실질적으로 `Spring Batch` 테스트에 필요한 여러 클래스를 제공하는 `Annotation` 이다. 
대표 적으로 `Job` 을 실행하는 `JobLauncherTestUtils` 와 메타 데이터 정보를 조작할 수 있는 `JobRepositoryTestUtils` 가 있다. 
- `@EnableBatchProcessing` : `Spring Batch` 를 사용할 수 있도록 활성화 하는 `Annotation` 이다. 
- `@EnableAutoConfiguration` : `Spring Boot` 혹은 `@Configuration` 에서 정의한 빈들을 자동 설정 및 생성하도록하는 `Annotation` 이다. 

`@SpringBatchTest` `Annotation` 을 통해 `JobRepository` 에 대한 테스트 클래스를 사용할수 있다고 언급했었다. 
별도의 `DataSource` 설정을 주입해주지 않는다면 `auto-configuration` 에서는 `in-memory` 용 `JobRepository` 를 생성한다. 
이렇게 만들어진 `JobRepositoryTestUtils` 는 테스트 클래스 하나에서 여러번 반복적으로 테스트를 수행할 때, 
`Context` 정보와 같은 메타데이터 정보를 초기화 시켜주는 용도로 사용될 수 있다.  

`Spring Batch` 테스트 코드를 구성할때 주의해야 할 점은 단위 테스크 클래스 하나당 하나의 `Job` 만 설정돼야 한다는 점이다.  

이후 진행될 예제에서는 공통적인 단위 테스트 설정이 있는 클래스를 정의해 두고, 단위 테스트 코드가 작성되는 클래스에서 
공통 설정 클래스를 등록하는 방법으로 구성해서 진행한다. 

```java
@EnableBatchProcessing
@EnableAutoConfiguration
public class BatchTestConfig {
}
```  

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        SampleBatchConfig.class,
        BatchTestConfig.class
})
public class SampleJobTest {

    .. 단위 테스트 코드 .. 

}
```  

### 예제 프로젝트 

```
└─src
    ├─main
    │  ├─java
    │  │  └─com
    │  │      └─windowforsun
    │  │          └─batchunittest
    │  │              │  BatchUnitTestApplication.java
    │  │              │
    │  │              ├─config
    │  │              │      ItemBatchConfig.java
    │  │              │      TaskletBatchConfig.java
    │  │              │
    │  │              ├─item
    │  │              │      LoggingItemWriter.java
    │  │              │      SimpleItemReader.java
    │  │              │      UpperCaseItemProcessor.java
    │  │              │
    │  │              └─tasklet
    │  │                      SampleTasklet.java
    │  │                      SampleTasklet2.java
    │  │
    │  └─resources
    │          application.yaml
    │
    └─test
        └─java
          └─com
              └─windowforsun
                  └─batchunittest
                      └─endtoend
                      │      ItemBatchEndToEndTest.java
                      │      TaskletBatchEndToEndTest.java
                      │
                      ├─stepscope
                      │      ItemBatchStepScopeTest.java
                      │
                      └─steptest
                              ItemBatchStepTest.java
                              TaskletBatchStepTest.java

```  

- `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.2.RELEASE'
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.9'

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

#### Tasklet
`Tasklet` 으로 구성된 `Job` 과 2개의 `Step` 은 간단한 로그 한줄씩만 출력하고 종료되는 배치 잡이다. 

- `SampleTasklet`

```java
@Slf4j
public class SampleTasklet implements Tasklet {
    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        log.info("Sample is run !");

        return RepeatStatus.FINISHED;
    }
}
```  

- `SampleTasklet2`

```java
@Slf4j
public class SampleTasklet2 implements Tasklet {
    public static boolean IS_FAIL = false;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        if(IS_FAIL) {
            log.info("Sample2 is fail !");
            IS_FAIL = false;

            throw new Exception("fail!!");
        } else {
            log.info("Sample2 is run !");
        }

        return RepeatStatus.FINISHED;
    }
}
```  

- `TaskletBatchConfig`

```java
@Configuration
@AllArgsConstructor
public class TaskletBatchConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Tasklet sampleTasklet1() {
        return new SampleTasklet();
    }

    @Bean
    public Step taskletStep() {
        return this.stepBuilderFactory
                .get("taskletStep")
                .tasklet(this.sampleTasklet1())
                .build();
    }

    @Bean
    public Tasklet sampleTasklet2() {
        return new SampleTasklet2();
    }

    @Bean
    public Step taskletStep2() {
        return this.stepBuilderFactory
                .get("taskletStep2")
                .tasklet(this.sampleTasklet2())
                .build();
    }

    @Bean
    public Job taskletJob() {
        return this.jobBuilderFactory
                .get("taskletJob")
                .start(this.taskletStep())
                .next(this.taskletStep2())
                .build();
    }
}
```  

#### Item
`Item` 으로 구성된 `Job` 과 `Step` 은 `ItemReader` 로 부터 받은 문자열을 
`ItemProcessor` 에서 대문자로 변경하고 `ItemWriter` 로 전달해서 콘솔로 출력하는 배치 잡이다. 

- `SimpleItemReader`

```java
public class SimpleItemReader implements ItemReader<String> {
    private String[] strings = new String[] {
            "a", "b", "c"
    };

    public static int INDEX = 0;

    @Override
    public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if(INDEX < this.strings.length) {
            String str = this.strings[INDEX++];
            log.info("simple reader : " + str);

            return str;
        }

        return null;
    }
}
```  

- `UpperCaseItemProcessor`

```java
@Slf4j
public class UpperCaseItemProcessor implements ItemProcessor<String, String> {
    @Override
    public String process(String item) throws Exception {
        log.info("uppercase processor : " + item);

        return item.toUpperCase();
    }
}
```  

- `LoggingItemWriter`

```java
@Slf4j
public class LoggingItemWriter implements ItemWriter<String> {
    @Override
    public void write(List<? extends String> items) throws Exception {
        for(String item : items) {
            log.info("LoggingItemWriter : " + item);
        }
    }
}
```  

- `ItemBatchConfig`

```java
@Configuration
@AllArgsConstructor
public class ItemBatchConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public ItemReader<String> reader() {
        return new SimpleItemReader();
    }

    @Bean
    public ItemProcessor<String, String> upperCaseProcessor() {
        return new UpperCaseItemProcessor();
    }

    @Bean
    public ItemWriter<String> writer() {
        return new LoggingItemWriter();
    }

    @Bean
    public Step itemStep() {
        return this.stepBuilderFactory
                .get("itemStep")
                .<String, String>chunk(2)
                .reader(this.reader())
                .processor(this.upperCaseProcessor())
                .writer(this.writer())
                .build();
    }

    @Bean
    public Job itemJob() {
        return this.jobBuilderFactory
                .get("itemJob")
                .start(this.itemStep())
                .build();
    }
}
```  

### End to End Test
`End to End` 테스트란 구성한 `Job` 을 실행해서 `Job` 에 구성된 `Step` 및 이하 구성 요소들을 모두 테스트하는 것을 의미한다. 
`ItemReader` 및 `ItemWriter` 가 데이터베이스에서 데이터를 읽어오고 결과를 작성한다면 데이터베이스에 대한 의존성도 필요할 수 있다. 
`Job` 이 성공적으로 실행됐음은 `JobExecution` 에서 `JobInstance` 가져와서 `ExitStatus` 가 `COMPLETED` 인지 검증함으로써 가능하다.  

#### Tasklet
`SampleTasklet2` 클래스 전역 변수로 `IS_FAIL` 에 따라 임의로 `Step` 의 성공 실패 여부를 정의할 수 있다. 
`IS_FAIL` 초기 값인 `false` 로 테스를 수행하면 `SampleTasklet`, `SampleTasklet2` 모두 실행되고 `Job` 까지 성공함을 확인 할 수 있다. 
`Is_FAIL` 을 `true` 로 설정하고 테스트를 수행하면 `SampleTasklet` 은 성공하지만 `SampleTasklet2` 에서 실패되어, 
`Job` 이 실패하게 된다.   

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        TaskletBatchConfig.class
})
public class TaskletBatchEndToEndTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void whenLaunchTaskletJob_thenSuccess() throws Exception {
        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchJob();
        JobInstance actualJobInstance = actualJobExecution.getJobInstance();

        // then
        assertThat(actualJobInstance.getJobName(), is("taskletJob"));
        assertThat(actualJobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Sample is run"),
                containsString("Sample2 is run")
        ));
    }

    @Test
    public void givenSampleTasklet2Fail_whenLaunchTaskletJob_ThenFail() throws Exception {
        // given
        SampleTasklet2.IS_FAIL = true;

        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchJob();
        JobInstance actualJobInstance = actualJobExecution.getJobInstance();

        // then
        assertThat(actualJobInstance.getJobName(), is("taskletJob"));
        assertThat(actualJobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Sample is run"),
                containsString("Sample2 is fail")
        ));
    }
}
```  

#### Item
`Item` 으로 구성된 `Step` 은 `ItemReader`, `ItemProcessor`, `ItemWriter` 로 구성돼 있고 
테스트를 실행하면 모든 구성요소들이 유기적으로 실행돼 `Job` 이 성공됨을 확인할 수 있다.  

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        ItemBatchConfig.class
})
public class ItemBatchEndToEndTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void whenLaunchItemJob_thenSuccess() throws Exception {
        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchJob();
        JobInstance actualJobInstance = actualJobExecution.getJobInstance();

        // then
        assertThat(actualJobInstance.getJobName(), is("itemJob"));
        assertThat(actualJobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                containsString("simple reader : a"),
                containsString("simple reader : b"),
                containsString("simple reader : c"),
                containsString("uppercase processor : a"),
                containsString("uppercase processor : b"),
                containsString("uppercase processor : c"),
                containsString("LoggingItemWriter : A"),
                containsString("LoggingItemWriter : B"),
                containsString("LoggingItemWriter : C")
        ));
    }
}
```  

### Step Test
하나의 `Job` 이 여러 `Step` 으로 구성된다면, 
`Job` 단위로 테스트를 수행하기에는 큰 부담이 있을 수 있다. 
`JobLaunchTestUtils` 에서는 이러한 이유로 `Job` 에 구성된 `Step` 을 개별적으로 테스트 할 수 있도록 `launchStep()` 메소드를 제공한다.  

#### Tasklet

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        TaskletBatchConfig.class
})
public class TaskletBatchStepTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void givenTaskletStep_whenLaunchTaskletStep_thenOnlyLaunchTaskletStep() throws Exception {
        // given
        String stepName = "taskletStep";

        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchStep(stepName);
        Collection<StepExecution> actualStepExecutions = actualJobExecution.getStepExecutions();

        // then
        assertThat(actualStepExecutions, hasSize(1));
        assertThat(actualJobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Sample is run"),
                not(containsString("Sample2 is run"))
        ));
    }

    @Test
    public void givenTaskletStep2_whenLaunchTaskletStep2_thenOnlyLaunchTaskletStep2() throws Exception {
        // given
        String stepName = "taskletStep2";

        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchStep(stepName);
        Collection<StepExecution> actualStepExecutions = actualJobExecution.getStepExecutions();

        // then
        assertThat(actualStepExecutions, hasSize(1));
        assertThat(actualJobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), allOf(
                containsString("Sample2 is run"),
                not(containsString("Sample is run"))
        ));
    }
}
```  

#### Item
`Step` 단위로 테스트를 진행함으로써 `Step` 동작에 대해 
읽은 개수, 처리 개수, 쓴 개수 등과 같이 더욱 디테일한 정보에 대해 테스트를 수행할 수 있다. 

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        ItemBatchConfig.class
})
public class ItemBatchStepTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void givenItemStep_whenLaunch_thenSuccessItemCount() throws Exception {
        // given
        String stepName = "itemStep";

        // when
        JobExecution actualJobExecution = this.jobLauncherTestUtils.launchStep(stepName);
        Collection<StepExecution> actualStepExecutions = actualJobExecution.getStepExecutions();

        // when
        actualStepExecutions.forEach(stepExecution -> {
            assertThat(stepExecution.getReadCount(), is(3));
            assertThat(stepExecution.getProcessSkipCount(), is(0));
            assertThat(stepExecution.getWriteCount(), is(3));
        });
    }
}
```  

### Step Scope Test
하나의 `Step` 은 `ItemReader`, `ItemProcessor`, `ItemWriter` 로 이루어 진다. 
이러한 포함관계에서 `Step` 을 구성하는 요소들의 역할일 커지게 되면 `Step` 별로 테스트 또한 난해할 수 있다. 
`Spring Batch` 는 모의 `Step` 을 만들어서 `Step` 을 구성하는 요소별로 테스트를 수행해 볼 수 있다. 
모의 `Step` 은 `MetaDataInstanceFactory` 클래스에서 `createStepExecution() 메소드를 사용해서 생성가능하다.  

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchTestConfig.class,
        ItemBatchConfig.class
})
public class ItemBatchStepScopeTest {
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();
    @Autowired
    private ItemReader<String> reader;
    @Autowired
    private ItemProcessor<String, String> processor;
    @Autowired
    private ItemWriter<String> writer;

    @Test//sequence
    public void givenMockStep_whenCallReader_thenReturnSequentialString() throws Exception {
        // given
        StepExecution stepExecution = MetaDataInstanceFactory.createStepExecution();

        // when
        StepScopeTestUtils.doInStepScope(stepExecution, () -> {

            // then
            assertThat(this.reader.read(), is("a"));
            assertThat(this.reader.read(), is("b"));
            assertThat(this.reader.read(), is("c"));

            return null;
        });
    }

    @Test
    public void givenMockStep_whenCallProcessor_thenReturnUpperCaseString() throws Exception {
        // given
        StepExecution stepExecution = MetaDataInstanceFactory.createStepExecution();
        String inputStr = "str";

        // when
        StepScopeTestUtils.doInStepScope(stepExecution, () -> {

            // then
            String actual = this.processor.process(inputStr);
            assertThat(actual, is(inputStr.toUpperCase()));

            return null;
        });
    }

    @Test
    public void givenMockStep_whenCallWriter_thenLoggingInputString() throws Exception {
        // given
        StepExecution stepExecution = MetaDataInstanceFactory.createStepExecution();

        // when
        StepScopeTestUtils.doInStepScope(stepExecution, () -> {
            this.writer.write(List.of("11", "22"));
            this.writer.write(List.of("33"));

            // then
            assertThat(this.capture.getOut(), allOf(
                    containsString("LoggingItemWriter : 11"),
                    containsString("LoggingItemWriter : 22"),
                    containsString("LoggingItemWriter : 33")
            ));

            return null;
        });
    }
}
```  

---
## Reference
[Unit Testing](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/testing.html#testing)  
[Testing a Spring Batch Job](https://www.baeldung.com/spring-batch-testing-job)  