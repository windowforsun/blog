--- 
layout: single
classes: wide
title: "[Spring 실습] Spring TaskExecutor 로 동시성 적용하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring TaskExecutor 를 사용해서 스레드 기반 동시성 프로그램을 개발해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-core
    - TaskExecutor
---  

# 목표
- Thread Concurrent(스레드 기반 동시성) 프로그램을 Spring 으로 개발한다.

# 방법
- Spring TaskExecutor 를 통해 Thread Concurrent 프로그램을 개발할 수 있다.
- TaskExecutor 는 기본 Java 의 Executor, CommonJ 의 WorkManager 등 다양한 구현체를 제공하며 필요하면 커스텀 구현체를 만들어 사용할 수도 있다.
- Spring 은 이런 구현체를 모두 Java Executor 인터페이스로 단일화 하였다.

# 예제
- Java SE 가 제공하는 표준 Thread 를 이용해 개발하다보면 어려움에 빠지기 쉽다.
- Concurrent 는 Server-Side 컴포넌트에서는 중요하지만, Enterprise Java 에서는 표준 이라할게 없다.
- Thread 를 명시적으로 생성하거나 조작하는 행위를 금지하는 내용만 Java EE 명세에 포함되어 있다.
- Java Thread 및 Concurrent 의 역사
	- JDK 1.0 에 표준 java.lang.Thread
	- JDK 1.3 에서는 작업을 주기적으로 실행시킬 수 있는 java.util.TimerTask 
	- JDK 1.5 java.util.concurrent, java.util.concurrent.Executor 를 중심으로 ThreadPool 개편됨
	
## Java SE Executor 및 그 이하 API

```java
package java.util.concurrent;

public interface Executor {
	void execute(Runnable command);
}
```  

- Thread 관리 기능이 강화된 ExecutorService 이하 인터페이스는 shutdown() 메서드 처럼 Thread 에 이벤트를 일으키는 메서드를 제공한다.

	```java
	package java.util.concurrent;

	public interface ExecutorService extends Executor {
		 void shutdown();
		 <T> Future<T> submit(Callable<T> task);
		 <T> Future<T> submit(Runnable task, T result);
		 
		 // ...
	}
	```
- Java 5 부터 java.util.concurrent 패키지의 정적 팩토리 메서드를 사용할 수 있는 구현 클래스가 추가되었다.
- ExecutorService 클래스 에는 Future<T> 형 객체를 반환하는 submit() 메서드가 있다.
- Future<T> 인스턴스는 비동기 실행 Thread 의 진행 상황을 추적하는 용도로 사용된다.
- Future.isDone(), Future.isCancelled() 메서드는 각각 어떤 Job 이 완료되었는지, 취소되었는지 확인한다.

```java
Runnable task = new Runnable() {
	public void run() {
		try {
			Thread.sleep(60 * 1000);
			System.out.println("Thread sleep one minute!");
		} catch(Exception e) {
			
		}
	}
};

ExecutorService executorService = Executors.newCachedThreadPool();

if(executorService.submit(task, Boolean.TRUE).get().equal(Boolean.TRUE)) {
	System.out.println("Task finished");
}
```  

- run() 메서드가 반환형 없는 Runnable 인스턴스 내부에서 ExecutorService 와 submit() 메서드를 사용할 경우, 반환된 Future 의 get() 메서드를 호출하면 null 또는 전송 시 지정한 값이 반환된다.
- Runnable 을 응용한 시간 경과 구현 클래스

```java
public class DemonstrationRunnable implements Runnable {

    public void run() {

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName());
        System.out.printf("Hello at %s \n", new Date());
    }
}
```  

- DemonstrationRunnable 인스턴스를 Java Executors 에서 사용한 예제

```java
public class ExecutorsDemo {

    public static void main(String[] args) throws Throwable {
        Runnable task = new DemonstrationRunnable();

        // 스레드 풀을 생성하고 가급적 이미 생성된 스레드를 사용하려고 시도합니다.
        ExecutorService cachedThreadPoolExecutorService = Executors
                .newCachedThreadPool();
        if (cachedThreadPoolExecutorService.submit(task).get() == null)
            System.out.printf("The cachedThreadPoolExecutorService "
                    + "has succeeded at %s \n", new Date());

        // 생성 스레드 개수를 제한하고 나머지 스레드는 큐잉합니다.
        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(100);
        if (fixedThreadPool.submit(task).get() == null)
            System.out.printf("The fixedThreadPool has " +
                            "succeeded at %s \n",
                    new Date());

        // 한번에 한 스레드만 사용합니다.
        ExecutorService singleThreadExecutorService = Executors
                .newSingleThreadExecutor();
        if (singleThreadExecutorService.submit(task).get() == null)
            System.out.printf("The singleThreadExecutorService "
                    + "has succeeded at %s \n", new Date());

        // 캐시된 스레드 풀을 사용합니다.
        ExecutorService es = Executors.newCachedThreadPool();
        if (es.submit(task, Boolean.TRUE).get().equals(Boolean.TRUE))
            System.out.println("Job has finished!");

        // 타이머 기능을 모방합니다.
        ScheduledExecutorService scheduledThreadExecutorService = Executors
                .newScheduledThreadPool(10);
        if (scheduledThreadExecutorService.schedule(
                task, 30, TimeUnit.SECONDS).get() == null)
            System.out.printf("The scheduledThreadExecutorService "
                    + "has succeeded at %s \n", new Date());

        // 다음 문은 예외가 발생하거나 취소되지 않는 한 계속 실행됩니다.
        scheduledThreadExecutorService.scheduleAtFixedRate(task, 0, 5,
                TimeUnit.SECONDS);
    }
}
```  

- ExecutorService 하위 인터페이스에서 Callable<T> 를 인수로 받는 submit() 메서드를 호출하면 Callable 의 call() 메서드가 반환한 값을 그대로 다시 반환한다.

```java
package java.util.concurrent;

public interface Callable<V> {
    V call() throws Exception;
}
```  

- Java EE 에서는 Thread 를 다루는 설계 자체가 금기시되어 이런 부류의 문제들을 다른 방식으로 접근해 해결해 왔다.
	- Quartz(Job Scheduling Framework) 는 Thread/Concurrent 에 관한 Java 의 부족한 부분을 채우고자 처음으로 고안된 솔루션이다.
	- IBM, BEA 등에서 다양한 관련 솔루션들이 등장했고, CommonJ API 라는 오픈소스도 등장했다.
	- Java SE 와는 달리 관리되는 환경에서 컴포넌트에 동시성을 부여하고 Thread 를 제어할 수 있는 간단하면서 이식성 좋은 표준이 없었다.

## Spring 의 TaskExecutor
- Spring 은 Java5의 java.util.concurrent.Executor 를 상속한 org.springframework.core.task.TaskExecutor 인터페이스라는 통합 솔루션을 제공한다.
- TaskExecutor 인터페이스는 Spring Framework 내부에서 다양하게 사용된다.
	- Spring Quartz 연계 및 Message-Driven POJO 컨테이너 지원 기능 등
	
```java
package org.springframework.core.task;

public interface TaskExecutor extends Executor {
	@Override
	void execute(Runnable task);
}

```  

- TaskExecutor Java SE 및 Java EE 를 기분으로 한 여러 솔루션의 다리 역할을 할수 있다.
	- TaskExecutor 에서 Java SE 의 Executor, ExecutorService 를 사용할 수 있다.
- DemonstrationRunnable 를 Spring TaskExecutor 로 활용한 예제 이다.

```java
@Component
public class SpringExecutorsDemo {

    @Autowired
    private SimpleAsyncTaskExecutor asyncTaskExecutor;
    @Autowired
    private SyncTaskExecutor syncTaskExecutor;
    @Autowired
    private TaskExecutorAdapter taskExecutorAdapter;
    @Autowired
    private ThreadPoolTaskExecutor threadPoolTaskExecutor;
    @Autowired
    private DemonstrationRunnable task;

    @PostConstruct
    public void submitJobs() {
        syncTaskExecutor.execute(task);
        taskExecutorAdapter.submit(task);
        asyncTaskExecutor.submit(task);

        for (int i = 0; i < 500; i++)
            threadPoolTaskExecutor.submit(task);
    }

    public static void main(String[] args) {
        new AnnotationConfigApplicationContext(ExecutorsConfiguration.class)
                .registerShutdownHook();
    }

}
```  

- 위 클래스는 단순 POJO 클래스로 여러 TaskExecutor 인스턴스를 Autowired 하고, Runnable 인스턴스를 TaskExecutor 에 전달해준다.
- 아래는 위 클래스에서 사용하는 빈 인스턴스와 관련 설정 클래스이다.

```java
@Configuration
@ComponentScan
public class ExecutorsConfiguration {

    @Bean
    public TaskExecutorAdapter taskExecutorAdapter() {
        return new TaskExecutorAdapter(Executors.newCachedThreadPool());    
    }

    @Bean
    public SimpleAsyncTaskExecutor simpleAsyncTaskExecutor() {
        return new SimpleAsyncTaskExecutor();
    }

    @Bean
    public SyncTaskExecutor syncTaskExecutor() {
        return new SyncTaskExecutor();
    }

    @Bean
    public ScheduledExecutorFactoryBean scheduledExecutorFactoryBean(ScheduledExecutorTask scheduledExecutorTask) {
        ScheduledExecutorFactoryBean scheduledExecutorFactoryBean = new ScheduledExecutorFactoryBean();
        scheduledExecutorFactoryBean.setScheduledExecutorTasks(scheduledExecutorTask);
        return scheduledExecutorFactoryBean;
    }

    @Bean
    public ScheduledExecutorTask scheduledExecutorTask(Runnable runnable) {
        ScheduledExecutorTask scheduledExecutorTask = new ScheduledExecutorTask();
        scheduledExecutorTask.setPeriod(1000);
        scheduledExecutorTask.setRunnable(runnable);
        return scheduledExecutorTask;
    }

    @Bean
    public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(50);
        taskExecutor.setMaxPoolSize(100);
        taskExecutor.setAllowCoreThreadTimeOut(true);
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        return taskExecutor;
    }
}
```  

- 위 설정 클래스에서 다양한 TaskExecutor 구현체를 생성하는 방법이 나와있다.
- 대부분 단순한 방식이며 ScheduledExecutorFactoryBean 의 경우에만 팩토리 빈에 위임하여 실행을 자동 트리거한다.

### 다양한 TaskExecutor 구현
- TaskExecutorAdapter
	- java.util.concurrent.Executors 인스턴스를 감싼 단순 래퍼이므로 Spring TaskExecutor 인터페이스와 같은 방식으로 사용할 수 있다.
	- Spring 을 이용해 Executor 인스턴스를 구성하고, TaskExecutorAdapter 의 생성자 인수로 전달한다.
- SimpleAsyncTaskExecutor
	- 사용하는 Job 마다 Thread 를 새로 만들어 제공하며 Thread 를 Pooling 하거나 재사용하지 않는다.
	- 사용하는 각 Job 들은 Thread 에서 Async(비동기) 로 실행된다.
- SyncTaskExecutor
	- 가장 단순한 TaskExecutor 구현체이다.
	- 동기적으로 Thread 를 띄워 Job 을 실행한 다음, join() 메서드로 바로 연결한다.
	- Threading 은 하지 않고 호출 Thread 에서 run() 메서드를 수동 실행한 것과 같다.
- ScheduledExecutorFactoryBean
	- ScheduledExecutorTask 빈으로 정의된 잡을 자동 트리거 한다.
	- ScheduledExecutorTask 인스턴스 목록을 지정해서 여러 Job 을 동시 실행할 수도 있다.
	- ScheduledExecutorTask 인스턴스에는 직접 실행 간 공백 시간을 인수로 넣을 수 있다.
- ThreadPoolTaskExecutor
	- java.util.concurrent.ThreadPoolExecutor 를 기반으로 모든 기능이 있는 Thread Pool 구현체이다.
- TaskExecutor 지원 기능은 애플리케이션 서버에서 하나로 통합된 인터페이스를 사용해 서비스를 스케쥴링하는 강력한 방법이다.
- 모든 애플리케이션 서버(Tomcat, Jetty 등)에 배포 가능한 더 확실한 솔류션이 필요할 경우에는 Spring Quartz 를 고려하는 것도 하나의 방법이다. (다른 것들에 비해 무거울 수도 있다)
	
### 그 외 Thread/Concurrent
- IBM WebSphere 같은 애플리케이션 서버에서 사용가능한 CommonJ WorkManager/TimerManager 지원 기능을 이용해 애플리케이션을 개발할 때는 org.springframework.scheduling.commonj.WorkManagerTaskExecutor 를 사용한다.
	- WorkManagerTaskExecutor 는 WebSphere 내부에서 CommonJ WorkManager 레퍼런스에 할 일을 넘긴다.
	- 보통은 해당 리소스를 바라보는 JNDI 레퍼런스를 지정하게 된다.
- JEE7 부터는 javax.enterpirse.concurrent 패키지에 ManagerExecutorService 가 추가 됐다.
	- JEE7 호환 서버는 반드시 ManagerExecutorService 인스턴스를 제공하도록 규정되어 있다.
	- Spring TaskExecutor 지원 기능과 병행하려면 DefaultManagedTaskExecutor 를 구성해 기본 ManagerExecutorService 를 감지하게 하거나 개발자가 명시하면 된다.



---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
