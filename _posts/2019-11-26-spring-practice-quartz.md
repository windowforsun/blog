--- 
layout: single
classes: wide
title: "[Spring 실습] Quartz Job Scheduling"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Quartz Job Scheduler 에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Job
    - Scheduler
    - Quartz
---  

## Quartz
- [Quartz](http://www.quartz-scheduler.org/)는 오픈소스 스케쥴러 라이브러리이다.
- 완전한 자바로 개발되었고, 쉽게 연동해서 스케쥴링을 사용할 수 있다.
- 기존 Spring 환경에서도 연동이 잘되었지만, Spring Boot 2.0 부터 `Starter` 를 제공하여 더욱 간단하게 사용 가능하다.
- 수십에서 수천개의 작업도 스케쥴링 가능하다.
- `Interval` 형식 또는 `Cron` 표현식을 통해 스케쥴링의 주기를 설정할 수 있다.

## 장점
- 별도의 설정없이 바로 사용하능한 In-Memory Job Scheduler 를 제공한다.
- DB 기반으로 Job Scheduler 뿐만 아니라, Scheduler 간의 Clustering 기능을 제공한다. (Fail-over, Random 방식 분산처리)
- 다양한 기본 Plug-in 을 제공한다. 
	- ShutdownHookPlugin : JVM 종료 이벤트로 스케쥴러에게 종료 알림
	- LoggingJobHistoryPlugin : Job 실행에 대한 로깅
	
## 단점
- Random 방식의 Clustering 으로 완벽한 분산이 되지 않는다.
- Admin UI 을 제공하지 않는다.
- 스케쥴링 실행에 대한 History 를 관리하지 않는다.
- Fixed Delay 타입을 보장하지 않아, 추가 작업이 필요하다.

## Quartz 구성
### Job 
- 실행 해야 할 작업을 의미한다.
- `Job` 인터페이스는 `execute` 메서드를 가지고 있고, 이를 구현해서 처리할 작업을 정의한다.

	```java
	public interface Job {
        void execute(JobExecutionContext var1) throws JobExecutionException;
    }
	```  

- `Job` 의 `Trigger` 가 발생해서 스케쥴러는 `JobExecutionContext` 객체를 넘겨주면서 `execute` 메서드를 호출한다.
- `JobExecutionContext` 에는 `Scheduler`, `Trigger`, `JobDetail` 등의 정보가 있다.
	
	```java
	public interface JobExecutionContext {
        Scheduler getScheduler();
    
        Trigger getTrigger();
    
        Calendar getCalendar();
    
        boolean isRecovering();
    
        TriggerKey getRecoveringTriggerKey() throws IllegalStateException;
    
        int getRefireCount();
    
        JobDataMap getMergedJobDataMap();
    
        JobDetail getJobDetail();
    
        Job getJobInstance();
    
        Date getFireTime();
    
        Date getScheduledFireTime();
    
        Date getPreviousFireTime();
    
        Date getNextFireTime();
    
        String getFireInstanceId();
    
        Object getResult();
    
        void setResult(Object var1);
    
        long getJobRunTime();
    
        void put(Object var1, Object var2);
    
        Object get(Object var1);
    }
	```  
	
### JobDetail
- `Job` 을 실행하기 위한 상세 정보이고, `JobBuilder` 에 의해 생성된다.

	```java
	public interface JobDetail extends Serializable, Cloneable {
        JobKey getKey();
    
        String getDescription();
    
        Class<? extends Job> getJobClass();
    
        JobDataMap getJobDataMap();
    
        boolean isDurable();
    
        boolean isPersistJobDataAfterExecution();
    
        boolean isConcurrentExectionDisallowed();
    
        boolean requestsRecovery();
    
        Object clone();
    
        JobBuilder getJobBuilder();
    }
	```  

### JobDataMap
- `Job` 을 실행할 때 사용해야 하거나, 사용이 필요한 정보를 담을 수 있는 객체이다.
- `JobDetail` 을 생성할 때 추가해서 사용할 수 있다.

### JobBuilder
- `JobDetail` 을 생성할때 사용하고, 빌터 패턴을 사용한다.

### Trigger
- `Job` 실행하기 위한 작업 실행 주기, 횟수 등에 대한 정보이다.
- `Scheduler` 는 `Trigger` 정보를 통해 `Job` 을 실행 시킨다.
- 한개의 `Job` 은 필수적으로 한개의 `Trigger` 를 가져야 한다.
- 한개의 `Job` 은 여러개의 `Trigger` 를 가질 수도 있다.
- `Trigger` 를 표현하는 방법은 `SimpleTrigger` 과 `CronTrigger` 가 있다.
	- `SimpleTrigger` : 특정 시간에 `Job` 을 수행해야 할때 사용하고, 반복 횟수나 실행 간격을 설정할 수 있다.
	- `CronTrigger` : `linux(unix)` 시스템의 [`crontab`](https://www.freeformatter.com/cron-expression-generator-quartz.html) 에서 사용하는 표현식을 사용해서 더 복잡한 설정이 가능하다.
- `Trigger` 에는 `TriggerState` 라는 트리거 상태 값이 저장되어 있다.

### TriggerBuilder
- `Trigger` 를 생성할때 사용하고, 빌더 패턴을 사용한다.

## Scheduler
- `SchedulerFactory` 를 통해 생성할 수 있고, Quartz Job Scheduler 의 핵심 객체이다.
- `JobDetail` 과 `Trigger` 를 관리한다.

```java
// 생성시 사용했던 JobKey 를 사용해서 Job을 조회 한다.
JobDetail getJobDetail(JobKey var1) throws SchedulerException;

// Job 의 그룹 리스트를 조회 한다.
List<String> getJobGroupNames() throws SchedulerException;

// Job 에 등록된 Trigger 리스트를 조회 한다.
List<? extends Trigger> getTriggersOfJob(JobKey var1) throws SchedulerException;

// Job 을 생성한다. 기존에 존재할 경우 boolean 값을 통해 갱신 여부를 설정할 수 있다. true 이면 기존 Job 을 업데이트 한다.
Date scheduleJob(JobDetail var1, Trigger var2) throws SchedulerException;

Date scheduleJob(Trigger var1) throws SchedulerException;

void scheduleJobs(Map<JobDetail, Set<? extends Trigger>> var1, boolean var2) throws SchedulerException;

void scheduleJob(JobDetail var1, Set<? extends Trigger> var2, boolean var3) throws SchedulerException;

// Job 을 중지한다. Job 에 연결되어 있는 Trigger 를 모두 중지시킨다.
void pauseJob(JobKey var1) throws SchedulerException;

void pauseJobs(GroupMatcher<JobKey> var1) throws SchedulerException;

// 중지 되었던 Job 을 재시작 시킨다.
void resumeJob(JobKey var1) throws SchedulerException;

void resumeJobs(GroupMatcher<JobKey> var1) throws SchedulerException;
```  

## Misfire Instructions
- `Misfire` 은 `Job` 이 실행되야 할때(fire time) 실행 되지 못했을 떄를 뜻한다.
- 가용한 스레드가 없는 등 다양한 경우에 발생할 수 있다.
- `Misfire` 가 된 `Trigger` 처리에 대한 Policy 를 지원한다.
	- `MISFIRE_INSTRUCTION_FIRE_NOW` : 바로 실행
	- `MISFIRE_INSTRUCTION_DO_NOTHING` : 아무 수행하지 않음
	
### Listener
- `Scheduler` 이벤트를 받을 수 있도록 제공되는 인터페이스이다.
- `JobListener` 는 `Job` 실행 전, 후에 이벤트를 받을 수 있다.

	```java
	public interface JobListener {
        String getName();
    
        void jobToBeExecuted(JobExecutionContext var1);
    
        void jobExecutionVetoed(JobExecutionContext var1);
    
        void jobWasExecuted(JobExecutionContext var1, JobExecutionException var2);
    }
	```  
	
- `TriggerListener` 는 `Trigger` 가 발생, 완료, `Misfire` 발생에 이벤트를 받을 수 있다.

	```java
	public interface TriggerListener {
        String getName();
    
        void triggerFired(Trigger var1, JobExecutionContext var2);
    
        boolean vetoJobExecution(Trigger var1, JobExecutionContext var2);
    
        void triggerMisfired(Trigger var1);
    
        void triggerComplete(Trigger var1, JobExecutionContext var2, CompletedExecutionInstruction var3);
    }
	```  
	
### JobStore
- `Job`, `Trigger` 에 대한 정보는 메모리, DB, Redis 등 다양한 저장소에 저장 할 수 있다.
- `RAMJobStore`
    - 메모리에 저장하는 옵션으로 기본설정 값이다.
    - 성능적으로 유리하지만, 애플리케이션이 종료됐을 때 데이터 유지가 되지 않는다.
- `JDBCJobStore`
	- DB 에 정보를 저장한다.
	- 애플리케이션 종료 시에도 저장된 정보의 유지가 가능하다.
- [`RedisJobStore`](https://github.com/RedisLabs/redis-quartz) 은 별도로 구현이 필요하다. 
- [`MongoDbJobStore`](https://github.com/michaelklishin/quartz-mongodb) 도 별도로 구현이 필요하다.


## Spring Boot Quartz 사용하기
- Quartz 관련 설정은 `application.properties`, `application.yml` 파일을 통해 할 수 있다.
	- [설정 참조](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigMain.html)
	- `spring.quartz.properties.` 프리픽스를 붙여 설정에 사용한다. (`spring.quartz.properties.org.quartz.threadPool.threadCount=1`)
- 예제에서는 아무 설정도 하지 않았을 때 기본 설정으로 사용법에 대해 알아본다.
	- `JobStore` 은 `RANJobStore`
	- 10 개의 쓰레드 사용
	
- `CronJob`
	- `CronTrigger` 를 사용해서 설정된 `Job` 의 실행 구현체이다.
	
	```java
	@Slf4j
	public class CronJob implements Job {
	    public static int count = 0;
	
	    @Override
	    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
	        count++;
	        Util.jobLog(jobExecutionContext);
	    }
	}
	```  
	
- `SimpleJob`
	- `SimpleTrigger` 를 사용해서 설정된 `Job` 의 실행 구현체이다.
	
	```java
	@Slf4j
	public class SimpleJob implements Job {
	    public static int count = 0;
	
	    @Override
	    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
	        count++;
	        Util.jobLog(jobExecutionContext);
	    }
	}
	```  
	
- `Util`
	- 유틸성 클래스로 `Job` 이 실행될 때 남기는 로그를 정형화 시킨 메서드가 있다.
	
	```java
	@Slf4j
	public class Util {
	    public static void jobLog(JobExecutionContext jobExecutionContext) {
	        JobDataMap jobDataMap = jobExecutionContext.getJobDetail().getJobDataMap();
	        log.info("JobKey : {}, TriggerKey : {}, FireTime : {}, JobData : {}",
	                jobExecutionContext.getJobDetail().getKey(),
	                jobExecutionContext.getTrigger().getKey(),
	                jobExecutionContext.getFireTime(),
	                jobDataMap.getString("registerTime"));
	    }
	}
	```  

### Quartz 라이브러리를 이용한 Schedule 구성
- `QuartzSchedule`
	- Quartz 라이브러리를 이용해서 스케쥴러를 구성한 클래스
	
	```java
	@Component
	@RequiredArgsConstructor
	public class QuartzSchedule {
	    private final QuartzTriggerListener quartzTriggerListener;
	    private final QuartzJobListener quartzJobListener;
	
	    @PostConstruct
	    public void start() throws Exception {
	        // 스케쥴러 시작
	        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
	        scheduler.start();
	
	        // Job 에서 사용할 데이터
	        JobDataMap jobDataMap = new JobDataMap();
	        jobDataMap.put("registerTime", LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
	
	        // CronTrigger 를 사용하는 CronJob 생성 및 구성
	        JobKey cronJobKey = new JobKey("QuartzCronJob", "Quartz");
	        JobDetail cronJobDetail = JobBuilder
	                .newJob(CronJob.class)
	                .withDescription("QuartzCronJob")
	                .withIdentity(cronJobKey)
	                .setJobData(jobDataMap)
	                .build();
	        TriggerKey cronJobTriggerKey = new TriggerKey("QuartzCronJobTrigger", "Quartz");
	        Trigger cronTrigger = TriggerBuilder
	                .newTrigger()
	                // 매초 마다 실행
	                .withSchedule(CronScheduleBuilder.cronSchedule("* * * * * ? *"))
	                .withIdentity(cronJobTriggerKey)
	                .build();
	
	        // SimpleTrigger 를 사용하는 SimpleJob 생성 및 구성
	        JobKey simpleJobKey = new JobKey("QuartzSimpleJob", "Quartz");
	        JobDetail simpleJobDetail = JobBuilder
	                .newJob(SimpleJob.class)
	                .withDescription("QuartzSimpleJob")
	                .withIdentity(simpleJobKey)
	                .setJobData(jobDataMap)
	                .build();
	        TriggerKey simpleJobTriggerKey = new TriggerKey("QuartzSimpleJobTrigger", "Quartz");
	        Trigger simpleTrigger = TriggerBuilder
	                .newTrigger()
	                // 매초 마다 실행
	                .withSchedule(SimpleScheduleBuilder
	                        .repeatSecondlyForever(1))
	                .withIdentity(simpleJobTriggerKey)
	                .build();
	
	        // TriggerListener 등록
	        // 그룹으로 지정 가능
	//        scheduler.getListenerManager().addTriggerListener(this.sampleTriggerListener, GroupMatcher.triggerGroupContains("Quartz"));
	        // CronTrigger 에만 리스너 등록
	        scheduler.getListenerManager().addTriggerListener(this.quartzTriggerListener, KeyMatcher.keyEquals(cronJobTriggerKey));
	
	        // JobListener 등록
	        // 그룹으로 지정 가능
	//        scheduler.getListenerManager().addJobListener(this.sampleJobListener, GroupMatcher.jobGroupContains("Quartz"));
	        // CronJob 에만 리스터 등록
	        scheduler.getListenerManager().addJobListener(this.quartzJobListener, KeyMatcher.keyEquals(cronJobKey));
	
	        // Job 등록
	        scheduler.scheduleJob(cronJobDetail, cronTrigger);
	        scheduler.scheduleJob(simpleJobDetail, simpleTrigger);
	    }
	}
	```  
	
- `QuartzTriggerListener`

	```java
	@Component
	@Slf4j
	public class QuartzTriggerListener implements TriggerListener {
	    @Override
	    public String getName() {
	        return this.getClass().getName();
	    }
	
	    @Override
	    public void triggerFired(Trigger trigger, JobExecutionContext jobExecutionContext) {
	        log.info("triggerFired : {}", trigger.getJobKey());
	    }
	
	    @Override
	    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext jobExecutionContext) {
	        /*
	        현재 Trigger 를 veto(거부, 금지) 할지 결정하는 메서드이다.
	        true 이면 veto 로 Job 이 실행 되지 않는다.
	        false 이면 Job 이 정상적으로 실행된다.
	         */
	        return false;
	    }
	
	    @Override
	    public void triggerMisfired(Trigger trigger) {
	        log.info("triggerMisfired : {}", trigger.getJobKey());
	    }
	
	    @Override
	    public void triggerComplete(Trigger trigger, JobExecutionContext jobExecutionContext, Trigger.CompletedExecutionInstruction completedExecutionInstruction) {
	        log.info("triggerComplete : {} start {} end {}", trigger.getJobKey(), trigger.getStartTime(), trigger.getEndTime());
	    }
	}
	```  
	
- `QuartzJobListener`

	```java
	@Component
	@Slf4j
	public class QuartzJobListener implements JobListener {
	    @Override
	    public String getName() {
	        return this.getClass().getName();
	    }
	
	    @Override
	    public void jobToBeExecuted(JobExecutionContext jobExecutionContext) {
	        log.info("jobToBeExecuted : {}", jobExecutionContext.getJobDetail().getKey());
	    }
	
	    @Override
	    public void jobExecutionVetoed(JobExecutionContext jobExecutionContext) {
	        log.info("jobExecutionVoted : {}", jobExecutionContext.getJobDetail().getKey());
	    }
	
	    @Override
	    public void jobWasExecuted(JobExecutionContext jobExecutionContext, JobExecutionException e) {
	        log.info("jobWasExecuted : {}", jobExecutionContext.getJobDetail().getKey());
	    }
	}
	```  
	
- UnitTest 코드

	```java
	@RunWith(SpringRunner.class)
	@SpringBootTest
	public class QuartzJobTest {
	    @Test
	    public void cronJob() throws Exception {
	        // given
	        JobDataMap jobDataMap = new JobDataMap();
	        jobDataMap.put("registerTime", LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
	        JobDetail jobDetail = JobBuilder
	                .newJob(CronJob.class)
	                .withDescription("QuartzCronJob")
	                .setJobData(jobDataMap)
	                .build();
	
	        Trigger trigger = TriggerBuilder
	                .newTrigger()
	                .withSchedule(CronScheduleBuilder.cronSchedule("* * * * * ? *"))
	                .build();
	
	        // when
	        Scheduler defaultScheduler = StdSchedulerFactory.getDefaultScheduler();
	        defaultScheduler.start();
	        defaultScheduler.scheduleJob(jobDetail, trigger);
	        Thread.sleep(3 * 1000);
	        defaultScheduler.shutdown(true);
	
	        // then
	        assertThat(CronJob.count, is(4));
	    }
	
	    @Test
	    public void simpleJob() throws Exception {
	        // given
	        JobDataMap jobDataMap = new JobDataMap();
	        jobDataMap.put("registerTime", LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
	        JobDetail jobDetail = JobBuilder
	                .newJob(SimpleJob.class)
	                .withDescription("QuartzSimpleJob")
	                .setJobData(jobDataMap)
	                .build();
	
	        Trigger trigger = TriggerBuilder
	                .newTrigger()
	                .withSchedule(SimpleScheduleBuilder
	                        .repeatSecondlyForever(1))
	                .build();
	
	        // when
	        Scheduler defaultScheduler = StdSchedulerFactory.getDefaultScheduler();
	        defaultScheduler.start();
	        defaultScheduler.scheduleJob(jobDetail, trigger);
	        Thread.sleep(3 * 1000);
	        defaultScheduler.shutdown(true);
	
	        // then
	        assertThat(SimpleJob.count, is(4));
	    }
    }
	```  
	
- 구현된 애플리케이션을 빌드 및 실행 시키면 아래와 같은 로그를 확인 할 수 있다.

	```
	2019-11-27 23:29:56.001  INFO 12620 --- [eduler_Worker-5] c.e.s.b.o.QuartzTriggerListener          : triggerFired : Quartz.QuartzCronJob
    2019-11-27 23:29:56.001  INFO 12620 --- [eduler_Worker-5] c.e.s.b.onlyquartz.QuartzJobListener     : jobToBeExecuted : Quartz.QuartzCronJob
    2019-11-27 23:29:56.001  INFO 12620 --- [eduler_Worker-5] c.e.s.baeldung.Util                      : JobKey : Quartz.QuartzCronJob, TriggerKey : Quartz.QuartzCronJobTrigger, FireTime : Wed Nov 27 23:29:56 KST 2019, JobData : 2019-11-27T23:29:54.614
    2019-11-27 23:29:56.001  INFO 12620 --- [eduler_Worker-5] c.e.s.b.onlyquartz.QuartzJobListener     : jobWasExecuted : Quartz.QuartzCronJob
    2019-11-27 23:29:56.001  INFO 12620 --- [eduler_Worker-5] c.e.s.b.o.QuartzTriggerListener          : triggerComplete : Quartz.QuartzCronJob start Wed Nov 27 23:29:54 KST 2019 end null
    2019-11-27 23:29:56.621  INFO 12620 --- [eduler_Worker-6] c.e.s.baeldung.Util                      : JobKey : Quartz.QuartzSimpleJob, TriggerKey : Quartz.QuartzSimpleJobTrigger, FireTime : Wed Nov 27 23:29:56 KST 2019, JobData : 2019-11-27T23:29:54.614
    2019-11-27 23:29:57.001  INFO 12620 --- [eduler_Worker-7] c.e.s.b.o.QuartzTriggerListener          : triggerFired : Quartz.QuartzCronJob
    2019-11-27 23:29:57.001  INFO 12620 --- [eduler_Worker-7] c.e.s.b.onlyquartz.QuartzJobListener     : jobToBeExecuted : Quartz.QuartzCronJob
    2019-11-27 23:29:57.001  INFO 12620 --- [eduler_Worker-7] c.e.s.baeldung.Util                      : JobKey : Quartz.QuartzCronJob, TriggerKey : Quartz.QuartzCronJobTrigger, FireTime : Wed Nov 27 23:29:57 KST 2019, JobData : 2019-11-27T23:29:54.614
    2019-11-27 23:29:57.001  INFO 12620 --- [eduler_Worker-7] c.e.s.b.onlyquartz.QuartzJobListener     : jobWasExecuted : Quartz.QuartzCronJob
    2019-11-27 23:29:57.001  INFO 12620 --- [eduler_Worker-7] c.e.s.b.o.QuartzTriggerListener          : triggerComplete : Quartz.QuartzCronJob start Wed Nov 27 23:29:54 KST 2019 end null
    2019-11-27 23:29:57.622  INFO 12620 --- [eduler_Worker-8] c.e.s.baeldung.Util                      : JobKey : Quartz.QuartzSimpleJob, TriggerKey : Quartz.QuartzSimpleJobTrigger, FireTime : Wed Nov 27 23:29:57 KST 2019, JobData : 2019-11-27T23:29:54.614
	```  
	
	- `CronJob` 의 경우 `TriggerListener` 와 `JobListener` 가 등록되어 있기 때문에 `Job` 이 실행될 떄 마다 5개의 로그가 출력된다.
	- `SimpleJob` 의 경우 `Job` 실행 마다 출력하는 로그는 1개이다.



---
## Reference
[Scheduling in Spring with Quartz](https://www.baeldung.com/spring-quartz-schedule)   
[Bootiful Quartz](https://brunch.co.kr/@springboot/53)   
[Quartz Job Scheduler란?](https://advenoh.tistory.com/51)   
