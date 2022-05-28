--- 
layout: single
classes: wide
title: "[Java 개념] Java Thread State, Thread Dump"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Java Thread 와 그 상태 그리고 Thread 를 분석할 수 있는 Thread Dump 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Thread
  - Thread State
  - Thread Dump
  - VisualVM
toc: true 
use_math: true
---  

## Java Thread
`Java` 그리고 현대의 대다수 언어는 `Thread` 를 사용해서 동시성을 확보하고 성능을 끌어 올리는 전략을 사용한다. 
적개는 수십에서 많개는 수백, 수천까지도 시스템 자원이 허용하는 한 `Thread` 를 사용할 수 있다. 
하지만 `Thread` 를 통해 동시성을 확보할 수 있는 만큼 여러 `Thread` 가 같은 자원을 이용하는 상황에서는 
경합(Contention) 이 발생하고 최악의 경우에는 데드락(Deadlock) 까지 발생한다.  

경합은 `N` 개의 `Thread` 가 하나의 자원을 안전하게 이용하기 위해서는 자원의 통일성을 위해 `Lock` 을 사용하는데, 
이 `Lock` 을 얻기 위해 서로 기다리는 상태를 의미한다. (`Lock` 을 얻어야 자원을 사용할 수 있다.)  

그리고 데드락은 경합의 특별(최악)의 경우로 두개 이상의 `Thread` 가 서로 상대의 `Lock` 을 얻기위해 계속해서 대기하는 상황을 의미한다.  

개발자는 하나의 애플리케이션에서 사용되는 많은 `Thread` 의 상태와 문제점(경합, 데드락)을 파악하고 분석하기 위해서 스레드 덤프(Thread Dump) 를 사용할 수 있다.  


### 스레드 동기화
`Thread` 를 통해 동시성을 늘렸지만 `Thread` 가 사용하는 자원은 고유하다. 
여러 `Thread` 가 공유 자원을 사용할 때 정합성을 보장하기 위해 `스레드 동기화` 사용한다. 
`스레드 동기화` 는 하나의 자원에는 하나의 스레드만 접근할 수 있도록 해서 각 자원의 정합성을 보장한다. 
`Java` 는 이러한 `스레드 동기화` 를 `Monitor` 를 사용하는데, 
`Java` 의 모든 객체는 `Monitor` 를 가지고 있다. 
각 객채의 `Monitor` 는 하나의 `Thread`만 소유 가능하고 다른 `Thread` 는 `Monitor` 가 반환 될때 까지 
`Wait Queue` 에서 대기해야 한다.  

### 스레드 상태
[Enum Thread.State](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.State.html)
를 보면 `Java Thread` 를 관리하는 상태의 종류에 대해 확인 할 수 있다. 


![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-1.png)

|구분|상태|설명
|---|---|---
|객체 생성|NEW|스레드 객체 생성, `start()` 메소드 호출 전이므로 실행 전인 상태
|실행|RUNNABLE|`CPU` 자원을 점유해서 실행 중인 상태, `BLOCKED`, `WAITING`, `TIMED_WAITING` 상태로 전환 될 수 있다. 
|일시 정지|WAITING|`wait()`, `join()`, `park()` 메소드로 무한정 대기 중인 상태, 다른 스레드 완료 및 통지를 통해 빠져 나올 수 있다. 
| |TIMED_WAITING|`sleep()`, `wait()`, `join()`, `parkNano()`, `parkUntil()` 메소드로 일정 시간 대기중인 상태, `WAITING` 과 동일하게 빠져나올 수도 있고 최대 대기 시간 이후에는 빠져나오게 된다. 
| |BLOCKED|`Monitor` 를 획득하기 위해 다른 `Thread` 의 락 해제를 대기하고 있는 상태
|종료|TERMINATED|`Thread` 가 주어진 동작을 모두 수행하고 종료된 상태

도식화 된 그림은 좀더 자세하게 그리면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-2.png)


### 스레드 종류
`Java Thread` 는 크게 데몬 스레드(Daemon Thread) 와 비데몬 스레드(Non-Daemon Thread) 로 나뉜다. 

- `Daemon Thread` : 다른 `Non-Daemon Thread` 가 없다면 동작이 중지된다. 대표적으로 `Garbage Collector`, `JMX` 스레드 등이 처럼 `JVM` 이 필요에 의해 사용되는 스레드이다. 주로 `Daemon Thread` 는 무한루프와 조건문을 통해 실행 후 대기를 반복하는 방식으로 작성된다. 
- `Non-Daemon Thread` : `public static void main(String[] args)` 인 메인 스레드가 대표적이다. 즉 메인 스레드가 중지되면 `Daemon Thread` 에 해당하는 모든 스레드도 함께 중지된다. `Non-Daemon Thread` 는 사용자가 정의한 스레드도 포함된다. 


### 스레드 덤프
`Thread Dump` 를 획득하는 방법은 아래와 같은 방법들이 있다. 

#### jstack
먼저 `jps -v` 명령으로 현재 사용자가 실행한 `Java Application` 의 `PID` 값을 확인한다. 

```bash
$ jps -v
25544 ThreadDumpApplication -XX:TieredStopAtLevel=1 -Xverify:none -Dspring.output.ansi.enabled=always .. 생략 ..
```  

이후 `jstack <PID>` 로 `Thread Dump` 를 획득할 수 있다.  

```bash
$ jstack 25544
Full thread dump OpenJDK 64-Bit Server VM (11+28 mixed mode):

Threads class SMR info:
_java_thread_list=0x00000220d114cc40, length=18, elements={
0x00000220cf0df000, 0x00000220cf150800, 0x00000220cf1be800, 0x00000220cf1c3800,
0x00000220cf1c4800, 0x00000220cf1cf800, 0x00000220cf09c800, 0x00000220cfea0000,
0x00000220cfea4800, 0x00000220d00b7800, 0x00000220d0a14800, 0x00000220d09cf800,
0x00000220d106f800, 0x00000220d00cc000, 0x00000220d1227800, 0x000002208c2b2800,
0x00000220d00b9000, 0x00000220d1229000
}

"Reference Handler" #2 daemon prio=10 os_prio=2 cpu=0.00ms elapsed=87.47s tid=0x00000220cf0df000 nid=0x8194 waiting on condition  [0x0000008c7fafe000]
   java.lang.Thread.State: RUNNABLE
        at java.lang.ref.Reference.waitForReferencePendingList(java.base@11/Native Method)
        at java.lang.ref.Reference.processPendingReferences(java.base@11/Reference.java:241)
        at java.lang.ref.Reference$ReferenceHandler.run(java.base@11/Reference.java:213)

"Finalizer" #3 daemon prio=8 os_prio=1 cpu=0.00ms elapsed=87.47s tid=0x00000220cf150800 nid=0x654c in Object.wait()  [0x0000008c7fbfe000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(java.base@11/Native Method)
        - waiting on <0x0000000404401780> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11/ReferenceQueue.java:155)
        - waiting to re-lock in wait() <0x0000000404401780> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11/ReferenceQueue.java:176)
        at java.lang.ref.Finalizer$FinalizerThread.run(java.base@11/Finalizer.java:170)

"Signal Dispatcher" #4 daemon prio=9 os_prio=2 cpu=0.00ms elapsed=87.45s tid=0x00000220cf1be800 nid=0x9bd0 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

.. 생략 ..
```  

#### VisualVM
`VisualVM` 은 대표적은 `Java Application` 모니터링 `GUI` 프로그램이다.  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-3.png)  

좌측에서 모니터링하고자 하는 애플리케이션을 선택해주고 `Threads` 탭을 누른 다음 `Thread Dump` 버튼을 눌러준다. 

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-4.png)  

위와 같이 `Thread Dump` 결과를 확인 할 수 있다.  

#### kill
`Linux` 환경이라면 `kill` 명령어를 통해서도 `Thread Dump` 를 획득 할 수 있다. 
먼저 `ps -ef | grep java` 명령으로 현재 실행 중인 `Java Application` 목록 중 `PID` 를 확인 한다.  

```bash
$ ps -ef | grep java
windowf+   816     9 58 18:53 pts/0    00:00:07 java -jar thread-dump-1.0-SNAPSHOT.jar
```  

그리고 `kill -3 <PID>`(or `-QUIT`, `-SIGQUIT`) 명령을 수행해주면 `Java Application` 출력의 출력으로 `Thread Dump` 가 출력 된다.  

```bash
$ kill -3 816



.. Java Application Console ..

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.4)

2022-05-28 18:53:34.039  INFO 816 --- [           main] c.w.j.threaddump.ThreadDumpApplication   : Starting ThreadDumpApplication using Java 11.0.14 on AL01770947 with PID 816
2022-05-28 18:53:34.045  INFO 816 --- [           main] c.w.j.threaddump.ThreadDumpApplication   : No active profile set, falling back to 1 default profile: "default"
2022-05-28 18:53:36.411  INFO 816 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port 8080
2022-05-28 18:53:36.422  INFO 816 --- [           main] c.w.j.threaddump.ThreadDumpApplication   : Started ThreadDumpApplication in 3.157 seconds (JVM running for 3.793)
2022-05-28 18:54:54
Full thread dump OpenJDK 64-Bit Server VM (11.0.14+9-Ubuntu-0ubuntu2.20.04 mixed mode, sharing):

Threads class SMR info:
_java_thread_list=0x00007fa6988d74e0, length=12, elements={
0x00007fa71c3c7800, 0x00007fa71c3d1800, 0x00007fa71c3d9800, 0x00007fa71c3db800,
0x00007fa71c3dd800, 0x00007fa71c3e8000, 0x00007fa71c3ea000, 0x00007fa71c41d000,
0x00007fa71cba4000, 0x00007fa71d4ae000, 0x00007fa71d4ba000, 0x00007fa71c016000
}

"Reference Handler" #2 daemon prio=10 os_prio=0 cpu=1.72ms elapsed=81.63s tid=0x00007fa71c3c7800 nid=0x338 waiting on condition  [0x00007fa6e040f000]
   java.lang.Thread.State: RUNNABLE
        at java.lang.ref.Reference.waitForReferencePendingList(java.base@11.0.14/Native Method)
        at java.lang.ref.Reference.processPendingReferences(java.base@11.0.14/Reference.java:241)
        at java.lang.ref.Reference$ReferenceHandler.run(java.base@11.0.14/Reference.java:213)

"Finalizer" #3 daemon prio=8 os_prio=0 cpu=0.23ms elapsed=81.63s tid=0x00007fa71c3d1800 nid=0x339 in Object.wait()  [0x00007fa6e030e000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(java.base@11.0.14/Native Method)
        - waiting on <0x000000050a81e1a8> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11.0.14/ReferenceQueue.java:155)
        - waiting to re-lock in wait() <0x000000050a81e1a8> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11.0.14/ReferenceQueue.java:176)
        at java.lang.ref.Finalizer$FinalizerThread.run(java.base@11.0.14/Finalizer.java:170)

"Signal Dispatcher" #4 daemon prio=9 os_prio=0 cpu=0.34ms elapsed=81.63s tid=0x00007fa71c3d9800 nid=0x33a waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Service Thread" #5 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=81.63s tid=0x00007fa71c3db800 nid=0x33b runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

.. 생략 .. 

```  

만약 콘솔 로그로 출력하는게 아니라 파일로 `Thread Dump` 를 획득하고 싶다면 실행 인자에 `-XX:+UnlockDiagnosticVMOptions -XX:+LogVMOutput -XX:LogFile=~/jvm.log` 를 추가해 준다.  

```bash
$ java -jar -XX:+UnlockDiagnosticVMOptions -XX:+LogVMOutput -XX:LogFile=~/jvm.log thread-dump-1.0-SNAPSHOT.jar

$ ps -ef | grep java
windowf+   931   740  0 18:58 pts/1    00:00:00 grep --color=auto java

$ kill -3 931

$ ls | grep jvm.log
jvm.log

$ cat jvm.log
<?xml version='1.0' encoding='UTF-8'?>
<hotspot_log version='160 1' process='883' time_ms='1653731853765'>
<vm_version>
<name>
OpenJDK 64-Bit Server VM
</name>
<release>
11.0.14+9-Ubuntu-0ubuntu2.20.04
</release>
<info>
OpenJDK 64-Bit Server VM (11.0.14+9-Ubuntu-0ubuntu2.20.04) for linux-amd64 JRE (11.0.14+9-Ubuntu-0ubuntu2.20.04), built on Jan 25 2022 14:03:04 by &quot;unknown&quot; with gcc 9.3.0
</info>
</vm_version>
<vm_arguments>
<args>
-XX:+UnlockDiagnosticVMOptions -XX:+LogVMOutput -XX:LogFile=~/jvm.log </args>
<command>
thread-dump-1.0-SNAPSHOT.jar

.. 생략 .. 

Full thread dump OpenJDK 64-Bit Server VM (11.0.14+9-Ubuntu-0ubuntu2.20.04 mixed mode, sharing):

Threads class SMR info:
_java_thread_list=0x00007f44600956f0, length=12, elements={
0x00007f44d03c8800, 0x00007f44d03cb000, 0x00007f44d03d3000, 0x00007f44d03d5000,
0x00007f44d03d7000, 0x00007f44d03e1000, 0x00007f44d03e3000, 0x00007f44d0416000,
0x00007f44d0909000, 0x00007f44d147c000, 0x00007f44d1488000, 0x00007f44d0017800
}

&quot;Reference Handler&quot; #2 daemon prio=10 os_prio=0 cpu=1.85ms elapsed=43.85s tid=0x00007f44d03c8800 nid=0x37b waiting on condition  [0x00007f449414f000]
   java.lang.Thread.State: RUNNABLE
        at java.lang.ref.Reference.waitForReferencePendingList(java.base@11.0.14/Native Method)
        at java.lang.ref.Reference.processPendingReferences(java.base@11.0.14/Reference.java:241)
        at java.lang.ref.Reference$ReferenceHandler.run(java.base@11.0.14/Reference.java:213)

&quot;Finalizer&quot; #3 daemon prio=8 os_prio=0 cpu=0.27ms elapsed=43.85s tid=0x00007f44d03cb000 nid=0x37c in Object.wait()  [0x00007f448dffe000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(java.base@11.0.14/Native Method)
        - waiting on &lt;0x000000050a81e808&gt; (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11.0.14/ReferenceQueue.java:155)
        - waiting to re-lock in wait() &lt;0x000000050a81e808&gt; (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(java.base@11.0.14/ReferenceQueue.java:176)
        at java.lang.ref.Finalizer$FinalizerThread.run(java.base@11.0.14/Finalizer.java:170)

&quot;Signal Dispatcher&quot; #4 daemon prio=9 os_prio=0 cpu=0.45ms elapsed=43.85s tid=0x00007f44d03d3000 nid=0x37d waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   
.. 생략 ..
```  

### 스레드 덤프 내용
위 방법으로 `Thread Dump` 를 획득하면 아래와 같은 내용을 확인 할 수 있다.  

```
"reactor-http-epoll-1" #25 daemon prio=5 os_prio=0 cpu=2.69ms elapsed=39.98s tid=0x00007f44d147c000 nid=0x39e runnable  [0x00007f44129b6000]
   java.lang.Thread.State: RUNNABLE
        at io.netty.channel.epoll.Native.epollWait(Native Method)
        at io.netty.channel.epoll.Native.epollWait(Native.java:193)
        at io.netty.channel.epoll.Native.epollWait(Native.java:186)
        at io.netty.channel.epoll.EpollEventLoop.epollWaitNoTimerChange(EpollEventLoop.java:290)
        at io.netty.channel.epoll.EpollEventLoop.run(EpollEventLoop.java:347)
        at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:986)
        at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74)
        at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
        at java.lang.Thread.run(java.base@11.0.14/Thread.java:829)
```  

위 `Thread Dump` 를 구성하는 내용은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-5.png)  

- `Thread Name` : 스레드 이름으로 사용자가 정의한 이름으로도 노출된다. 일반적으로 분석에 용의한 이름을 미리 정의해 주는게 좋다. 
- `ID` : `JVM` 내에서 각 스레드에 할당된 고유한 `ID` 이다. 
- `Thread Type` : `daemon` , `non-daemon` 등의 스레드의 타입이다. 
- `Thread Priority` : `Java` 에서 스레드의 우선순위 이다. 
- `OS Thread Priority` : `Java Thread` 는 곧 `OS Thread` 와 매핑 되는데, 이때 `OS Thread` 의 우선순위이다. 
- `CPU Usage` : 해당 스레드가 `CPU` 를 사용한 시간을 의미한다. 
- `Total Times` : 해당 스레드가 생성된 총 시간을 의미한다. 
- `Java Level Thread ID` : `JVM`(`JNI` 코드) 에서 관리하는 `Native Thread` 구조체의 포인터 주소이다. 
- `Native Thread ID` : `Java Thread` 와 매핑된 `OS Thread` 의 `ID` 로, `Windows` 의 경우 `OS Level` 의 `Thread ID` 이고, `Linux` 에서는 `LWP`(`Light Weight process`) 의 `ID` 를 의미한다. 
- `Enum Thread State` : 해당 스레드의 상태에 해당하는 `Enum` 값을 의미한다.  
- `Thread State` : 해당 스레드의 상태를 의미한다. 
- `Last Known Java Stack Pointer` : 스레드의 현재 `Stack Pointer`(SP) 의 주소를 의미한다. 
- `Call Stack` : 해당 스레드가 수행되는 함수들의 호출 관계를 표현하는 정보이다. 

### Thread Dump 분석 예제
몇가지 예제를 진행하며 상황에 따른 `Thread Dump` 를 분석하는 방법에 대해 살펴 본다. 
`VisualVM` 과 `Thread Dump` 등을 함께 사용해서 분석을 진행한다.  

`VisualVM` 에서 사용하는 스레드 상태의 종류는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-9.png)  

#### Running 
`Thread` 가 중간에 대기 상태 없이 100% 사용된 경우이다. 

```java
Thread runningThread = new Thread(() -> {
	while(true) {}
}, "myRunningThread");

runningThread.start();
runningThread.join();
```  


![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-6.png)  

```
"myRunningThread" #28 prio=5 os_prio=0 [CPU Usage]cpu=21203.13ms [Total Times]elapsed=21.23s tid=0x000001adf0820800 nid=0x4138 [Thread State]runnable  [0x000000a7746fe000]
   java.lang.Thread.State: [Enum Thread State]RUNNABLE
        at com.windowforsun.javathread.threaddump.ThreadStateTest.lambda$running$0(ThreadStateTest.java:71)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$415/0x0000000800223c40.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)
```  

`CPU Usage` 와 `Total Times` 시간이 동일한 것을 확인 가능하고, 
`Thread State`, `Enum Thread State` 또한 `RUNNABLE` 인것을 확인 할 수 있다.  

#### Sleep
`Thread` 가 계속해서 `sleep()` 메소드로 인해 `TIMED_WAITING` 상태가 된 경우이다.  

```java
Thread sleepThread = new Thread(Utils.sleep, "mySleepThread");
	
sleepThread.start();
sleepThread.join();
```  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-7.png)

```
"mySleepThread" #28 prio=5 os_prio=0 [CPU Usage]cpu=0.00ms [Total Times]elapsed=21.88s tid=0x0000022df6287800 nid=0x68d4 [Thread State]waiting on condition  [0x000000a0f20fe000]
   java.lang.Thread.State: [Enum Thread State]TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(java.base@11/Native Method)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:10)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:5)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$415/0x0000000800223c40.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)

```  

`Total Times` 를 보면 `Thread` 는 총 25초 동안 실행 됐지만, 
계속 `Sleep` 상태 였기 때문에 `CPU Usage` 은 0인 것을 확인 할 수 있다. 
그리고 `Thread State` 는 `waiting on condition` 이고, 
`Enum Thread State` 는 `TIMED_WAITING (sleeping)` 인 것을 확인 할 수 있다.  

#### Thread.wait
`Thread.wait` 메소드를 통해 `Thread` 가 다른 `Thread` 의 종료를 대기하는 경우이다.  

```java
Thread sleepThread = new Thread(() -> {
	synchronized (this) {
		Utils.sleep();
	}
}, "mySleepThread");

Thread waitThread = new Thread(() -> {
	synchronized (sleepThread) {
		try {
			sleepThread.wait();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}, "myWaitThread");

sleepThread.start();
Thread.sleep(100);
waitThread.start();
sleepThread.join();
waitThread.join();
```  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-8.png)

```
mySleepThread" #28 prio=5 os_prio=0 cpu=0.00ms elapsed=21.63s tid=0x00000202f115e800 nid=0x78b0 waiting on condition  [0x000000e48c6fe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(java.base@11/Native Method)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:10)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:5)
        at com.windowforsun.javathread.threaddump.ThreadStateTest.lambda$threadWait_waiting$6(ThreadStateTest.java:150)
        - locked <0x0000000442779d10> (a com.windowforsun.javathread.threaddump.ThreadStateTest)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$415/0x0000000800224c40.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)


"myWaitThread" #29 prio=5 os_prio=0 [CPU Usage]cpu=0.00ms [Total Times]elapsed=21.53s tid=0x00000202f0ca3800 nid=0x9a4c in [Thread State]Object.wait()  [0x000000e48c8fe000]
   java.lang.Thread.State: [Enum Thread State]WAITING (on object monitor)
        at java.lang.Object.wait(java.base@11/Native Method)
        - waiting on <0x0000000441bec7e0> (a java.lang.Thread)
        at java.lang.Object.wait(java.base@11/Object.java:328)
        at com.windowforsun.javathread.threaddump.ThreadStateTest.lambda$threadWait_waiting$7(ThreadStateTest.java:156)
        - waiting to re-lock in wait() <0x0000000441bec7e0> (a java.lang.Thread)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$416/0x0000000800225040.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)
```  

`Thread` 시작 부터 계속 `mySleepThread` 가 끝나기를 대기하고 있었기 때문에, 
`Total Times` 는 21초 이지만, `CPU Usage` 는 0초이다. 
그리고 `Thread State` 는 `Object.wait()` 로 대기 중인 것을 나타내고 있고, 
`Enum Thread State` 또한 `WAITING (on object monitor)` 으로 다른 대기 상태를 나타내고 있다.  


#### synchronized
`Java Synchronized` 동기화 블럭을 다른 스레드가 락을 얻기 위해 대기하는 경우이다.    

```java
public synchronized void synchronizedSleep() {
	Utils.sleep();
}

Thread sleepThread = new Thread(this::synchronizedSleep, "mySleepThread");
Thread blockThread = new Thread(this::synchronizedSleep, "myBlockThread");

sleepThread.start();
Thread.sleep(100);
blockThread.start();
sleepThread.join();
blockThread.join();
```  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-10.png)  

```
"mySleepThread" #29 prio=5 os_prio=0 cpu=0.00ms elapsed=21.04s tid=0x00000265f5657800 nid=0x8840 waiting on condition  [0x000000ee84eff000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(java.base@11/Native Method)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:10)
        at com.windowforsun.javathread.threaddump.Utils.sleep(Utils.java:5)
        at com.windowforsun.javathread.threaddump.ThreadStateTest.synchronizedSleep(ThreadStateTest.java:59)
        - locked [Lock Key]<0x0000000440c00ba8> (a com.windowforsun.javathread.threaddump.ThreadStateTest)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$415/0x000000080022b840.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)

"myBlockThread" #30 prio=5 os_prio=0 cpu=0.00ms elapsed=20.93s tid=0x00000265f5657000 nid=0x8ed4 [Thread State]waiting for monitor entry  [0x000000ee84ffe000]
   java.lang.Thread.State: [Enum Thread State]BLOCKED (on object monitor)
        at com.windowforsun.javathread.threaddump.ThreadStateTest.synchronizedSleep(ThreadStateTest.java:59)
        - waiting to lock [Lock Key]<0x0000000440c00ba8> (a com.windowforsun.javathread.threaddump.ThreadStateTest)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$416/0x000000080022bc40.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)
```  

`mySleepThread` 가 먼저 `synchronizedSleep` 메소드를 호출해서 `<0x0000000440c00ba8>` 락을 소유 했다. 
그리고 100ms 이후 `blockThread` 가 `synchronizedSleep` 메소드 호출을 위해 `<0x0000000440c00ba8>` 락을 획득하기 위해 대기하고 있는 것을 확인 할 수 있다. 
그러므로 `Thread State` 는 `waiting for monitor entry` 로 나타나고, 
`Enum Thread State` 는 `BLOCKED (on object monitor)` 로 나타난 것을 확인 할 수 있다.  



#### park
`LockSupport.park()` 메소드를 사용해서 `Thread` 가 `Waiting` 상태에 빠진 경우이다.  

```java

Thread parkThread = new Thread(LockSupport::park, "myParkThread");

parkThread.start();
parkThread.join();
```  

![그림 1]({{site.baseurl}}/img/java/concept-thread-state-dump-11.png)

```
myParkThread" #24 prio=5 os_prio=0 cpu=0.00ms elapsed=22.41s tid=0x0000026afd360000 nid=0x1f98 waiting on condition  [0x000000a0bcdfe000]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@11/Native Method)
        at java.util.concurrent.locks.LockSupport.park(java.base@11/LockSupport.java:323)
        at com.windowforsun.javathread.threaddump.ThreadStateTest$$Lambda$386/0x00000008001e1c40.run(Unknown Source)
        at java.lang.Thread.run(java.base@11/Thread.java:834)
```  








---
## Reference
[스레드 덤프 분석하기](https://d2.naver.com/helloworld/10963)  
[Difference Between Wait And Park Methods In Java Thread](https://www.w3spot.com/2020/07/difference-between-wait-and-park-java-thread.html)  
[Class LockSupport](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/LockSupport.html)  
[LockSupport.park()란](https://applefarm.tistory.com/124)  
[VisualVM - Thread States](https://stackoverflow.com/questions/27406200/visualvm-thread-states)  
[Does synchronized park a concurrent thread like Lock.lock() does?](https://stackoverflow.com/questions/17233599/does-synchronized-park-a-concurrent-thread-like-lock-lock-does)  
[Enum Thread.State](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.State.html)  
[[Java] JVM Thread Dump 분석하기](https://steady-coding.tistory.com/597)  


