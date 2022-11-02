--- 
layout: single
classes: wide
title: "[Java 실습] Java Benchmark Tool JMH"
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











---
## Reference
[openjdk/jmh](https://github.com/openjdk/jmh)  
[openjdk/jmh samples](https://github.com/openjdk/jmh/tree/master/jmh-samples/src/main/java/org/openjdk/jmh/samples)  
[melix/jmh-gradle-plugin](https://github.com/melix/jmh-gradle-plugin)  
[JMH - Java Microbenchmark Harness](https://jenkov.com/tutorials/java-performance/jmh.html)  


