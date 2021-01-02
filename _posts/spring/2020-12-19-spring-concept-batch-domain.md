--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch 의 구성요소"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Batch 를 구성하는 전체적인 구성요소에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Job
    - Step
    - ExecutionContext
toc: true
use_math: true
---  

## Spring Batch 구성 요소
`Spring Batch` 를 구성하는 요소들을 간단하게 다이어그램 형태로 그려보면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-domain-1.png)  

위 다이어그램에서 보면 알 수 있듯이 하나의 `Job` 은 하나 이상의 `Step` 을 가질 수 있다. 
그리고 하나의 `Step` 은 하나씩의 `ItemReader`, `ItemProcessor`, `ItemWriter` 를 가지게 된다. 
그리고 `Job` 은 `JobLauncher` 에 의해 실행될 수 있고, 
실행된 `Job` 과 `Step` 은 메타데이터 형태로 `JobRepository` 에 정보가 저장되고 관리된다. 

### Job
`Spring Batch` 에서 `Job` 은 하나의 배치 처리를 캡슐화한 하나의 엔티티를 의미한다. 
또한 `Job` 은 `XML` 혹은 `Java Config` 를 사용해서 구성할 수 있다. 
그리고 `Job` 은 아래 다이어그램과 같이 `Job` 을 이루는 계층에서 최상위에 위치하는 엔티티이다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-domain-2.png)  

`Job` 은 하나의 배치 작업을 여러 논리적인 흐름으로 나눠 설계된 `Step` 의 조합으로 구성될 수 있다. 
이러한 포함관계에서 `Job` 을 통해 여러 `Step` 에서 공통적으로 필요한 글로벌 속성을 설정하거나, 
재시작성 등을 설정할 수 있다. 
`Job` 설정에서 아래와 같은 것들을 설정할 수 있다. 
- `Job` 이름
- `Step` 정의 및 실행 순서
- `Job` 의 재실행 여부

`Spring Batch` 에서는 `Job` 인터페이스에 대한 기본 구현체인 `SimpleJob` 을 제공한다. 
이를 사용해서 사용자는 간단하게 `Job` 을 생성할 수 있고, `Java Config` 를 사용한 예시는 아래와 같다. 

```java
@Bean
public Job footballJob() {
    return this.jobBuilderFactory.get("footballJob")
        .start(playerLoad())
        .next(gameLoad())
        .next(playerSummarization())
        .end()
        .build();
}
```  

#### JobInstance
`JobInstance` 는 주기적으로 실행된 논리적인 `Job` 을 의미한다. 
앞서 설명한 `Job` 의 계층도에서 현재 `Job` 은 `EndOfDay` 로, 하루에 한번씩 실행되는 `Job` 을 의미하고 있다. 
여기서 하루에 한번 실행된다는 것을 정의하는 것은 `Job` 이고, 
정의된 `Job` 이 실제로 하루에 한번씩 실행되는 것을 `JobInstance` 라고 한다. 
또한 하루에 한번 실행된 `JobInstance` 는 여러번의 실행을 가질 수 있는데, 
`JobInstance` 가 한번 실행될 때 마다 `JobExecution`(이후 설명) 을 가지게 된다. 
그리고 `Job` 에서 `JobInstance` 를 식별하는 단위는 `JobParameters`(이후 설명)에 의해 결정된다.  

`JobInstance` 는 배치 처리를 위해 로드하는 데이터와는 관계가 있지 않다. 
배치 처리를 위해 데이터를 로드하는 것은 `ItemReader` 의 구현에 따라 달리지게 된다. 
하지만 배치 처리가 같은 `JobInstance` 를 사용하는 경우, 이전 배치 처리 상태를 사용할지에 대한 여부를 결정 할 수 있다. 
여기서 새로운 `JobInstance` 를 사용해서 처음부터 배치 처리를 수행할 수도 있고,  
기존 존재하는 `JobInstance` 를 사용해서 중단된 곳에서 부터 시작할 수도 있다.  

#### JobParameters
앞서 `JobInstance` 는 실행된 `Job` 이 구분되는 논리적인 단위라고 설명했었다. 
`JobParameters` 가 바로 정의된 `Job` 에서 `JobInstance` 가 구분될 수 있도록 하는 기준 값의 역할을 한다. 
`JobParameters` 는 배치 작업이 처리 될때 필요한 파라미터의 역할을 하면서, 
주기적으로 실행되는 `Job` 의 식별자 역할을 하게 된다.  

![그림 1]({{site.baseurl}}/img/spring/concept-batch-domain-3.png)  

위 그림을 보면 하루에 한번 실행되는 `Job` 이 있을때 `JobParameters` 로 `2007/05/05` 를 전달해서 배치 잡을 실행한다. 
그러면 결과로 `2007/05/05` 에 해당하는 `JobInstance` 가 생성되는 것을 확인 할 수 있다. 
만약 다음 배치 작업으로 `2007/05/06` 을 전달하면 `2007/05/06` 에 해당하는 새로운 `JobInstance` 가 생성될 것이다. 
그리고 이후 설명 하겠지만 그림을 보면 생성된 `JobInstance` 에 대해서 첫번째 실행에 대한 `JobExecution` 도 함꼐 생성된 것을 확인 할 수 았다. 

#### JobExecution
`JobExecution` 은 정의된 `Job` 이 실제로 한번 실행되는 것을 의미하는 단위이다. 
`JobExecution` 은 `JobInstance` 와 달리 배치 작업의 성공과 실패를 관리하게 된다. 
하루에 한번 실행되는 `Job` 에서 `2007/05/05` 의 `JobParameters` 를 사용해서 실행하면, 
`2007/05/05` 에 해당하는 `JobInstance` 가 생성되고 첫번째 실행에 대한 `JobExecution` 이 생성된다. 
여기서 첫번째 실행 `JobExecution` 이 실패했다고 가정하고, 
다시 동일한 `JobParameters` 로 배치 작업을 실행하면 동일한 `JobInstance` 을 사용해서 새로운 두번째 `JobExecution` 이 생성되고 
배치 작업이 처리된다.  

`Job` 은 배치 작업이 어떤것이고 어떻게 수행되는지 정의하고, 
`JobInstance` 는 올바른 배치 작업의 재시작 처리나 배처 작업 실행의 그룹화해서 관리하는 객체를 의미한다. 
그리고 `JobExecution` 은 실행 중 실제로 발생한 부분들을 스토리지에 기록하고 관리하는 역할을 수행한다. 
`JobExecution` 이 관리하는 프로퍼티는 아래와 같다. 
- `Status`
- `startTime`
- `endTime`
- `exitStatus` 
- `createTime`
- `lastUpdated`
- `executionContext`
- `failureExceptions`

이후 실제로 배치 작업의 성공여부와 정상동작 여부는 위 프로퍼티들을 바탕으로 확인 할 수 있다.  

만약 하루에 한번 실행되는 `Job` 이 `2007-05-05` 라는 `JobParameters` 로 `21:00` 에 처음 실행되고 `21:30` 에 실행 되었다면, 
`Spring Batch` 에서 관리하는 메타데이터를 요약해 보면 아래와 같다. 
- `BATCH_JOB_INSTANCE`

    JOB_INST_ID|JOB_NAME
    ---|---
    1|EndOfDay
    
- `BATCH_JOB_EXECUTION_PARAMS`

    JOB_EXECUTION_ID|TYPE_CD|KEY_NAME|DATE_VAL|IDENTIFYING
    ---|---|---|---|---
    1|DATE|schedule.Date|2007-05-05 00:00:00|TRUE
    
- `BATCH_JOB_EXECUTION`

    JOB_EXEC_ID|JOB_INST_ID|START_TIME|END_TIME|STATUS
    ---|---|---|---|---
    1|1|2007-05-05 21:00|2007-05-05 21:30:00|FAILED

`2007-05-05 00:00:00` `JobParameters` 로 실행된 `JobExecution` 이 실패한 상태에서 
실패 이슈를 해결하기 위해 하루가 걸리게 되었다고 가정한다. 
이슈 해결이후 동일한 `JobParameters`(`2007-05-05 00:00:00`)로 다음날인 `2007-05-06 21:00` 시간에 배치 작업을 시작했다. 
그 와중에 스케쥴러에 의해 실행된 `2007-05-06 00:00:00` 를 `JobParameters` 로 갖는 배치 작업도 함께 실행 된 상황이다. 
그리고 실행된 모든 배치 작업은 시간이 지나 완료된 상태에서 메타데이터를 요약하면 아래와 같다. 
- `BATCH_JOB_INSTANCE`

    JOB_INST_ID|JOB_NAME
    ---|---
    1|EndOfDay
    2|EndOfDay
    
- `BATCH_JOB_EXECUTION_PARAMS`

    JOB_EXECUTION_ID|TYPE_CD|KEY_NAME|DATE_VAL|IDENTIFYING
    ---|---|---|---|---
    1|DATE|schedule.Date|2007-05-05 00:00:00|TRUE
    2|DATE|schedule.Date|2007-05-05 00:00:00|TRUE
    3|DATE|schedule.Date|2007-05-06 00:00:00|TRUE
    
- `BATCH_JOB_EXECUTION`

    JOB_EXEC_ID|JOB_INST_ID|START_TIME|END_TIME|STATUS
    ---|---|---|---|---
    1|1|2007-05-05 21:00|2007-05-05 21:30:00|FAILED
    2|1|2007-05-06 21:00|2007-05-06 21:30:00|COMPLETED
    3|2|2007-05-06 21:10|2007-05-06 21:40:00|COMPLETED

메타데이터를 보면 동일한 `Job` 에 대해 2개의 `JobExecution` 이 겹치는 시간대에 실행된 것을 확인 할 수 있다. 
하지만 `Spring Batch` 에서는 `JobInstance` 가 다른 `JobExecution` 이 동시에 실행되는 부분에 대해서는 별도의 처리를 하지 않는다. 
대신 `JobInstance` 에 해당하는 `JobExecution` 이 실행 중일때, 
동일한 `JobInstance` 에서 새로운 `JobExecution` 이 실행한다면 `JobExecutionAlreadyRunningException` 예외를 던지게 된다. 
즉 `Spring Batch` 에서는 배치 작업에 대한 스케쥴리에 대한 부분은 별로도 관리해 주지 않고 별도로 스케줄링을 담당하는 곳에 맡기게 된다. 


### Step
`Step` 은 배치 작업에서 독립적이면서 연속적인 단계를 캡슐화한 엔티티이다. 
그리고 `Job` 은 하나 이상의 `Step` 으로 구성되어 전체적인 배치 작업을 수행하게 된다. 
`Step` 에는 독립적인 배치 처리를 정의 및 제어하는데 필요한 모든 정보가 포함 된다. 
`Step` 은 말그대로 배치 처리에 필요한 작업을 구현하는 곳으로 이는 어떠한 배치 처리를 수행하느냐에 따라 `Step` 의 코드가 달라질 수 있다. 
또한 `Job` 이 실제 실행 단위인 `JobExecution` 을 갖는 것처럼, 
`Step` 또한 실제 실행 단위인 `StepExecution` 을 가지게 되고 이는 `JobExecution` 과 연관성이 있다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-domain-4.png)  

#### StepExecution
`StepExecution` 은 `Job` 과 `JobExecution` 관계와 동일하게 실제로 실행된 `Step` 을 의미한다. 
정의된 `Step` 이 실행될 때 마다 `StepExeuction` 이 생성되는 관계이다. 
그리고 `Job` 에서 여러 `Step` 이 순서대로 구성된 상태에서 이전 `Step` 이 실패한 경우 다음 `Step` 은 실행되지 않기 때문에, 
다음 `Step` 에 대한 `StepExecution` 도 생성되지 않는다.  

`Step` 에서 수행하는 동작과 정의는 `StepExecution` 클래스의 객체를 바탕으로 정의할 수 있다. 
실행된 `StepExecution` 에는 대응되는 `Step` 과 `JobExecution` 에 대한 참조와 트랜잭션과 관련된 데이터인 커밋 수, 롤백 수와 시작시간, 종료시간이 포함된다. 
그리고 `ExecutionContext` 를 포함하게 되는데, 
이는 재시작시에 필요한 통계정보, 상태정보나 전체 배치 작업에서 유지해야하는 데이터를 포함 시킬 수 있다. 
`StepExecution` 의 프로퍼티는 아래와 같다. 
- `Status`
- `startTime`
- `endTime`
- `exitStatus`
- `executionContext`
- `readCount`
- `writeCount`
- `commitCount`
- `rollbackCount`
- `readSkipCount`
- `processSkipCount`
- `filterCount`
- `writeSkipCount`


### ExecutionContext
`ExecutionContext` 는 `StepExecution` 객체 또는 `JobExecution` 객체의 범위에서 유지되어야 하는 `key-value` 컬렉션이다. 
이는 `Quartz` 프레임워크에서 `JobDataMap` 과 유사한 역할을 수행한다. 
가정 대표적인 예로 한번 실행된 배치 작업이 실패했을때 다시 재시작을 할때 `ExecutionContext` 에 저장된 데이터를 바탕으로 적절한 정보를 유지시키며 진행할 수 있다. 
아래의 코드와 같이 실패한 배치 작업에서 몇번째 줄까지 수행되었는지 `ExecutionContext` 에 데이터를 저장할 수 있다. 

```java
executionContext.putLong(getKey(LINES_READ_COUNT), reader.getPosition());
```  

앞서 예시로 하루에 한번 수행하는 `Job` 에서 위와 같은 코드가 `StepExecution` 객체이 있는 상태에서 실패 하게됐을 때, 
메타데이터를 요약하면 아래와 같다. 
- `BATCH_JOB_INSTANCE`

    JOB_INST_ID|JOB_NAME
    ---|---
    1|EndOfDay
    
- `BATCH_JOB_EXECUTION_PARAMS`

    JOB_EXECUTION_ID|TYPE_CD|KEY_NAME|DATE_VAL|IDENTIFYING
    ---|---|---|---|---
    1|DATE|schedule.Date|2007-05-05 00:00:00|TRUE
    
- `BATCH_JOB_EXECUTION`

    JOB_EXEC_ID|JOB_INST_ID|START_TIME|END_TIME|STATUS
    ---|---|---|---|---
    1|1|2007-05-05 21:00|2007-05-05 21:30:00|FAILED
    
- `BATCH_STEP_EXECUTION`

    STEP_EXEC_ID|JOB_EXEC_ID|STEP_NAME|START_TIME|END_TIME_STATUS
    ---|---|---|---|---|---
    1|1|loadData|2007-05-05 21:00:00|2007-05-05 21:30:00|FAILED

- `BATCH_STEP_EXECUTION_CONTEXT`

    STEP_EXEC_ID|SHORT_CONTEXT
    ---|---
    1|{piece.count=4123}

`JobParameter` 가 `2007-05-05 00:00:00` 로 실행된 `Job` 이 첫번째 `Step` 을 처리하던 과정에 실패 했고, 
`StepExecution` 에서 실행되었던 정보는 `ExecutionContext` 에 `{piece.count=4123}` 와 같이 저장되었다.  

위와 같이 실패한 배치 작업을 재실행 할때 `StepExecution` 의 `ExecutionContext` 를 사용해서, 
재시작시 이전 배치 작업에서 처리한 다음 라인부터 시작하도록 구성한다면 아래와 같다. 

```java
if (executionContext.containsKey(getKey(LINES_READ_COUNT))) {
    log.debug("Initializing for restart. Restart data is: " + executionContext);

    long lineCount = executionContext.getLong(getKey(LINES_READ_COUNT));

    LineReader reader = getReader();

    Object record = "";
    while (reader.getPosition() < lineCount && record != null) {
        record = readLine();
    }
}
```  

메타데이터부분을 보면 알수 있듯이 `Job` 이 여러 `Step` 으로 구성 돼있다면, 
각 `Step` 의 `StepExecution` 마다 별도의 `ExecutionContext` 를 가지는 것을 확인할 수 있다. 
만약 `JobParameter` 가 `2007-05-05 00:00:00` 로 주어지고 실행된다면, 
`Step` 을 실행할 때 동일한 `JobParameter` 로 실행한 이력이 있지만 실패 했기 때문에 `StepExecution` 은 `BATCH_STEP_EXECUTION_CONTEXT` 에 있는 `ExecutionContext` 정보를 참조하게 된다. 
그리고 `ExecutionContext` 에 있는 `piece.count` 값인 `4123` 라인부터 다시 해당 `Step` 처리를 수행하게 된다.  

`JobParameter` 가 `2007-05-06 00:00:00` 과 같이 새로운 값으로 실행되면다면 `StepExecution` 은 독립된 `ExecutionContext` 를 사용하게 되고, 
`2007-05-05 00:00:00` 로 실행된 `StepExecution` 의 `ExecutionContext` 와는 별도록 구성되고 사용된다.  

이렇게 `Job` 처리에서 지속성이 필요한 데이터를 저장할 수 있는 `ExecutionContext` 는 `StepExecution` 별로 구성되고, 
`JobExecution` 별로 구성하는 것도 가능하다. 

```java
ExecutionContext ecStep = stepExecution.getExecutionContext();
ExecutionContext ecJob = jobExecution.getExecutionContext();
```  

`ecStep` 은 `StepExecution` 단위의 `ExecutionContext` 이고, 
`ecJob` 은 `JobExecution` 단위의 `ExecutionContext` 이기 때문에 두 `ExecutionContext` 는 같지 않다.
`ecStep` 은 하나의 `Step` 스코프에서(`StepExecution`) 커밋 단위와 같이 지속성이 필요한 정보를 저장하는 데이터를 저장하는데 사용 할 수 있다. 
그리고 `ecJob` 은 하나의 `Job` 을 구성하는 여러 `Step` 이 있을때 `Job` 스코프에서 매 `Step` 마다 지속성이 필요한 데이터를 저장하는데 사용 할 수 있다.  


### JobRepository
`JobRepository` 는 앞서 언급한 모든 메타데이터를 저장하는 저장소를 의미한다. 
`JobRepository` 는 `JobLauncher`, `Job`, `Step` 에서 `CRUD` 연산을 제공한다. 
그리고 `Job` 이 처음 실행되면 `JobExecution` 이 실행하기 위한 정보를 `JobRepository` 에서 얻고, 
실행 중에도 관련 정보를 얻거나 저장 할 수 있다. 
`StepExecution`, `JobExecution` 에서 필요한 지속성에 대한 데이터도 `JobRepository` 를 통해 저장소에 저장된다.  

`JobRepository` 는 애플리케이션에서 `@EnableBatchProcessing` 어노테이션을 선언하게 되면 자동으로 설정에 따라 저장소가 구성된다.  


### JobLauncher
`JobLauncher` 는 `Job` 을 특정 `JobParameter` 을 통해 실행하도록 하는 간단한 인터페이스이다. 

```java
public interface JobLauncher {

public JobExecution run(Job job, JobParameters jobParameters)
            throws JobExecutionAlreadyRunningException, JobRestartException,
                   JobInstanceAlreadyCompleteException, JobParametersInvalidException;
}
```  

`JobLauncher` 의 구현체는 `Job` 실행을 위해 `JobRepository` 에서 적절한 `JobExecution` 을 얻을 수 있어야 한다.  


### ItemReader
`ItemReader` 는 `Step` 단위에서 처리를 위해 필요한 `Item` 을 읽어오는 추상 클래스이다. 
`ItemReader` 는 구현한 `Item` 단위로 읽어오게 되고, 만약 더 이상 읽을 `Item` 이 없다면 `null` 을 리턴하게 된다. 


### ItemWriter
`ItemWriter` 는 `Step` 단위의 결과물을 만드는 추상 클래스이다. 
`Itemwriter` 의 실행은 한번에 처리하는 `Item` 의 개수를 설정하는 `chunk` 개수에 따라 수행 횟수가 달라질 수 있다. 

### ItemProcessor
`ItemProcessor` 는 `Batch Job` 처리의 비지니스 로직을 구현할 수 있는 추상 클래스이다. 
`ItemReader` 에서 읽은 `Item` 단위 데이터는 `ItemProcessor` 에 전달되고 구현된 로직에 맞춰 처리된다. 
그리고 `ItemWriter` 에 전달하기 위해 설정된 `chunk` 의 수만큼 `Item` 을 처리하고 `ItemWriter` 로 `chunk` 단위 데이터를 전달한다. 
`Item` 의 유효성은 `null` 인지 아닌지 검사해서 처리하고 해당 데이터 부터는 `ItemWriter` 로 전달되지 않는다. 


---
## Reference
[The Domain Language of Batch](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/domain.html#domainLanguageOfBatch)  