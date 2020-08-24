--- 
layout: single
classes: wide
title: "[Spring 실습] HikariCP 및 설정 더 알아보기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'HikariCP 에 설정 가능한 옵션을 포함해서 관련 상황에 대해 테스트를 통해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - HikariCP
    - MySQL
    - Config
toc: true
use_math: true
---  

## HikariCP 더 알아보기
[여기]({{site.baseurl}}{% link _posts/spring/2020-08-19-spring-practice-hikari-basic.md %})
에서는 `HikariCP` 에 대한 기본 사용과 구성법에 대해 알아봤다. 
본 포스트에서는 `HikariCP` 에 대해 보다 세부적인 부분에 대해 알아 보고 테스트를 진행해 본다. 

## HikariCP Config
`HikariCP` 는 여러 옵션 값을 제공하고, 이를 사용해서 애플리케이션과 환경에 적합한 설정을 할 수 있다. 
기본적으로 시간관련 값의 경우 `milliseconds` 를 사용한다. 
설정 값에 대한 필드와 관련 설명은 [여기](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
에서 확인할 수 있다. 
그리고 `JDBC` 드라이버에서 지원하는 드라이버에 대한 설정 이름은 [여기](https://github.com/brettwooldridge/HikariCP#popular-datasource-class-names)
에서 확인할 수 있다. 

## Connection 획득과 반납
간단한 테스트코드를 바탕으로 `HikariCP` 에서 실제로 커넥션을 관리하는 흐름과 방식에 대해 알아본다. 

```java
@SpringBootTest
public class SimpleTest {
    @Autowired
    private HikariDataSource hikariDataSource;

    @Test
    public void connectionTest() throws Exception {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet rs = null;

        try {
            // 커넥션 획득
            connection = this.hikariDataSource.getConnection();
            preparedStatement = connection.prepareStatement("show variables like 'server_id'");
            rs = preparedStatement.executeQuery();
            rs.next();
            int result = rs.getInt(2);
        } catch(Throwable t) {
            throw new RuntimeException(t);
        } finally {
            if(preparedStatement != null) {
                preparedStatement.close();
            }

            // 커넥션 반납
            if(connection != null) {
                connection.close();
            }
        }
    }
}
```  

위와 같이 사용자 입장에서는 `dataSource.getConnection()` 으로 커넥션을 가져오고, `connection.close()` 로 커넥션을 반납하면 된다. 
이 두 과정 사이에는 `HikariCP` 가 커넥션을 관리하는 과정이 있을 것이다. 이부분에 대해 살펴보도록 한다.  

[HikariDataSource](https://github.com/brettwooldridge/HikariCP/blob/dev/src/main/java/com/zaxxer/hikari/HikariDataSource.java)
에서는 [HikariPool](https://github.com/brettwooldridge/HikariCP/blob/dev/src/main/java/com/zaxxer/hikari/pool/HikariPool.java)
객체를 사용해서 풀을 구성한다. 
그리고 `HikariPool` 에서는 커넥션을 다시 [ConnectionBag](https://github.com/brettwooldridge/HikariCP/blob/dev/src/main/java/com/zaxxer/hikari/util/ConcurrentBag.java)
이라는 자료구조를 사용하고, 
`ConnectionBag` 은 [PoolEntry](https://github.com/brettwooldridge/HikariCP/blob/dev/src/main/java/com/zaxxer/hikari/pool/PoolEntry.java)
라는 실제 `Connection` 객체를 한번 랩핑한 객체를 관리한다.  

`HikariPool` 가 [ConnectionBag.borrow()](https://github.com/brettwooldridge/HikariCP/blob/c993ef099282c3fd3b830f6cf9950c8cfe2bd8fb/src/main/java/com/zaxxer/hikari/util/ConcurrentBag.java#L120)
메소드를 호출하면 `Idle` 상태에 있는 커넥션을 반환하게 된다. 
이때 `ConnecitonBag` 은 실제 커넥션과 실행중인 스레드를 바탕으로 커넥션을 관리한다. 
실제로 `ConnectionBag.borrow()` 메소드를 확인하면, 한번 커넥션을 획득한 이력이 있는 스레드에 대한 커넥션 정보를 `threadList` 에서 조회한다.  
그리고 이후에 다시 해당 스레드가 커넥션을 요청하면 이전에 사용이력이 있는 커넥션 반환에 대한 동작을 한번 수행하게 된다.  

`HikariPool` 에서 반환되는 커넥션은 `Connection` 을 구현한 [ProxyConnection](https://github.com/brettwooldridge/HikariCP/blob/dev/src/main/java/com/zaxxer/hikari/pool/ProxyConnection.java)
타입의 객체이다.  

그리고 스레드에서 커넥션 사용을 마치고 `Connection.close()` 호출로 커넥션을 반납할 수 있다. 
해당 메소드를 호출하면 [ConnectionBag.requite()](https://github.com/brettwooldridge/HikariCP/blob/c993ef099282c3fd3b830f6cf9950c8cfe2bd8fb/src/main/java/com/zaxxer/hikari/util/ConcurrentBag.java#L175)
메소드가 호출된다. 
메소드에서는 커넥션의 상태값을 변경하고한다. 
그리고 커넥션을 대기중인 쓰레드가 있다면 `handoffQueue` 에 해당 커넥션을 추가해서 다른 스레드가 사용할 수 있도록 한다. 
마지막으로 반납한 커넥션을 `threadList` 에 추가한다.  

만약 스레드에 커넥션 사용이력이 없거나, 사용했었던 커넥션이 다른 스레드에서 사용 중이라면 전체 커넥션에 대해 사용 가능(`Idle`) 커넥션을 찾게 된다.  
전체 커넥션 풀에서 사용가능한 커넥션이 있는지 검사하고 있다면 해당 커넥션을 반환한다. 
위 상태에서 모든 커넥션이 사용 중이라면, `handoffQueue` 가 사용 가능한 커넥션을 반환 할때 까지 대기한다. 
대기 시간이 `connectionTimeout` 옵션에 설정된 시간을 넘어가게 되면 `Exceptione` 이 발생한다.   

`HikariDataSource` 에서 커넥션을 획득하기(`getConnection()`) 까지 과정을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/practice_hikari_adv_getconnection.png)

`HikariDataSource` 에서 커넥션을 반납하기(`close()`) 까지 과정을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/spring/practice_hikari_adv_close.png)

## Connection Pool 관리
`HikariCP` 를 사용해서 데이터베이스와 커넥션을 관리하고, 
모든 요청처리에 데이터베이스 커넥션을 사용하는 애플리케이션을 가정해 보겠다. 
최대 커넥션 풀 크기를 설정하는 `maximumPoolSize` 의 설정값이 2인 상태에서 동시에 10개의 요청이 왔다고 가정한다. 
먼저 커넥션을 획득한 2개의 요청은 이후 처리도 정상적으로 진행 할 수 있다. 
하지만 나머지 8개의 요청은 커넥션을 획득한 2개의 요청에서 커넥션을 반환할때 까지 대기하게 된다. 
그리고 `connectionTimeout` 에 설정된 시간동안 커넥션을 획득하지 못한 요청은 예외(`SQLTransientConnectionException`)를 발생시킨다.  

아래 테스트 코드로 위 상황을 재연해보았다. 

```java
public class ConnectionTester {
    private DataSource dataSource;
    private int threadCount;
    private int connectionCount;
    private int loopCount;
    private long sleepMillis;
    private Class exceptionClass;

    public ConnectionTester(DataSource dataSource, int threadCount, int connectionCount, int loopCount, long sleepMillis) {
        this.dataSource = dataSource;
        this.threadCount = threadCount;
        this.connectionCount = connectionCount;
        this.loopCount = loopCount;
        this.sleepMillis = sleepMillis;
    }

    public ConnectionTester(DataSource dataSource, int threadCount, int connectionCount, int loopCount, long sleepMillis, Class exceptionClass) {
        this(dataSource, threadCount, connectionCount, loopCount, sleepMillis);
        this.exceptionClass = exceptionClass;
    }

    public ConcurrentHashMap<Long, Integer> execute() {
        // 각 Thread 가 실제로 loopCount 를 어디까지 수행했는지 관리
        ConcurrentHashMap<Long, Integer> countMap = new ConcurrentHashMap<>();

        try {
            Thread[] threads = new Thread[this.threadCount];
            Runnable[] runnables = new Runnable[this.threadCount];

            // Thread 에서 지정된(exceptionClass) 예외가 발생하면 모든 스레드 종료
            Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
                @Override
                public void uncaughtException(Thread t, Throwable e) {
                    for (Thread thread : threads) {
                        if (!thread.isInterrupted()) {
                            thread.interrupt();
                        }
                    }
                }
            });

            // Thread 실행
            for (int i = 0; i < this.threadCount; i++) {
                runnables[i] = new Runnable() {
                    @Override
                    public void run() {
                        long id = Thread.currentThread().getId();
                        Connection[] cons = null;

                        try {
                            countMap.put(id, 0);

                            // 각 Thread 는 loopCount 만큼 커넥션 획득 -> 슬립 -> 반환을 반복
                            for (int k = 0; k < loopCount && !Thread.currentThread().isInterrupted(); k++) {
                                cons = new Connection[connectionCount];

                                // 커넥션 획득
                                for (int q = 0; q < connectionCount; q++) {
                                    cons[q] = dataSource.getConnection();
                                }

                                // 커넥션 사용
                                for (int q = 0; q < connectionCount; q++) {
                                    PreparedStatement psmt = cons[q].prepareStatement("show variables like 'server_id'");
                                    ResultSet rs = psmt.executeQuery();
                                    rs.next();

                                    assertThat(rs.getInt(2), is(1));
                                    psmt.close();
                                }

                                // 현재 스레드 loopCount 수 갱신
                                countMap.put(id, k + 1);

                                // 슬립
                                Thread.sleep(sleepMillis);

                                // 커넥션 반납
                                this.closeConnections(cons);
                            }

                        } catch (Exception e) {
                            System.out.println(e.getMessage());
                            // 예외가 발생한 스레드 커넥션 반납
                            this.closeConnections(cons);

                            // Thread.setDefaultUncaughtExceptionHandler 에서 캐치 가능하도록 지정된 예외는 다시 예외 발생
                            if (e.getClass() == exceptionClass) {
                                throw new RuntimeException(e);
                            }
                        }

                        // 인터럽트된 스레드 커넥션 반납
                        this.closeConnections(cons);
                    }

                    public void closeConnections(Connection[] cons) {
                        if (cons != null) {
                            int size = cons.length;

                            try {
                                for (int i = 0; i < size; i++) {
                                    if (cons[i] != null && !cons[i].isClosed()) {
                                        cons[i].close();
                                    }
                                }
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                        }
                    }
                };
            }

            for (int i = 0; i < this.threadCount; i++) {
                threads[i] = new Thread(runnables[i]);
                threads[i].start();
            }

            for (int i = 0; i < this.threadCount; i++) {
                threads[i].join();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        return countMap;
    }
}
```  

`threadCount` 수 만큼 스레드를 생성해서 테스트를 수행하는 클래스이다. 
각 스레드는 아래와 같은 동작을 `loopCount` 만큼 반복한다. 
1. 커넥션 획득
1. 쿼리 수행
1. 슬립
1. 커넥션 반납

그리고 각 스레드가 실제로 모든 `loopCount` 를 수행했는지 검증하기 위해 `countMap` 변수를 사용해서 관리한다. 
그리고 스레드에서 `exceptionClass` 와 같은 예외가 발생하면 모든 스레드는 중지되고 그 시점까지의 `countMap` 을 반환한다.

아래는 `ConnectionTester` 클래스를 사용해서 테스트를 수행하는 테스트 코드이다. 

```java
@SpringBootTest
@TestPropertySource(properties = {
        "spring.datasource.hikari.maximumPoolSize = 2",
        "spring.datasource.hikari.connectionTimeout = 10000"
})
public class ThreadSingleConnectionTest {
    @Autowired
    private DataSource dataSource;

    @Test
    public void thread_2_success() {
        // given
        int threadCount = 2;
        int connectionCount = 1;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(is(loopCount)));
    }

    @Test
    public void thread_10_timeout() {
        // given
        int threadCount = 10;
        int connectionCount = 1;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(lessThan(loopCount)));
    }
}
```  

`@TestPropertySource` 를 사용해서 테스트시 필요한 `DataSource` 옵션 값을 설정한다. 
- `maximumPoolSize` : 2
- `connectionTimeout` : 10

최대 커넥션의 수가 2개이기 때문에 스레드 2개가 계속해서 커넥션을 획득하고 반납하는 동작에서는 `SQLTransientConnectionException` 예외가 발생하지 않을 것이다. 
하지만 2개 보다 큰 스레드가 위 동작을 반복한다면 바로 발생하진 않겠지만, 
스레드의 수와 커넥션 획득과 반납까지 소요 시간에 따라 예외가 발생하게 될것이다.  

실제로 테스트를 수행하면 `thread_2_success()` 테스트는 `loopCount` 를 늘려도 관련 에러가 발생하지 않는다. 
하지만 `thread_10_timeout()` 테스는 설정된 타임아웃 시간인 10초 후에 에러가 발생하고 스레드가 모두 종료 된다.  

아주 당연하면서도 중요한 부분이라고 생각한다. 
예외를 예방하려고 풀 사이즈를 크게 잡으면 낭비되는 자원이 생기고, 
낭비되는 자원을 줄이기 위해서 타임 아웃을 길게 설정하면 요청 처리에 지원이 발생할 수 있다. 
그러므로 애플리케이션의 스펙과, 서비스 및 인프라 상황을 잘 예측해서 낭비와 결핍없는 설정이 필요하다.  

그렇다면 어떠한 조건과 방법으로 풀 사이즈를 설정해야할까 ? 
이부분은 [About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
을 참고해서 어느정도 해결할 수 있다. 
위 링크에서 제시한 풀 사이즈를 도출할 수 있는 공식은 아래와 같다. 

$$
pool size = T_n \times (C_m - 1) + 1
$$  

여기서 $T_n$ 은 커넥션을 사용하는 최대 스레드의 수이고, $C_m$ 은 하나의 스레드에서 필요한 최대 커넥션 수를 의미한다.  

$T_n$ 이 3이고, $C_m$ 이 2일때 계산하면 필요한 풀 사이즈는 4가 된다. 
아래는 제시한 상황을 테스트하는 코드이다. 

```java
@SpringBootTest
@TestPropertySource(properties = {
        "spring.datasource.hikari.maximumPoolSize = 4",
        "spring.datasource.hikari.connectionTimeout = 10000"
})
public class Thread3Connection2Test {
    @Autowired
    private DataSource dataSource;

    @Test
    public void thread_3_connection_2_success() {
        // given
        int threadCount = 3;
        int connectionCount = 2;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(is(loopCount)));
    }

    @Test
    public void thread_3_connection_3_timeout() {
        // given
        int threadCount = 3;
        int connectionCount = 3;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(lessThan(loopCount)));
    }

    @Test
    public void thread_4_connection_3_timeout() {
        // given
        int threadCount = 4;
        int connectionCount = 3;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(lessThan(loopCount)));
    }
}
```  

풀 사이즈를 4로 설정했을 때 $T_n = 3, C_m = 2$ 일때는 정상적으로 모든 수행이 성공하지만, 
$T_n = 3, C_m = 3$ 일때는 `SQLTransientConnectionException` 이 발생하는 것을 확인 할 수 있다.  

이번에는 $T_n$ 이 5이고, $C_m$ 이 3일때 계산하면 필요한 풀 사이즈는 11이 된다. 
아래는 제시한 상황을 테스트하는 코드이다.   

```java
@SpringBootTest
@TestPropertySource(properties = {
        "spring.datasource.hikari.maximumPoolSize = 11",
        "spring.datasource.hikari.connectionTimeout = 10000"
})
public class Thread5Connection3Test {
    @Autowired
    private DataSource dataSource;

    @Test
    public void thread_5_connection_3_success() {
        // given
        int threadCount = 5;
        int connectionCount = 3;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(is(loopCount)));
    }

    @Test
    public void thread_5_connection_4_timeout() {
        // given
        int threadCount = 5;
        int connectionCount = 4;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(lessThan(loopCount)));
    }

    @Test
    public void thread_6_connection_4_timeout() {
        // given
        int threadCount = 6;
        int connectionCount = 4;
        int loopCount = 50;
        long sleepMillis = 2000;
        ConnectionTester tester = new ConnectionTester(
                this.dataSource,
                threadCount,
                connectionCount,
                loopCount,
                sleepMillis,
                SQLTransientConnectionException.class
        );

        // when
        ConcurrentHashMap<Long, Integer> actual = tester.execute();

        // then
        assertThat(actual.size(), is(threadCount));
        assertThat(actual.values(), everyItem(lessThan(loopCount)));
    }
}
```  

테스트 코드를 바탕으로 어느정도 풀 사이즈를 예측에 대한 검증을 수행했지만, 
이는 `Deaed lock` 을 회피할 수 있는 최소한의 풀 사이즈이다. 
실 환경에서는 이보다 더 다양한 변수가 있으므로, 그에 따른 상황을 고려해야 할것이다.    

## Connection 생존 시간
`HikariCP` 는 다른 `Tomcat DBCP` 와 달리 `test-while-idle` 과 같은 커넥션 갱신 기능은 제공하지 않는다. 
대신 `maxLifetime` 과 같은 옵션 값을 조정해서 커넥션 유지에 대한 관리를 할 수 있다.  
그리고 커넥션을 계속해서 유지하는 방식이 아닌 옵션에 설정된 생존 시간만큼 커넥션을 유지하는 방식을 사용한다.  

`HikariCP 2.3.2+` 부터 [Dropwizard HealthChecks](https://github.com/brettwooldridge/HikariCP/wiki/Dropwizard-HealthChecks
기능을 사용해서 커넥션 갱신과 같은 동작을 수행할 수는 있다.  

`HikariCP` 에서 `maxLifetime` 을 사용해서 커넥션을 관리하는 것은 `HikariPool` 클래스에서 `houseKeeper` 네이밍이 있는 변수들이다. 
실제로 커넥션 랩퍼인 `PoolEntry` 를 생성하는 [HikariPool.createPoolEntry()](https://github.com/brettwooldridge/HikariCP/blob/c993ef099282c3fd3b830f6cf9950c8cfe2bd8fb/src/main/java/com/zaxxer/hikari/pool/HikariPool.java#L474)
를 보면 `maxLifetime` 을 받아 `ScheduledFuture` 에 설정하는 것을 확인할 수 있다. 
`maxLifetime` 연산에 `variance` 라는 변수값을 사용해서 커넥션 만료로 동시에 커넥션을 끊는 동작을 방지 한다.  

그리고 스케쥴러에 등록되는 메소드인 [softEvictConnection()](https://github.com/brettwooldridge/HikariCP/blob/c993ef099282c3fd3b830f6cf9950c8cfe2bd8fb/src/main/java/com/zaxxer/hikari/pool/HikariPool.java#L611)
을 확인하면 만료되면 커넥션에 `eviection` 을 설정하고, 상태가 `STATE_NOT_IN_USE` 라면 커넥션 종료 사용 중이라면 커넥션을 유지한다. 
유지된 커넥션은 사용 후, `HikariPool.getConnection()` 메소드에서 [PoolEntry.isMarkedEviected()](https://github.com/brettwooldridge/HikariCP/blob/c993ef099282c3fd3b830f6cf9950c8cfe2bd8fb/src/main/java/com/zaxxer/hikari/pool/HikariPool.java#L186)
메소드를 통해 걸러질 수 있다.  

아래는 위 상황을 코드로 풀어낸 테스트 코드이다. 

```java
@SpringBootTest
@TestPropertySource(properties = {
        "spring.datasource.hikari.maxLifetime = 30000",
        "spring.datasource.hikari.connectionTimeout = 10000"
})
public class MaxLifetimeTest {
    @Autowired
    private HikariDataSource dataSource;

    @Test
    public void get_twice_connection_same() throws Exception {
        // given
        Connection con = this.dataSource.getConnection();
        long connectionId = con.unwrap(ConnectionImpl.class).getId();
        con.close();

        // when
        con = this.dataSource.getConnection();
        long actual = con.unwrap(ConnectionImpl.class).getId();
        con.close();

        // then
        assertThat(actual, is(connectionId));
    }

    @Test
    public void get_twice_connection_sleep_notsame() throws Exception {
        // given
        Connection con = this.dataSource.getConnection();
        long connectionId = con.unwrap(ConnectionImpl.class).getId();
        con.close();

        // when
        Thread.sleep(30100);
        con = this.dataSource.getConnection();
        long actual = con.unwrap(ConnectionImpl.class).getId();
        con.close();

        // then
        assertThat(actual, not(connectionId));
    }

    @Test
    public void connection_use_and_sleep_notevict() throws Exception {
        // given
        Connection con = this.dataSource.getConnection();
        long waitingMillis = 30100;
        long start = System.currentTimeMillis();
        while(System.currentTimeMillis() - start >= waitingMillis) {
            con.prepareStatement("show variables like 'server_id'").executeQuery();
            Thread.sleep(100);
        }

        // when
        PreparedStatement psmt = con.prepareStatement("show variables like 'server_id'");
        ResultSet rs = psmt.executeQuery();
        rs.next();
        long actual = rs.getInt(2);
        psmt.close();
        con.close();

        // then
        assertThat(actual, is(1l));
    }

    @Test
    public void connection_sleep_evict() throws Exception {
        // given
        Connection con = this.dataSource.getConnection();
        Thread.sleep(30100);

        // then
        assertThrows(CommunicationsException.class, () -> {
            PreparedStatement psmt = con.prepareStatement("show variables like 'server_id'");
            ResultSet rs = psmt.executeQuery();
            rs.next();
            long actual = rs.getInt(2);
            psmt.close();
            con.close();
        });
    }
}
```  

`maxLifetime` 옵션의 최소값은 30000(30초) 이다. 
빠른 테스트 진행을 위해 30초로 설정하고 진행했다. 
같은 스레드에서 연속해서 2번 커넥션을 가져오고, 
`Connection.unwrap(ConnectionImpl.class).getId()` 을 통해 커넥션의 고유 아이디를 가져와 판별하면 같은 아이디인 것을 확인할 수 있다. 
하지만 커넥션을 가져오는 사이에 `maxLifetime` 만큼 슬립을 걸고 수행하면 다른 아이디인것을 확인 가능하다.   

그리고 하나의 커넥션을 `maxLifetime` 이 넘는 시간동안 계속 사용하는 것은 가능하지만, 
커넥션을 가져온 후 사용하지 않은 상태로 `maxLifetime` 이 지나면 `CommunicationsException` 예외가 발생하는 것을 확인할 수 있다.  







---
## Reference
[HikariCP Configuration (knobs, baby!)](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)  
[HikariCP Dead lock에서 벗어나기 (이론편)](https://woowabros.github.io/experience/2020/02/06/hikaricp-avoid-dead-lock.html)  
[HikariCP Dead lock에서 벗어나기 (실전편)](https://woowabros.github.io/experience/2020/02/06/hikaricp-avoid-dead-lock-2.html)  

