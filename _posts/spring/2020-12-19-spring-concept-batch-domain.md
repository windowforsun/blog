--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Batch 의 구성요소"
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
toc: true
use_math: true
---  

## Spring Batch 구성 요소
`Spring Batch` 를 구성하는 요소들을 간단하게 다이어그램 형태로 그려보면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-doamin-1.png)  

위 다이어그램에서 보면 알 수 있듯이 하나의 `Job` 은 하나 이상의 `Step` 을 가질 수 있다. 
그리고 하나의 `Step` 은 하나씩의 `ItemReader`, `ItemProcessor`, `ItemWriter` 를 가지게 된다. 
그리고 `Job` 은 `JobLauncher` 에 의해 실행될 수 있고, 
실행된 `Job` 과 `Step` 은 메타데이터 형태로 `JobRepository` 에 정보가 저장되고 관리된다. 

### Job
`Spring Batch` 에서 `Job` 은 하나의 배치 처리를 캡슐화한 하나의 엔티티를 의미한다. 
또한 `Job` 은 `XML` 혹은 `Java Config` 를 사용해서 구성할 수 있다. 
그리고 `Job` 은 아래 다이어그램과 같이 `Job` 을 이루는 계층에서 최상위에 위치하는 엔티티이다. 

![그림 1]({{site.baseurl}}/img/spring/concept-batch-doamin-2.png)  

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

![그림 1]({{site.baseurl}}/img/spring/concept-batch-doamin-3.png)  

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

![그림 1]({{site.baseurl}}/img/spring/concept-batch-doamin-4.png)  

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

스텝의 실행은 Step Execution 클래스의 객체에 의해 나타납니다. 각 실행에는 대응하는 스텝, 작업 실행 및 커밋 수나 롤백 수, 시작 시각 및 종료 시각과 같은 트랜잭션 관련 데이터에 대한 참조가 포함됩니다. 또한 각 단계의 실행에는 Execution Context가 포함됩니다.Execution Context에는 재시동에 필요한 통계정보나 상태정보 등 개발자가 배치실행 전체에서 유지해야 할 모든 데이터가 포함됩니다. 다음 표에 스텝 수행 속성들을 보여 줍니다.









`Spring Batch` 는 크게 `Job` 과 `Step` 을 바탕으로 실행되면서, 
개발자가 제공하는 `ItemReader`, `ItemWriter` 등을 통해 데이터를 처리하게 된다. 
이는 아래와 같은 `Spring` 패턴, 템플릿, 콜백 등을 


---
## Reference
[The Domain Language of Batch](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/domain.html#domainLanguageOfBatch)  