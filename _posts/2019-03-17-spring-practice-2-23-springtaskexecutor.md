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
- Spring 은 Java5의 java.util.concurrent.Executor 를 상속한 org.springframeworks.core.task.TaskExecutor 인터페이스라는 통합 솔류선을 제공한다.
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

```







---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
