--- 
layout: single
classes: wide
title: "[Java 실습] JMH, Java Benchmark Tool"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Java 에서 간단한 성능 비교 테스트를 수행할 수 있는 JMH에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - JMH
  - Benchmark
toc: true 
use_math: true
---  

## JMH
[JMH(Java Microbenchmark Harness)](https://github.com/openjdk/jmh)
는 `OpenJDK` 에서 만든 성능 측정용 라이브러리이다. 
개발을 진행할때 성능 문제를 해결해야 하는 경우나, 
몇갸지 케이스 중 효율적인 걸 골라야 할때 성능 측정은 필수 절차이다. 
대부분의 성능 테스트들은 특정 환경에 애플리케이션을 배포하고 나서 측정을 하는 방식을 사용한다. 
이러한 방식을 통해 좀더 디테일한 성능 지표들을 확인해 볼 수 있지만 절차에 있어서는 불편함이 있다. 

하지만 위와 달리 `JMH` 는 간단한 코드에 대한 비교도 쉽고 빠르게 비교를 하며 개발을 진행할 수 있다는 점에서 큰 장점이 될 수 있다. 
디테일한 성능 지표는 확인 해볼 순 없지만, 어떠한 경우가 더 효율적인지에 대한 판단의 지표로는 충분히 사용될 수 있다.  

### Gradle 기반 JMH 적용하기
공싱 레퍼런스에서는 `Maven` 에 대한 사용을 권장하지만, 
필자는 `Gradle` 환경의 프로젝트가 대부분이기 때문에 `Gradle` 에서 적용하는 방법을 소개하고자 한다. 

적용 방법은 크게 어려움은 없고 [jmh-gradle-plugin](https://github.com/melix/jmh-gradle-plugin)
에 소개된 방법으로 설정만 해주면 된다.  

#### build.gradle

```groovy
plugins {
	id 'java'
	// 플러그인 추가
	id 'me.champeau.gradle.jmh' version '0.5.3'
}
```  

`Gradle` 버전별 필요한 최소 버전은 아래와 같다. 

|Gradle|플러그인 최소 버전
|7.0|0.5.3
|5.5|0.5.0
|5.1|0.4.8
|4.9|0.4.7 
|4.8|0.4.5
|4.7|0.4.5
|4.6|0.4.5
|4.5|0.4.5
|4.4|0.4.5
|4.3|0.4.5
|4.2|0.4.4
|4.1|0.4.4

#### 디렉토리 구조
`build.gradle` 에서 `JMH` 플러그인만 추가해주면 사용준비는 모두 완료된 상태이다. 
하지만 `JMH` 의 사용은 기존 디렉토리 구조가 아닌 별도의 구조를 따른다. 
그 예시는 아래와 같다.  

```bash
src
├── jmh
│   ├── java
│   │   └── com
│   │       └── windowforsun
│   │           └── benchmark
│   │               └── jmh
│   │                   └── FirstTest.java
│   └── resources
└── main
    ├── java
    │   └── com
    │       └── windowforsun
    │           └── benchmark
    │               └── jmh
    │                   └── Main.java
    └── resources
```  

눈에 띄는 차이는 일반 적으로는 `src/main` 와 같은 구조만 사용하지만, 
`JMH` 를 위해서는 `src/jmh` 라는 디렉토리 하위에 패키지를 생성해줘야 한다는 점이다.  

#### 간단한 테스트 실행
위 디렉토리 트리구조에서 보였던 `FirstTest` 파일을 바탕으로 간단한 테스트를 진행해보려고 한다.  

```java
@State(Scope.Thread)
@Fork(value = 1)
public class FirstTest {

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void fastest() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(1);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void fast() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void slow() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(10);
    }
}
```  

위 테스트는 `fastest()`, `fast()`, `slow()` 메소드의 성능을 비교하는 테스트를 작성한 예시이다. 
이제 프로젝트 루트 경로에서 아레 명령을 실행하면 테스트를 자동으로 실행해 준다.  

```bash
$ ./gradle jmh

.. 생략 ..

# Warmup Iteration   1: 6.052 ms/op
# Warmup Iteration   2: 6.046 ms/op
# Warmup Iteration   3: 6.035 ms/op
# Warmup Iteration   4: 6.045 ms/op
# Warmup Iteration   5: 6.109 ms/op
Iteration   1: 6.133 ms/opING [56s]
Iteration   2: 6.178 ms/opING [1m 6s]
Iteration   3: 6.141 ms/opING [1m 16s]
Iteration   4: 6.101 ms/opING [1m 26s]
Iteration   5: 6.165 ms/opING [1m 36s]

.. 생략 ..

Benchmark          Mode  Cnt   Score   Error  Units
FirstTest.fast     avgt    5   6.144 ± 0.115  ms/op
FirstTest.fastest  avgt    5   1.322 ± 0.092  ms/op
FirstTest.slow     avgt    5  12.070 ± 0.266  ms/op
```  

위 처럼 테스트 결과를 출력해 준다. 
결과를 보면 알 수 있듯이 테스트 실행 횟수와 `Mode` 에 맞는 `Score` 로 결과를 알려주는데, 
현재 `Mode` 가 `AverageTime` 이기 때문에 `Uits`(`ms/op`) 과 함꼐 `Score` 값을 확인하면 성능의 정확한 결과를 확인 할 수 있다.  


다양한 테스트 샘플은 [여기](https://github.com/openjdk/jmh/tree/master/jmh-samples/src/main/java/org/openjdk/jmh/samples)
에서 확인 가능하다.  

#### @BenchmarkMode
`@BenchmarkMode` 를 통해 벤치마크 테스트를 수행할때 어떤 방법으로 결과를 측정할지 정할 수 있다. 
설정가능한 모드는 아래와 같다.  

Mode|설명
---|---
Throughput|초당 작업수 측정(기본값)
AverageTime|작업이 수행되는 평균 시간값 측정
SampleTime|최대, 최소 등 작업이 수행하는데 소요되는 시간 측정
SingleShotTime|단일 작업 소요 시간 측정(Cold Start 테스트에 적합)
All|모든 모드 수행

예제 코드는 아래와 같다.  

```java
public class BenchmarkModes {
    @Benchmark
    @BenchmarkMode(Mode.Throughput)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureThroughput() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureAvgTime() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode(Mode.SampleTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureSamples() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode(Mode.SingleShotTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureSingleShot() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode({Mode.Throughput, Mode.AverageTime})
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureMultiple() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }

    @Benchmark
    @BenchmarkMode(Mode.All)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void measureAll() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(5);
    }
}
```  

실제 테스트 수행 결과는 아래와 같다.  

```bash
$ ./gradlew jmh

Benchmark                                               Mode   Cnt   Score   Error   Units
BenchmarkModes.measureThroughput                       thrpt     2   0.163          ops/ms
BenchmarkModes.measureAvgTime                           avgt     2   6.117           ms/op
BenchmarkModes.measureSamples                         sample  3296   6.062 ± 0.030   ms/op
BenchmarkModes.measureSamples:measureSamples·p0.00    sample         5.022           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.50    sample         6.210           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.90    sample         6.357           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.95    sample         6.636           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.99    sample         7.792           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.999   sample        11.512           ms/op
BenchmarkModes.measureSamples:measureSamples·p0.9999  sample        13.124           ms/op
BenchmarkModes.measureSamples:measureSamples·p1.00    sample        13.124           ms/op
BenchmarkModes.measureSingleShot                          ss     2   6.263           ms/op
BenchmarkModes.measureMultiple                         thrpt     2   0.162          ops/ms
BenchmarkModes.measureMultiple                          avgt     2   6.096           ms/op
BenchmarkModes.measureAll                              thrpt     2   0.163          ops/ms
BenchmarkModes.measureAll                               avgt     2   6.095           ms/op
BenchmarkModes.measureAll                             sample  3291   6.072 ± 0.028   ms/op
BenchmarkModes.measureAll:measureAll·p0.00            sample         5.030           ms/op
BenchmarkModes.measureAll:measureAll·p0.50            sample         6.259           ms/op
BenchmarkModes.measureAll:measureAll·p0.90            sample         6.324           ms/op
BenchmarkModes.measureAll:measureAll·p0.95            sample         6.521           ms/op
BenchmarkModes.measureAll:measureAll·p0.99            sample         7.932           ms/op
BenchmarkModes.measureAll:measureAll·p0.999           sample         9.721           ms/op
BenchmarkModes.measureAll:measureAll·p0.9999          sample        10.142           ms/op
BenchmarkModes.measureAll:measureAll·p1.00            sample        10.142           ms/op
BenchmarkModes.measureAll                                 ss     2   6.261           ms/op
```  

#### @OutputTimeUint
벤치마크 테스트를 수행 결과를 출력할 시간 단위를 `TimeUnit` 을 통해 설정할 수 있다. 
그 종류는 아래와 같다.  

- `TimeUnit.NANOSECONDS`
- `TimeUnit.MICROSECONDS`
- `TimeUnit.MILLISECONDS`
- `TimeUnit.SECONDS`
- `TimeUnit.MINUTES`
- `TimeUnit.HOURS`
- `TimeUnit.DAYS`

예제 코드는 아래와 같다. 

```java
public class OutputTimeUnits {
    @Benchmark
    @OutputTimeUnit(TimeUnit.HOURS)
    public void hours() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }

    @Benchmark
    @OutputTimeUnit(TimeUnit.MINUTES)
    public void minutes() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }

    @Benchmark
    @OutputTimeUnit(TimeUnit.SECONDS)
    public void seconds() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }

    @Benchmark
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void millis() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }

    @Benchmark
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void micro() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }

    @Benchmark
    @OutputTimeUnit(TimeUnit.NANOSECONDS)
    public void nanoseconds() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(100);
    }
}
```  

```bash
Benchmark                     Mode  Cnt      Score   Error    Units
OutputTimeUnits.hours        thrpt       34599.875           ops/hr
OutputTimeUnits.micro        thrpt          ≈ 10⁻⁵           ops/us
OutputTimeUnits.millis       thrpt           0.010           ops/ms
OutputTimeUnits.minutes      thrpt         578.952          ops/min
OutputTimeUnits.nanoseconds  thrpt          ≈ 10⁻⁸           ops/ns
OutputTimeUnits.seconds      thrpt           9.647            ops/s
```  

#### @State
`JMH` 를 사용해서 테스트를 할때 수행하는 메소드의 `Arguement` 를 주입할 수 있는데, 이 `Arguement` 의 상태를 지정하는 것이 바로 `@State` 이다. 
`@State` 를 통해 테스트 횟수마다 초기화가 필요하거나, 테스트 전체에서 계속 값을 유지해야 하는 등의 `Scope` 를 지정할 수 있다. 

Scope|설명
---|---
Thread|각 Thread 마다 인스턴스 생성
Benchmark|Benchmark 테스트 모든 스레드에서 인스턴스 공유
Group|Thread 그룹마다 인스턴스 생성

여기서 `Thread` 란 테스트시 설정 값으로 넣어준 `Thread` 의 수를 의미한다.  

```java
@Threads(value = 5)
public class States {
    @State(Scope.Benchmark)
    public static class BenchmarkState {
        volatile  int x = 0;
    }

    @State(Scope.Thread)
    public static class ThreadState {
        volatile int x = 0;
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    public void measureUnShared(ThreadState state) throws InterruptedException {
        // 모든 벤치마크 스레드가 해당 메소드를 호출한다.
        // 하지만 각 스레드마다 ThreadState 의 값은 새롭게 초기화 된다.
        TimeUnit.SECONDS.sleep(1);
        state.x++;
        System.out.println("UnShard : " + state.x);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    public void measureShared(BenchmarkState state) throws InterruptedException {
        // 모든 벤치마크 스레드가 해당 메소드를 호출한다.
        // 모든 벤치마크 스레드가 BenchmarkState 의 값을 공유한다.
        TimeUnit.SECONDS.sleep(1);
        state.x++;
        System.out.println("Shard : " + state.x);
    }
}
```  

출력 값을 살펴보면 `measureUnShared` 는 5개 스레드 마다 다른 인스턴스를 갖기 때문에 출력값이 `measureShared` 보다 낮은 걸 확인 할 수 있다.  

```bash
Shard : 104
UnShard : 22

Benchmark               Mode  Cnt  Score   Error  Units
States.measureShared    avgt       1.005           s/op
States.measureUnShared  avgt       1.004           s/op
```  



#### @Setup, @TearDown
`@Setup` 과 `@TearDown` 은 메소드에서 사용가능한 어노테이션이다. 
`@Setup` 은 벤치마크가 실행되기 전 `Object` 의 설정을 하기 위해 사용하고, 
`@TearDown` 은 벤치마크가 종료된 후 `Object` 를 정리하기 위해 사용된다. 
두 가지모두 실제 벤치마크 결과에는 포함되지 않는다. 
그리고 두 가지에 필드로 설정할 수 있는 값들이 있는데 그 종류와 설명은 아래와 같다.  

Level|설명
---|---
Trial|벤치마크를 실행(fork)할 때마다 한번씩 호출된다.
Iteration|벤치마크 반복(Iteration)이 수행될 때마다 호출 된다. 
Invocation|벤치마크 테스트 메소드가 실행 될떄 호출 된다. 

```java
/**
 * jmh {
 * operationsPerInvocation = 1
 * iterations = 1
 * fork = 1
 * warmupIterations = 1
 * }
 */
@State(Scope.Benchmark)
public class SetUpTearDown {
    private List<String> list = new ArrayList<>();

    @Setup(Level.Trial)
    public void setupTrial() {
        this.list.add("Setup:Trial");
        System.out.println("setup Trial : " + Arrays.toString(this.list.toArray()));
    }

    @Setup(Level.Iteration)
    public void setupIteration() {
        this.list.add("Setup:Iteration");
        System.out.println("setup Iteration : " + Arrays.toString(this.list.toArray()));
    }

    @Setup(Level.Invocation)
    public void setupInvocation() {
        this.list.add("Setup:Invocation");
        System.out.println("setup Invocation : " + Arrays.toString(this.list.toArray()));
    }

    @TearDown(Level.Trial)
    public void tearDownTrial() {
        this.list.add("TearDown:Trial");
        System.out.println("tearDown Trial :" + Arrays.toString(this.list.toArray()));
    }

    @TearDown(Level.Iteration)
    public void tearDownIteration() {
        this.list.add("TearDown:Iteration");
        System.out.println("tearDown Iteration :" + Arrays.toString(this.list.toArray()));
    }

    @TearDown(Level.Invocation)
    public void tearDownInvocation() {
        this.list.add("TearDown:Invocation");
        System.out.println("tearDown Invocation :" + Arrays.toString(this.list.toArray()));
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    public List<String> test() throws InterruptedException {
        this.list.add("test");
        System.out.println("test : " + Arrays.toString(this.list.toArray()));
        TimeUnit.SECONDS.sleep(1);
        return this.list;
    }

}
```  

예시의 대략적인 실행 흐름은 아래와 같다. 

```
.. Fork start ..
Setup:Trial
.. Iteration start..
Setup:Iteration
.. Invocation start
Setup:Invocation
test
TearDown:Invocation
.. Invocation end ..
TearDown:Iteration
.. Iteration end ..
TearDown:Trial ..
```  


#### Loop Optimization
벤치마크 메소드를 작성하다보면 좀더 큰 부하를 준다던가 하는 등의 이유로 메소드내 반복문을 추가해서 여러번 반복 수행하도록 구현할 수 있다. 
하지만 `JVM` 은 `Loop` 문을 최적화 하는 부분이 있기 때문에 기대하는 결과와 다른 결과가 나올 수가 있다. 
이러한 이유로 벤치마크 메소드내에 `Loop` 문 작성을 피해야 한다.  


#### Dead Code 주의하기
벤치마크 메소드를 작성할떄 주의해야 할 점중 하나는 `Daed Code` 이다. 
아래 코드예시를 보자. 

```java
@Benchmark
public void test() {
	int a = 1;
    int b = 2;
    int sum = a + b;
}
```  

위 벤치마크 메소드는 `a + b` 에 대한 벤치마크 결과를 측정하지 위함이지만, 
`JVM` 은 계산 결과가 한번도 사용되지 않는 `sum = a + b` 에 대한 부분과 `a`, `b` 선언 코드를 최적화 관점에서 아예 삭제해 버릴 수 있다. 
이렇게 되면 실질적으로 측정하고 싶은 부분을 측정하지 못하는 문제가 발생하게 된다.  

위와 같은 `Dead Code` 이슈를 회피하는 방법으로는 어떤 것들이 있을까 ?
즉 `sum = a + b` 에 대한 성능 측정을 실질적으로 수행하고 싶다면 아래와 같은 2가지 방법이 있다. 

- 결과 값을 벤치마크 메소드에서 리턴하도록 해라.

```java
@Benchmark
public int test() {
	int a = 1;
    int b = 2;
    int sum = a + b;
    
    return sum;
}
```  

- 결과 값을 `JMH` 가 제공하는 `Blackhole` 을 통해 넘겨 줘라. 

```java
@Benchmark
public void test(Blackhole blackhole) {
	int a = 1;
    int b = 2;
    int sum = a + b;
    
    blackhole.consume(sum);
}
```  

#### Blackhole 이란
위에서 `Dead Code` 를 회피하는 방법 중 `Blackhole` 방법을 소개했다. 
`Blackhole` 이란 `consume()` 메소드의 인자로 소비할 값을 넘겨주면 `JVM` 에서는 계산된 값을 사용한 것으로 인식을 할 수 있어, 
`Dead Code` 와 같이 해당 코드가 최적화 과정에서 삭제되는 일을 방지할 수 있다. 
한 벤치마크 메소드내 여러 값이 있다면, 이 또한 필요한 횟수만큼 `blackhole.consum()` 을 통해 전달해주면 된다.  


#### Constant Folding
`Constant Folding` 은 `JVM` 의 최적화 중 하나이다. 
고정된 상수에 의한 계산은 항상 동일한 결과가 도출 될 것이다. 
이러한 이유로 `JVM` 에 감지되면 결과를 계산하는 것이 아니라 최적화 과정에서 계산된 결과를 결과 변수에 넣는 식으로 변경될 수 있다.  

아래와 같은 코드가 있다고 가정해보자. 

```java
@Benchmark
public int test() {
	int a = 1;
	int b = 2;
	int sum = a + b;

	return sum;
}
```  

`sum = a + b` 이지만, 항상 `a = 1` 이고 `b = 2` 이기 때문에 결과는 매번 `3` 이 나온다. 
그래서 `JVM` 은 아래와 같이 최적화를 수행할 수 있다.  

```java
@Benchmark
public int test() {
    int sum = 3;
    
    return sum;
}
```  

계산을 직접 수행해서 리턴하는 것이 아니라, 항상 `sum = 3` 이기 때문에 계산을 바로 하지 않는 것이다. 
또는 바로 결과 값을 리턴해 버리는 `return 3` 으로 최적화 될 수도 있고, `test()` 메소드가 리턴하는 값은 매번 `3` 임이 보장 되기 때문에 
메소드를 호출하지 않고 호출 부분에 `3` 을 바로 넣어 버릴 수도 있다.  

#### Constant Folding 회피
`Constant Folding` 을 회피하는 방법중 하나는 `JVM` 이 값을 예측하기 어렵도록 복잡한 코드를 작서앟는 것이다. 
그 방법 중 하나는 계산에 사용하는 `a`, `b` 변수를 로컬변수로 두는 것이 아니라, 오브젝트의 멤버 변수로 만들어 참조하는 방식을 사용해 감지가 어렵도록 만드는 방법이다.  

```java
@State(Scope.Thread)
public static class MyState {
    public int a = 1;
    public int b = 2;
}

@Benchmark
public int test(MyState state) {
    int sum = state.a + state.b;
    
    return sum;
}
```  

만약 `MyState` 의 값들을 바탕으로 여러 결과 값을 만들어낸다면 `Blackhole` 을 사용할 수 있다.  

```java
@Benchmark
public void test(MyState state, Blackhole blackhole) {
	int sum1 = state.a + state.b;
    int sum2 = state.a + state.a + state.b + state.b;
    
    blackhole.consume(sum1);
    blackhole.consume(sum2);
}
```  

---
## Reference
[openjdk/jmh](https://github.com/openjdk/jmh)  
[openjdk/jmh samples](https://github.com/openjdk/jmh/tree/master/jmh-samples/src/main/java/org/openjdk/jmh/samples)  
[melix/jmh-gradle-plugin](https://github.com/melix/jmh-gradle-plugin)  
[JMH - Java Microbenchmark Harness](https://jenkov.com/tutorials/java-performance/jmh.html)  


