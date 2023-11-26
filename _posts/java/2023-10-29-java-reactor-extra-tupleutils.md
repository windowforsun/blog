--- 
layout: single
classes: wide
title: "[Java 실습] Reactive Stream, Reactor Extra TupleUtils"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 zip 연산의 결과를 더 간편하게 사용 할 수 있는 Reactor Extra 의 TupleUtils 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Reactor Extra
  - TupleUtils
  - Zip
toc: true 
use_math: true
---  

## TupleUtils
[TupleUtils](https://projectreactor.io/docs/extra/release/api/index.html?reactor/function/TupleUtils.html)
는 `Reactor Extra` 에서 제공하는 유틸성 클리스 중 하나로, 
`zip` 연산 이후 `upstream` 에서 병합된 `Tuple` 에 담긴 결과를 `downstream` 에서 받아 처리할 떄의 불편함을 해소 할 수 있는 유틸리티성 클래스이다. 
`downstream` 에서 결과가 머지된 `Tuple` 의 데이터를 변환이나 추가 처리를 위해서는 값을 아래와 같이 하나씩 가져 오는 작업이 필요하다.  

```java
Mono.zip(Mono.just("a"), Mono.just(1), Mono.just(1L), Mono.just(1d), Mono.just(true))
        .map(tuple5 -> {
            String str = tuple5.getT1();
            int i = tuple5.getT2();
            long l = tuple5.getT3();
            double d = tuple5.getT4();
            boolean b = tuple5.getT5();
            
            return new Result(str, i, l, d, b);
        })
```  

`TupleUtils` 에 정의된 정적 메소드를 사용하면 최소 2개에서 부터 최대 8개 까지 원소를 갖는 `Tuple` 를 사용 할 수 있는, 
함수형 인터페이스와 유사한 인터페이스로 연결해준다. 
정적 메소드는 `consumer`, `function`, `predicate` 와 같이 3종류가 있는데 
이름과 매칭되는 역할을 수행한다.  

아래는 앞선 코드 예시와 대응되는 역할을 수행하는 `function` 을 사용해 개선한 결과이다.  

```java
Mono.zip(Mono.just("a"), Mono.just(1), Mono.just(1L), Mono.just(1d), Mono.just(true))
    .map(TupleUtils.function(Result::new))
```  

위 결과와 같이 `TupleUtils` 의 정적 메소드를 잘 활용하면 `Reactor` 의 `zip` 연산을 사용할 때 
코드를 좀더 간결하고, 명시적인 함수형 인터페이스 사용을 통해 가독성도 올릴 수 있다.  

아래는 좀더 자세한 `TupleUtils` 의 사용 예시 이다.  

```java
public class TupleUtilsTest {
  public String concatString(String str, int i) {
    return new StringBuilder()
            .append(str)
            .append(":")
            .append(i)
            .toString();
  }

  @Test
  public void mono_zip_map() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .map(tuple2 -> {
              String str = tuple2.getT1();
              int i = tuple2.getT2();

              return this.concatString(str, i);
            })
            .as(StepVerifier::create)
            .expectNext("a:1")
            .verifyComplete();
  }

  @Test
  public void mono_zip_map_with_tupleUtils_function() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .map(TupleUtils.function(this::concatString))
            .as(StepVerifier::create)
            .expectNext("a:1")
            .verifyComplete();
  }

  @Test
  public void mono_zip_doOnNext() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .doOnNext(tuple2 -> {
              String str = tuple2.getT1();
              int i = tuple2.getT2();

              System.out.println(this.concatString(str, i));
            })
            .as(StepVerifier::create)
            .expectNext(Tuples.of("a", 1))
            .verifyComplete();
  }

  @Test
  public void mono_zip_doOnNext_with_tupleUtils_consumer() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .doOnNext(TupleUtils.consumer((s, integer) -> System.out.println(this.concatString(s, integer))))
            .as(StepVerifier::create)
            .expectNext(Tuples.of("a", 1))
            .verifyComplete();
  }

  @Test
  public void mono_zip_filter() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .filter(tuple2 -> {
              String str = tuple2.getT1();
              int i = tuple2.getT2();

              return this.concatString(str, i).startsWith("b");
            })
            .as(StepVerifier::create)
            .verifyComplete();
  }

  @Test
  public void mono_zip_filter_with_tupleUtils_predicate() {
    Mono.zip(Mono.just("a"), Mono.just(1))
            .filter(TupleUtils.predicate((s, integer) -> this.concatString(s, integer).startsWith("b")))
            .as(StepVerifier::create)
            .verifyComplete();

  }
}
```  

---
## Reference
[Reactor Extra, TupleUtils](https://projectreactor.io/docs/extra/release/api/index.html?reactor/function/TupleUtils.html)  
[TupleUtils and Functional Interfaces](https://projectreactor.io/docs/core/release/reference/#extra-tuples)  

