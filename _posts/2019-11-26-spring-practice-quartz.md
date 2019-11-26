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

### TriggerBuilder
- `Trigger` 를 생성할때 사용하고, 빌더 패턴을 사용한다.

## Scheduler
- `SchedulerFactory` 를 통해 생성할 수 있고, Quartz Job Scheduler 의 핵심 객체이다.
- `JobDetail` 과 `Trigger` 를 관리한다.

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




---
## Reference
[Scheduling in Spring with Quartz](https://www.baeldung.com/spring-quartz-schedule)   
[Bootiful Quartz](https://brunch.co.kr/@springboot/53)   
[Quartz Job Scheduler란?](https://advenoh.tistory.com/51)   
