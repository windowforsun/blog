--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch Tasklet"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Batch Chunk 방식과 다르게 활용할 수 있는 Tasklet에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Tasklet
toc: true
use_math: true
---  

## Tasklet
[여기]({{site.baseurl}}{% link _posts/spring/2021-01-30-spring-concept-batch-chunk-oriented-processing.md %})
에서는 `Step` 을 구성하고 다루는 `Chunk` 방식에 대해 알아봤다. 
하지만 `Step` 을 구성하는 방법은 `Chunk` 방식외에 `Tasklet` 기반으로 사용하는 방법도 있다. 
`Chunk` 를 사용하는 방법은 일반적으로 데이터를 받고(`ItemReader`), 그 데이터를 처리하고(`ItemProcessor`), 
마지막으로 데이터를 별도의 저장소에 저장하는(`ItemWriter`) 흐름으로 동작한다. 
이와 비교해 `Tasklet` 은 `execute()` 메소드 하나만 존재하는 인터페이스로, 잡 처리를 위한 여러 스텝 중 `Chunk` 의 
흐름에서 벗어난 작업을 할때 사용할 수 있다.  

그리고 각 `Tasklet` 마다 트랜잭션을 갖는다.  

`Tasklet` 인터페이스의 원형은 아래와 같다.  

```java
public interface Tasklet {

	@Nullable
	RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;

}
```  

앞서 언급한 것과 같이 `execute()` 메소드만 존재하고 `RepeatStatus` 타입을 리턴하는 형태를 가지고 있다.  

간단하게 구성한 기본적인 `Tasklet` 의 예시는 아래와 같다.  

- `SimpleTasklet`

```java
@Slf4j
public class SimpleTasklet implements Tasklet {
    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        // do something ..

        log.info("SimpleTasklet.execute");

        return RepeatStatus.FINISHED;
    }
}
```  

  - `Tasklet` 은 `RepeatStatus.FINISHED` 를 리턴하며 종료할 수 있다. 

- `SimpleTaskletConfig`

```java
@Configuration
@RequiredArgsConstructor
public class SimpleTaskletConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Tasklet simpleTasklet() {
        return new SimpleTasklet();
    }

    @Bean
    public Step simpleStep() {
        return this.stepBuilderFactory
                .get("simpleStep")
                .tasklet(this.simpleTasklet())
                .build();
    }

    @Bean
    public Job simpleJob() {
        return this.jobBuilderFactory
                .get("simpleJob")
                .start(this.simpleStep())
                .build();
    }
}
```  

- `SimpleTaskletTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchConfig.class,
        SimpleTaskletConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class SimpleTaskletTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void givenSimpleTasklet_whenLaunchJob_thenCompleted() throws Exception {
        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }

    @Test
    public void givenSimpleTasklet_whenLaunchJob_thenExecuteLog() throws Exception {
        // when
        this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(this.capture.getOut(), containsString("SimpleTasklet.execute"));
    }
}
```  

### TaskletAdapter
`TaskletAdapter` 는 `ItemReader`, `ItemWriter` 어댑터처럼 기존 클래스의 메소드를 `Tasklet` 동작으로 사용할 수 있도록 제공하는 클래스이다. 
만약 `Tasklet` 동작을 수행하는 `Dao` 혹은 `Service` 클래스가 이미 존재하는 경우, 
별도 `Tasklet` 클래스를 구현하지 않고 `TaskletAdapter` 를 사용하면 동작을 그대로 수행할 수 있다.  

`Tasklet` 에 필요한 동작이 데이터를 지우는 동작이라고 가정했을 때, 
`Dao` 에 이미 해당 동작이 구현돼 있어 `TaskletAdapter` 를 사용해서 `Dao` 의 메소드를 `Tasklet` 동작으로 연결시키는 예제는 아래와 같다.  

- `SomeDao`

```java
@Slf4j
public class SomeDao {
    public void deleteBySome(String some) {
        log.info("SomeDao.deleteBySome({})", some);
    }
}
```  

  - `deleteBySome` 은 `Tasklet` 동작을 수행하는 메소드이다. 
  - `some` 에 해당하는 데이터를 지우는 동작을 수행한다고 가정하고, 동작 판별을 위해 로그를 찍는다. 
  
- `TaskletAdapterConfig`

```java
@Configuration
@RequiredArgsConstructor
public class TaskletAdapterConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public SomeDao someDao() {
        return new SomeDao();
    }

    @Bean
    public MethodInvokingTaskletAdapter taskletAdapter() {
        MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();
        adapter.setTargetObject(this.someDao());
        adapter.setTargetMethod("deleteBySome");
        adapter.setArguments(new String[]{"test"});

        return adapter;
    }

    @Bean
    public Step taskletAdapterStep() {
        return this.stepBuilderFactory
                .get("taskletAdapterStep")
                .tasklet(this.taskletAdapter())
                .build();
    }

    @Bean
    public Job taskletAdapterJob() {
        return this.jobBuilderFactory
                .get("taskletAdapterJob")
                .start(this.taskletAdapterStep())
                .build();
    }
}
```  

- `TaskletAdapterTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchConfig.class,
        TaskletAdapterConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class TaskletAdapterTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void givenTaskletAdapter_whenLaunchJob_thenCompleted() throws Exception {
        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }

    @Test
    public void givenTaskletAdapter_whenLaunchJob_thenDeleteExecuteLog() throws Exception {
        // when
        this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(this.capture.getOut(), containsString("SomeDao.deleteBySome(test)"));
    }
}
```  

### Tasklet 리턴값에 따른 동작
앞서 `Tasklet` 의 `execute` 메소드는 `RepeatStatus` 타입을 리턴하고, 
`RepeatStatus.FINISHED` 리턴을 통해 종료한다고 언급했었다. 
`RepeatStatus` 의 종류는 `FiNISHED` 외에 한가지 더 있는데 원형은 아래와 같다. 

```java
public enum RepeatStatus {

	CONTINUABLE(true), 
	FINISHED(false);

	private final boolean continuable;

	private RepeatStatus(boolean continuable) {
		this.continuable = continuable;
	}

	public static RepeatStatus continueIf(boolean continuable) {
		return continuable ? CONTINUABLE : FINISHED;
	}

	public boolean isContinuable() {
		return this == CONTINUABLE;
	}

	public RepeatStatus and(boolean value) {
		return value && continuable ? CONTINUABLE : FINISHED;
	}

}
```  

이름에서 유추할 수 있듯이 `CONTINUABLE` 을 리턴하게 되면 `Tasklet` 은 동작을 종료하지 않고, 
계속해서 `execute()` 메소드가 `FINISHED` 혹은 `null` 을 리턴할 때까지 반복 수행하게 된다. 
그리고 예외를 발생시키게 되면 `ExitStatus.FAILED` 로 `Tasklet` 이 종료된다.  

아래는 `execute()` 메소드 리턴값에 따른 동작에 대한 예시이다. 

- `RepeatableTasklet`

```java
@Slf4j
public class RepeatableTasklet implements Tasklet {
    private final int MAX = 10;
    private int i;

    @BeforeStep
    public void beforeStep() {
        this.i = 0;
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        if(this.i < MAX) {
            log.info("RepeatableTasklet i={}", this.i);
            i++;
            return RepeatStatus.CONTINUABLE;
        } else {
            return RepeatStatus.FINISHED;
        }
    }
}
```  

  - `beforeStep()` 메소드에서 스텝시작 전 `i` 값을 0으로 초기화 한다. 
  - `execute()` 에서는 `i` 가 `MAX` 값보다 작은 경우 `i` 값에 대한 로그를 찍고 `i` 값 증가후, `COMTINUABLE` 을 리턴한다. 
  - `i` 가 `MAX` 보다 같거나 큰 경우, `FINISHED` 를 리턴한다. 

- `RepeatableTaskletConfig`

```java
@Configuration
@RequiredArgsConstructor
public class RepeatableTaskletConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Tasklet repeatableTasklet() {
        return new RepeatableTasklet();
    }

    @Bean
    public Step repeatableTaskletStep() {
        return this.stepBuilderFactory
                .get("repeatableTaskletStep")
                .tasklet(this.repeatableTasklet())
                .build();
    }

    @Bean
    public Job repeatableTaskletJob() {
        return this.jobBuilderFactory
                .get("repeatableTaskletJob")
                .start(this.repeatableTaskletStep())
                .build();
    }
}
```  

- `RepeatableTaskletTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchConfig.class,
        RepeatableTaskletConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class RepeatableTaskletTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Test
    public void givenRepeatableTasklet_whenLaunchJob_thenCompleted() throws Exception {
        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
    }

    @Test
    public void givenRepeatableTasklet_whenLaunchJob_thenIncrementNumberLob() throws Exception {
        // when
        this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "RepeatableTasklet i=0",
                "RepeatableTasklet i=1",
                "RepeatableTasklet i=2",
                "RepeatableTasklet i=3",
                "RepeatableTasklet i=4",
                "RepeatableTasklet i=5",
                "RepeatableTasklet i=6",
                "RepeatableTasklet i=7",
                "RepeatableTasklet i=8",
                "RepeatableTasklet i=9"
        ));
    }
}
```  

### Intercepting Tasklet
`Chunk` 방식에서는 다양한 `Listener` 를 통해 전체 흐름과정에서 원하는 처리를 추가할 수 있었다. 
`Tasklet` 은 흐름이 단순해진 만큼 사용할 수 있는 `Listener` 도 비교적 한정적이다. 
- `ChunkListener`
- `StepExecutionListener`

다음은 `Tasklet` 성공, 실패에 따른 `Listener` 호출 순서를 확인할 수 있는 예제이다.  

- `SomeChunkListener`

```java
@Slf4j
public class SomeChunkListener implements ChunkListener {
    @Override
    public void beforeChunk(ChunkContext context) {
        log.info("SomeChunkListener.beforeChunk");
    }

    @Override
    public void afterChunk(ChunkContext context) {
        log.info("SomeChunkListener.afterChunk");
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        log.info("SomeChunkListener.afterChunkError");
    }
}
```  

- `SomeStepListener`

```java
@Slf4j
public class SomeStepListener implements StepExecutionListener {
    @Override
    public void beforeStep(StepExecution stepExecution) {
        log.info("SomeStepListener.beforeStep");
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        log.info("SomeStepListener.afterStep");
        return stepExecution.getExitStatus();
    }
}
```  

- `InterceptingTasklet`

```java
@Slf4j
public class InterceptingTasklet implements Tasklet {
    public static boolean IS_FAIL = false;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        if(IS_FAIL) {
            log.info("InterceptingTasklet.execute FAIL");
            throw new Exception("Tasklet Fail");
        } else {
            log.info("InterceptingTasklet.execute SUCCESS");
            return RepeatStatus.FINISHED;
        }
    }
}
```  

- `InterceptingTaskletConfig`

```java
@Configuration
@RequiredArgsConstructor
public class InterceptingTaskletConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Tasklet interceptingTasklet() {
        return new InterceptingTasklet();
    }

    @Bean
    public Step interceptingTaskletStep() {
        return this.stepBuilderFactory
                .get("interceptingTaskletStep")
                .tasklet(this.interceptingTasklet())
                .listener(new SomeChunkListener())
                .listener(new SomeStepListener())
                .build();
    }

    @Bean
    public Job interceptingTaskletJob() {
        return this.jobBuilderFactory
                .get("interceptingTaskletJob")
                .start(this.interceptingTaskletStep())
                .build();
    }
}
```  

- `InterceptingTaskletTest`

```java
@RunWith(SpringRunner.class)
@SpringBatchTest
@ContextConfiguration(classes = {
        BatchConfig.class,
        InterceptingTaskletConfig.class
})
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class InterceptingTaskletTest {
    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;
    @Rule
    public final OutputCaptureRule capture = new OutputCaptureRule();

    @Before
    public void setUp() {
        InterceptingTasklet.IS_FAIL = false;
    }

    @Test
    public void givenSuccessInterceptingTasklet_whenLaunchJob_thenSuccessLog() throws Exception {
        // given
        InterceptingTasklet.IS_FAIL = false;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.COMPLETED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SomeStepListener.beforeStep",
                "SomeChunkListener.beforeChunk",
                "InterceptingTasklet.execute SUCCESS",
                "SomeChunkListener.afterChunk",
                "SomeStepListener.afterStep"
        ));
    }

    @Test
    public void giveFailInterceptingTasklet_whenLaunchJob_thenFailLog() throws Exception {
        // given
        InterceptingTasklet.IS_FAIL = true;

        // when
        JobExecution jobExecution = this.jobLauncherTestUtils.launchJob();

        // then
        assertThat(jobExecution.getExitStatus().getExitCode(), is(ExitStatus.FAILED.getExitCode()));
        assertThat(this.capture.getOut(), stringContainsInOrder(
                "SomeStepListener.beforeStep",
                "SomeChunkListener.beforeChunk",
                "InterceptingTasklet.execute FAIL",
                "SomeChunkListener.afterChunkError",
                "SomeStepListener.afterStep"
        ));
    }
}
```  

---
## Reference
[Tasklet](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/step.html#taskletStep)  