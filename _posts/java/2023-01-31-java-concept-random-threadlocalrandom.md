--- 
layout: single
classes: wide
title: "[Java 실습] Java 난수(Random) 값 구하기 정리 및 주의사항"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Java 에서 난수를 구하는 Random, ThreadLocalRandom, SplittableRandom, SecureRandom 과 주의하상에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Benchmark
  - Random
  - ThreadLocalRandom
  - SplittableRandom
  - SecureRandom
toc: true 
use_math: true
---  

## Random 클래스
`Java` 언어에서 난수가 필요하다면 `Random` 클래스를 사용해서 아래와 같이 구현할 것이다.  

```java
Random random = new Random();

int randomValue = random.nextInt();
```  

기본 생성자로 인스턴스를 생성하면 `seed` 값은 `nanoTime()` 을 바탕으로 설정된다. 

```java
public Random() {
    this(seedUniquifier() ^ System.nanoTime());
}

private static long seedUniquifier() {
    // L'Ecuyer, "Tables of Linear Congruential Generators of
    // Different Sizes and Good Lattice Structure", 1999
    for (;;) {
        long current = seedUniquifier.get();
        long next = current * 1181783497276652981L;
        if (seedUniquifier.compareAndSet(current, next))
            return next;
    }
}
```  

`randomValue` 에는 `nextInt()` 메소드로 부터 반환되는 `int` 타입 범위내의 난수가 설정되는데, 
`Random` 클래스는 난수 생성을 `seed` 를 바탕으로 난수르 결정한다.  

기본적으로 컴퓨터에서 수학적 측면에서 완전한 난수를 생성하기에는 어려움이 많다. 
컴퓨터는 동일 입력에 대해 동일 출력이 보장되야하는 결정적 유한 오토마타(`Deterministic Finite Automata`) 이기 때문이다. 
결과적으로 `Random` 클래스 생성자에서 설정하는 `seed` 값이 동일할 경우 동일한 난수가 생성됨을 알 수 있다.  

```java
Random randomSeed1000_1 = new Random(1000);
Random randomSeed1000_2 = new Random(1000);

assertThat(randomSeed1000_1.nextInt(), is(randomSeed1000_2.nextInt()));
assertThat(randomSeed1000_1.nextInt(), is(randomSeed1000_2.nextInt()));
assertThat(randomSeed1000_1.nextInt(), is(randomSeed1000_2.nextInt()));
```  

지금까지는 `int` 전체 범위에 대한 난수 생성만 수행했지만, 특정 범위 내의 난수 생성도 가능하다. 
여기에서 주의해야할 점이 있는데, `Random` 클래스를 사용해서 특정 범위내 난수를 생성할 때 아래와 같은 구현은 위험 할 수 있다.  

### 잘못된 난수 생성

```java
Random random = new Random();

// 0 ~ 99 까지 난수 생성
int boundedRandomValue = random.nextInt() % 100;
```  

대부분의 경우에는 위 코드는 정상동작 하는 것처럼 보일 수 있지만, 
이는 `nextInt()` 메소드가 음수를 반환하게 되면 문제가 발생한다. 
실제로 아래와 같이 `seed` 값을 `18` 이나 `34` 를 넣게 되면 `nextInt()` 가 음수를 반환한다. 
그러므로 음수 처리를 위해 `Math.abs()` 메소드로 음수를 양수로 전환이 필요하다. 

```java
Random random = new Random(18);

// -1148559096
int randomValue = random.nextInt();
// 87
int bounedRandomValue = Math.abs(random.nextInt()) % 100;
```  

하지만 이러한 방식 자체가 모든 케이스에서 완벽한것은 아니다. 
만약 `nextInt()` 가 `Integer.MIN_VALUE` 를 반환하게 되면, 
`Math.abs(Integer.MIN_VALUE)` 도 음수를 반환하기 때문에 안전하지 않는 방법이다.  

결과적으로 `nextInt()` 는 `Integer.MIN_VALUE ~ Integer.MAX_VALUE` 사이의 난수 값이 필요한 경우에만 사용해야 한다. 
그리고 특정 범위내 난수가 필요한 경우 `nextInt(bound)` 메소드로 안전하게 난수를 구할 수 있다.  

```java
Random random = new Random(18);

// 0 (실제 난수 값이 0이다.)
int boundedRandomValue = random.nextInt(100);
```  

### 멀티 스레딩에서 문제 
먼저 `Random` 클래스는 멀티 스레드에서 하나의 인스턴스를 공유하여 전역적으로 동작한다. 
멀티 스레딩 동작에서 동일한 `nanoTime` 에 난수 생성 요청이 들어오더라도 다행히 동일한 난수는 반환하지 않는다. 
`Random` 클래스의 구현부를 확인하면 아래와 같다. [next()](https://github.com/openjdk/jdk/blob/a8a2246158bc53414394b007cbf47413e62d942e/src/java.base/share/classes/java/util/Random.java#L198)

```java
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```  

동일한 난수는 반환하지 않지만, 이로인해 성능적인 결함이 발생하는 부분이 있다. 
코드를 살펴보면 `Random` 의 의사 난수 생성은 선형 합동 생성기([Linear Congruential Generator](https://en.wikipedia.org/wiki/Linear_congruential_generator)) 
알고리즘을 바탕으로 동작하는데, 하나의 스레드가 동시 경합에서 이기면 다른 스레드는 다신이 이길때까지 반복된 동작으로 스레드간 경합이 발생하게 된다. 
이는 일반적인 `Java` 비동기 알고리즘에서 사용되는 방법이지만(`CAS(Compare And Set)`), 
여러 스레드가 반복적으로 난수를 생성하는 상황이라면 간단한 난수 생성 동작에서 심각한 지연이 발생할 수 있다.  

> - 의사난수 : 난수처럼 보이게 하기 위해 알고리즘을 바탕으로 규칙적인 난수를 생성

그러므로 멀티 스레딩 환경에서는 `Random` 보다는 다른 방식의 난수 생성이 필요하다.  


## ThreadLocalRandom
`Random` 클래스에서 멀티 스레드 경합 이슈를 간단하게 해결할 수 있는 방법이 바로 `ThreadLocalRandom` 이다. 
`ThreadLocalRandom` 은 `thread` 별로 격리된 `random number generator` 를 사용하기 때문에, 
`Random` 보다 더 고품질의 난수를 생성 할 수 있고 성능도 더 우수하다.  

```java
ThreadLocalRandom random = ThreadLocalRandom.current();

int randomValue = random.nextInt();
// 0 ~ 99
int boundedRandomValue = random.nextInt(100);
```  

인스턴스 생성 부분을 보면 아래와 같이 현재 스레드 정보를 기반으로 생성한다. ([current()](https://github.com/openjdk/jdk/blob/a8a2246158bc53414394b007cbf47413e62d942e/src/java.base/share/classes/java/util/concurrent/ThreadLocalRandom.java#L176))

```java
public static ThreadLocalRandom current() {
    if (U.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}

static final void localInit() {
  int p = probeGenerator.addAndGet(PROBE_INCREMENT);
  int probe = (p == 0) ? 1 : p; // skip 0
  long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
  Thread t = Thread.currentThread();
  U.putLong(t, SEED, seed);
  U.putInt(t, PROBE, probe);
}
```  

`Random` 도 앞서 언급한 것처럼 `thread-safe` 하고 `ThreadLocalRandom` 도 `thread-safe` 하다. 
세부적인 부분에서는 차이가 있는데, 
`Random` 은 `seed` 를 `AtomicLong` 을 사용하기 때문에, 멀티 스레드 요청에 대해서도 순서대로 처리해서 동기화 처리 처럼 수행되는 부분으로 성능저하를 언급했었다. 
`ThreadLocalRandom` 은 `AutomicLong` 을 사용하지 않고 스레드마다 `seed` 관리로 멀티 스레드 환경에서 더 적합하다.  

추가적으로 `ThreadLocalRandom` 에 `seed` 를 별도로 설정은 불가능한 것으로 확인된다. [관련 내용](https://stackoverflow.com/questions/15765399/should-i-prefer-threadlocalrandom-over-threadlocalrandom  



## SplittableRandom 
`SplittableRandom` 은 격리된 병렬처리에 특화된 `random number generator` 이다. 
병렬처리 환경에서 난수 품질이 더 중요한 경우 `ThreadLocalRandom` 보다 `SplittableRandom` 이 더 좋은 선택이 될 수 있다.  

```java
SplittableRandom random = new SplittableRandom();

int randomValue = random.nextInt();
// 0 ~ 99
int boundedRandomValue = random.nextInt(100);
```  

`SplittableRandom` 은 `thread-safe` 하지 않기 때문에, 잘못 사용하는 경우 문제가 발생 할 수 있다. 
`fork/join` 처리 처럼 병렬처리에서는 [split()](https://github.com/openjdk/jdk/blob/a8a2246158bc53414394b007cbf47413e62d942e/src/java.base/share/classes/java/util/SplittableRandom.java#L397)
메소드를 통해 새로운 `SplittableRandom` 인스턴스를 생성해 줘야한다.  

```java
public SplittableRandom split() {
    return new SplittableRandom(nextLong(), mixGamma(nextSeed()));
}
```  

만약 `Random` 클래스에서 사용하는 것과 동일하게 하나의 `SplittableRandom` 인스턴스를 
여러 스레드에서 공유해서 사용하게 되면 난수 품질이 떨어질 수 있다. 
그리고 `split()` 으로 인스턴스를 생성하는 것이 아니라 생성사를 사용해서 각 스레드에서 `SplittableRandom` 을 생성해서 
사용하더라도 `seed` 가 동일할 수 있기 때문에 매번 같은 난수가 생성되는 등의 품질이 안좋아 질 수 있다.  

`split()` 메서드는 새로운 `SplittableRandom` 인스턴스를 생성하기 때문에, 
인스턴스간 공유하는 `mutable state` 가 없다. 
그리고 `split()` 으로 생성된 여러 인스턴스들에서 생성된 난수는 하나의 `SplittableRandom` 인스턴스에서 
생성한 난수의 품질이 동일하다.  


## SecureRandom
`SecureRandom` 은 난수 생성 알고리즘을 직접 선택할 수 있다. 
그래서 지금까지 설명한 `Random`, `ThreadLocalRandom`, `SplittableRandom` 들 보다 
더욱 좋은 품질의 난수 생성이 가능하다. 
대부분 보안이 중요한 경우 `SecureRandom` 을 사용한다.  

```java
SecureRandom random = new SecureRandom();
// or
//  SecureRandom random = SecureRandom.getInstanceStrong();

int randomValue = random.nextInt();
// 0 ~ 99
int boundedRandomValue = random.nextInt(100);
```  

`Random` 은 `LCG` 알고리즘을 사용하기 때문에 `seed` 값에 따른 결과 패턴이 존재한다. 
이러한 이유로 보안이 중요한 경우 `seed` 값만 알아 낸다면 난수 재현이 가능하다는 문제가 있어 적합하지 않다. 
`SecureRandom` 은 `CSPRNG`([cryptographically strong pseudo-random number generator](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator))
알고리즘을 사용하기 때문에 [The Java SecureRandom Class](https://www.baeldung.com/java-secure-random) 
페이지에서 더 나은 품질 결과를 확인 할 수 있다.  

추가적으로 `SecureRandom` 이 사용하는 `CSPRNG` 가 사용 가능한 알고리즘은 아래와 같다. 
- `NativePRNG`
- `NativePRNGBlocking`
- `NativePRNGNonBlocking`
- `PKCS11`
- `SHA1PRNG`(기본 알고리즘)
- `Windows-PRNG` 

특정 알고리즘으로 인스턴스를 생성하는 방법은 아래와 같다.  

```java
SecureRandom random = SecureRandom.getInstance("NativePRNG");
```  

## 정리 및 성능테스트 
지금까지 `Java` 에서 난수를 생성할 수 있는 4가지 클래스에 대해서 알아 보았다. 
품질 측면에서 순으로 나열하면 아래와 같다. 
1. `SecureRandom`
2. `SplittableRandom`
3. `ThreadLocalRandom`
4. `Random`

그리고 멀티 스레딩 환경에서 성능에 대한 결과 확인을 위해 `JMH` 기반으로 성능 테스트를 진행해 보았다.  

```java
@Threads(value = 100)
public class RandomTest {
    private static final int LOOP_COUNT = 500;

    private static SecureRandom getSecureRandom() {
        try {
//            return SecureRandom.getInstanceStrong();
            return new SecureRandom();
        } catch (Exception ignore) {
            return null;
        }
    }

    // 전체 스레드 공유
    @State(Scope.Benchmark)
    public static class BenchmarkState {
        volatile Random random = new Random();
        volatile ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();

        volatile SplittableRandom splittableRandom = new SplittableRandom();

        volatile SecureRandom secureRandom = getSecureRandom();
    }

    // 스레드간 공유 X
    @State(Scope.Thread)
    public static class ThreadState {
        volatile Random random = new Random();
        volatile ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();

        volatile SplittableRandom splittableRandom = new SplittableRandom();

        volatile SecureRandom secureRandom = getSecureRandom();
    }

    // Random 전체 스레드 공유하는 경우
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void shareRandom(BenchmarkState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.random.nextInt();
        }

        blackhole.consume(sum);
    }

    // ThreadLocalRandom 전체 스레드 공유하는 경우
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void shareThreadLocalRandom(BenchmarkState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.threadLocalRandom.nextInt();
        }

        blackhole.consume(sum);
    }

    // SplittableRandom 전체 스레드 공유하는 경우
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void shareSplittableRandom(BenchmarkState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.splittableRandom.nextInt();
        }

        blackhole.consume(sum);
    }

    // SecureRandom 전체 스레드 공유하는 경우
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void shareSecureRandom(BenchmarkState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.secureRandom.nextInt();
        }

        blackhole.consume(sum);
    }

    // Random 스레드마다 인스턴스 생성
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void unShareRandom(ThreadState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.random.nextInt();
        }

        blackhole.consume(sum);
    }

    // ThreadLocalRandom 스레드마다 인스턴스 생성
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void unShareThreadLocalRandom(ThreadState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.threadLocalRandom.nextInt();
        }

        blackhole.consume(sum);
    }

    // SplittableRandom 스레드마다 인스턴스 생성(new)
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void unShareSplittableRandom(ThreadState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.splittableRandom.nextInt();
        }

        blackhole.consume(sum);
    }

    // SplittableRandom 스레드마다 인스턴스 생성(split())
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void unShareSplitSplittableRandom(BenchmarkState state, Blackhole blackhole) throws InterruptedException {
      long sum = 0;
      SplittableRandom random = state.splittableRandom.split();
      for(int i = 0; i < LOOP_COUNT; i++) {
        sum += random.nextInt();
      }
  
      blackhole.consume(sum);
    }

    // SecureRandom 스레드마다 인스턴스 생성
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public void unShareSecureRandom(ThreadState state, Blackhole blackhole) throws InterruptedException {
        long sum = 0;
        for(int i = 0; i < LOOP_COUNT; i++) {
            sum += state.secureRandom.nextInt();
        }

        blackhole.consume(sum);
    }
}
```  

`gradle jmh` 명령을 통해 성능 테스트를 진행할 수 있고, 
그 결과는 아래와 같다. 

```bash
Benchmark                                             Mode  Cnt      Score   Error  Units
RandomTest.shareRandom                   avgt       14302.677          us/op
RandomTest.shareSecureRandom             avgt       11577.258          us/op
RandomTest.shareSplittableRandom         avgt         148.094          us/op
RandomTest.shareThreadLocalRandom        avgt          13.005          us/op
RandomTest.unShareRandom                 avgt          48.697          us/op
RandomTest.unShareSecureRandom           avgt       12273.854          us/op
RandomTest.unShareSplittableRandom       avgt          13.556          us/op
RandomTest.unShareSplitSplittableRandom  avgt           8.244          us/op
RandomTest.unShareThreadLocalRandom      avgt          13.252          us/op
```  

정상적이지 않은 사용방법도 포함된 성능 테스트이다. 
멀티 스레드 환경에서 정상적으로 사용했을 때 성능이 좋은 순으로 나열하면 아래와 같다. 
- `SplittableRandom`
- `ThreadLocalRandom`
- `Random`
- `SecureRandom`

`SplittableRandom` 이 `split()` 메서드를 바탕으로 스레드마다 새로운 인스턴스를 생성하면 
가장 좋은 성능을 보여주었지만, 이러한 제어가 불가능 한 경우에는 `ThreadLocalRandom` 이 가장 좋은 성능으로 간편하게 사용 할 수 있을 것으로 보인다.  



---
## Reference
[Guide to ThreadLocalRandom in Java](https://www.baeldung.com/java-thread-local-random)  
[Random vs ThreadLocalRandom Classes in Java](https://www.geeksforgeeks.org/random-vs-threadlocalrandom-classes-in-java/)  


