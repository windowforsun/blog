--- 
layout: single
classes: wide
title: "[Java 개념] Stream API"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Test
    - Mockito
toc: true
use_math: true
---  

## Stream
스트림(`Stream`)은 선언형으로 컬렉션이나 배열 데이터를 처리할 수 있다. 
컬렉션, 배열 데이터를 `for` 나 `foreach` 를 사용해서 하나씩 조회하며 다루는 것이 아닌 
조금 더 데이터 흐름에 초점을 맞춰 우아하게 처리를 명시할 수 있다. 
그리고 스트림을 사용하면 멀티 스레드관련 처리를 구현하지 않아도 데이터를 병렬로 처리 할 수 있다.  

스트림의 특징을 정리하면 아래와 같은 것들이 있다. 
- `No Storage` : 스트림은 저장소나 데이터 구조체가 아닌 
배열, 컬렉션, `I/O` 채널 과같은 소스의 요소들을 연산 파이프라인으로 전달하는 역할을 한다. 
- `Functional in nature` : 스트림을 통해 수행되는 작업은 결과를 생성하지만 그 소스(배열, 컬렉션 ..)를 수정하지 않는다. 
스트림을 사용해서 필터링 작업을 수행한다면, 스트림은 소스는 수정하지않고 필터링에 해당하는 요소만 포함되는 새로운 스트림을 생성한다. 
- `Laziness-seeking` : 스트림을 사용하면 다양한 조건을 통해 소스를 탐색, 필터링, 제거 연산을 수행하게 된다. 
여기서 스트림은 이러한 연산을 `Lazy` 하게 처리한다. 
사용자가 데이터를 요청할 때만 소스를 바탕으로 데이터 처리를 하는 방식을 사용해서, 
불필요한 연산을 최소화 한다. 
- `Possibly unbounded` : 컬렉션, 배열은 크기가 정해져 있지만 스트림은 무한한 크기를 가질 수 있다. 
`Short-circuiting` 에는 `limit()`, `findFirst()` 와 같은 것들이 있는데, 
무한한 크기의 스트림에서 유한시간 내에 연산을 완료할 수 있도록 한다. 
- `Consumable` : 스트림은 일회용의 성격을 갖는다. 
스트림의 요소들은 단 한번 방문 가능하고 이는 `Iterator` 의 성격과 같다. 
만약 다시한번 요소를 방문해야 한다면, 새로운 스트림을 생성해야 한다. 

스트림과 컬렉션은 모두 순차적으로 데이터를 순회하는 기능을 제공한다. 
하지만 몇가지 차이점이 존재하는데, 
첫번째로는 연산시점이 있다. 
컬렉션은 연산을 위해서 자료구조가 포함하는 모든 값을 메모리에 적재해야 한다. 
그리고 컬렉션에 추가되기전에 데이터는 이미 연산이 수행된 상태여야 한다는 점이 있다. 
하지만 스트림은 요청할때 실제 연산을 수행해서 고정된 자료구조를 리턴하는 방식이다. 
이는 `Lazy` 하게 생성되는 컬렉션과 같다.  

두번째로는 외부반복과 내부반복이 있다. 
컬렉션은 다양한 방법으로 반복문을 사용할 수 있지만, 공통점은 사용자가 직접 반복문을 기술해야한다는 점이다. 
이러는 반복문을 외부 반복(`External Interation`) 이라고 한다. 
하지만 스트림은 컬렉션에서 사용하는 방법과 반대로 내부 반복(`Internal Interation`) 을 사용한다. 
반복문에 대한 동작만 명시해주면 스트림에서 수행하고 해당하는 값만 리턴한다. 
그리고 내부 반복은 투명한 작업을 제공하고, 
하드웨어를 활용한 병렬성 구현 자동화를 통해 최적화된 처리를 제공한다.  

하나의 파이프라인을 구성하는 스트림의 연산은 아래와 같은 과정과 연산으로 구성된다. 
1. 생성(`Create`) : 배열, 컬렉션을 기반으로 스트림을 생성한다. 
1. 중간 연산(`Intermediate`) : `filter`, `sorted` 와 같은 중간 연산은 새로운 스트림을 반환한다. 
그러므로 체이닝 방식으로 사용해서 여러 동작, 질의를 연결해서 구성할 수 있다. 
중간 연산은 최종연산 호출 전까지는 수행되지 않는다. 
중간 연산을 합친 다음에 합쳐진 중간 연산을 최종 연산으로 한번 처리하는 방식으로 
이전에 설명했던 스트림의 `Lazy` 특정은 중간연산에 해당하는 내용이다. 
1. 최종연산(`Terminal`) : 최종 연산은 스트림 파이프라인에서 결과를 도출하는 연산이다. 
일반적으론 최종 연산의 결과값으로 컬렉션이나 `Integer`,`void` 를 반환한다. 
또는 `System.out::println` 을 통해 연산의 결과를 출력해 볼 수도 있다.  

.. 그림 ..


### Create
일반적으로 스트림은 배열이나 컬렉션을 사용해서 생성하지만, 
이 외에도 다양한 방법으로 스트림을 생성할 수 있다.  

배열은 `Arrays.stream` 메소드를 사용해서 스트림을 생성할 수 있다. 

```java
@Test
public void array_All_To_Stream() {
    // given
    String[] strArray = new String[]{
            "a", "b", "c", "d", "e"
    };

    // when
    Stream<String> actual = Arrays.stream(strArray);

    // then
    assertThat(actual.toArray(String[]::new), arrayContaining(strArray));
}

@Test
public void array_Sub_To_Stream() {
    // given
    String[] strArray = new String[]{
            "a", "b", "c", "d", "e"
    };

    // when
    Stream<String> actual = Arrays.stream(strArray, 1, 4);

    // then
    assertThat(actual.toArray(String[]::new), arrayContaining(
            "b", "c", "d"
    ));
}
```  

컬렉션은 `Collection` 인터페이스에서 `default` 메소드인 `Collection.stream` 을 사용해서 스트림을 생성할 수 있다.  

```java
@Test
public void collection_All_To_Stream() {
    // given
    List<String> strList = Arrays.asList("a", "b", "c", "d", "e");

    // when
    Stream<String> actual = strList.stream();

    // then
    assertThat(actual.toArray(String[]::new), arrayContaining(
            "a", "b", "c", "d", "e"
    ));
}
```  

빈 스트림에 대한 표현은 스트림 객체에 `null` 값을 할당하는 방법으로도 가능하겠지만, 
`Stream.empty` 메소드를 사용해서 빈 스트림을 생성할 수 있다. 

```java
@Test
public void empty_Stream() {
    // when
    Stream<String> actual = Stream.empty();

    // then
    assertThat(actual.toArray(String[]::new), emptyArray());
}
```  

`Stream.builder` 메소드를 사용하면 직접 원하는 값을 추가하는 방법으로 스트림을 생성할 수 있다. 

```java
@Test
public void builder_Stream() {
    // when
    Stream<String> actual = Stream.<String>builder()
            .add("a")
            .add("b")
            .add("c")
            .add("d")
            .add("e")
            .build();

    // then
    assertThat(actual.toArray(String[]::new), arrayContaining(
            "a", "b", "c", "d", "e"
    ));
}
```  

`Stream.generate` 메소드는 인자가 없고 리턴만 있는 람다를 파라미터로 지정해서 스트림을 생성할 수 있다. 
여기서 `limit` 메소드로 개수를 명시해 주지 않으면 무한한 스트림이 생성된다. 

```java
@Test
public void generate_Stream() {
    // when
    Stream<String> actual = Stream.generate(() -> "str").limit(5);

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, everyItem(is("str")));
}
```  

`Stream.iterate` 메소드는 초기값과 초기값으로 반복해서 다루는 람다를 사용해서 스트림을 생성할 수 있다. 
이또한 `limit` 로 개수를 제한하지 않으면 무한한 스트름이 생성된다.

```java
@Test
public void iterate_Stream() {
    // when
    Stream<Integer> actual = Stream.iterate(1, n -> n * 2).limit(5);

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, contains(
            1, 2, 4, 8, 16
    ));
}
```  

지금까지는 `Stream` 객체에 제네릭 타입을 지정하는 방법으로 스트림을 사용했다. 
`java.util.stream` 패키지에서 스트림은 `BaseStream` 인터페이스 하위로 `Stream` 인터페이스 뿐만아니라, 
몇가지 기본 타입에 대한 스트림 인터페이스를 제공한다. 
이는 제네릭을 사용하지 않기 때문에 불필요한 오토박싱이 일어나지 않고, 
필요한 경우 `boxed` 메소드를 통해 다시 박싱을 할 수 있다. 

```java
@Test
public void int_Type_Stream_EndExclusive() {
    // when
    IntStream actual = IntStream.range(1, 5);

    // then
    List<Integer> listActual = actual.boxed().collect(Collectors.toList());
    assertThat(listActual, hasSize(4));
    assertThat(listActual, contains(
            1, 2, 3, 4
    ));
}

@Test
public void long_Type_Stream_EndInclusive() {
    // when
    LongStream actual = LongStream.rangeClosed(1, 5);

    // then
    List<Long> listActual = actual.boxed().collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, contains(
            1L, 2L, 3L, 4L, 5L
    ));
}

@Test
public void int_Type_Stream_Boxed() {
    // when
    Stream<Integer> actual = IntStream.range(1, 5).boxed();

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(4));
    assertThat(listActual, contains(
            1, 2, 3, 4
    ));
}
```  

그리고 몇가지 기본 타입에 대해 `Random` 클래스를 사용해서 랜덤값으로 구성된 스트림을 생성할 수 있다. 

```java
@Test
public void random_Stream() {
    // when
    Stream<Double> actual = new Random().doubles(5).boxed();

    // then
    List<Double> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, everyItem(instanceOf(Double.class)));
}
```  

`String.chars` 메소드를 사용하면 문자열을 사용해서 `char` 타입에 대한 스트림을 생성할 수 있다. 
타입은 `IntStream` 으로 리턴된다. 

```java
@Test
public void string_To_Stream() {
    // given
    String str = "0123456789";

    // when
    Stream<Integer> actual = str.chars().boxed();

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(10));
    assertThat(listActual, contains(
            48, 49, 50, 51, 52, 53, 54, 55, 56, 57
    ));
}
```  

그리고 정규표현식 클래스인 `Pattern` 에서 `Pattern.splitAsStream` 메소드를 사용하면, 
정규표현식을 사용해서 파싱한 문자열 스트림을 생성할 수 있다. 

```java
@Test
public void string_RegEx_To_Stream() {
    // given
    String str = "0 1 2 3 4 5 6 7 8 9";

    // when
    Stream<String> actual = Pattern.compile(" ").splitAsStream(str);

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(10));
    assertThat(listActual, contains(
            "0", "1", "2", "3", "4", "5", "6", "7", "8", "9"
    ));
}
```  


`java.nio` 패키지의 `Files` 클래스에서 `Files.lines` 메소드를 사용하면, 
파일을 읽어 라인단위로 구성된 스트림을 생성할 수 있다. 
`resources` 경로에 아래와 같은 `file-to-stream` 파일을 사용한다. 

```
0
1
2
3
4
5
6
7
8
9
```  

```java
@Test
public void file_To_Stream() throws Exception {
    // given
    String fileName = "file-to-stream";
    URL fileResource = getClass().getClassLoader().getResource(fileName);

    // when
    Stream<String> actual = Files.lines(Paths.get(fileResource.toURI()), Charset.forName("UTF-8"));

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(10));
    assertThat(listActual, contains(
            "0", "1", "2", "3", "4", "5", "6", "7", "8", "9"
    ));
}
```  

스트림을 병렬로 연산하기위해서는 병렬 스트림(`Parallel Stream`) 을 생성해야 한다. 
병렬 스트림은 지금까지 살펴본 스트림 생성에서 `parallel`, `parallelStream` 메소드를 사용하면 생성할 수 있다. 

```java
@Test
public void array_To_ParallelStream() {
    // given
    String[] strArray = new String[]{
            "a", "b", "c", "d", "e"
    };

    // when
    Stream<String> actual = Arrays.stream(strArray).parallel();

    // then
    assertThat(actual.isParallel(), is(true));
}

@Test
public void collection_To_ParallelStream() {
    // given
    List<String> strList = Arrays.asList("a", "b", "c", "d", "e");

    // when
    Stream<String> actual = strList.parallelStream();

    // then
    assertThat(actual.isParallel(), is(true));
}

@Test
public void other_To_ParallelStream() {
    // when
    Stream<Integer> actual = Stream.iterate(1, n -> n * 2).limit(5).parallel();

    // then
    assertThat(actual.isParallel(), is(true));
}
```  

병렬 스트림을 다시 `Sequential` 한 스트림으로 변경하고 싶다면, 
병렬 스트림에서 `sequential` 메소드로 가능하다. 

```java
@Test
public void parallelStream_To_Sequential() {
    // given
    Stream<Integer> parallel = Stream.iterate(1, n -> n * 2).limit(5).parallel();

    // when
    Stream<Integer> actual = parallel.sequential();

    // then
    assertThat(actual.isParallel(), is(false));
}
```  

미리 생성된 여러 스트림을 `Stream.concat` 메소드를 사용하면 하나의 스트림으로 연결해서 생성할 수 있다. 

```java
@Test
public void concat_Stream() {
    // given
    Stream<Integer> stream1 = Stream.iterate(1, n -> n * 2).limit(5);
    Stream<Integer> stream2 = IntStream.range(100, 103).boxed();

    // when
    Stream<Integer> actual = Stream.concat(stream1, stream2);

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(8));
    assertThat(listActual, contains(
            1, 2, 4 ,8, 16, 100, 101, 102
    ));
}
```  

### Intermediate Operation
스크림을 생성했다면 중간 연산과정을 통해 데이터를 조작할 수 있다. 
여러가지 중간 연산 `API` 를 사용해서 스트림을 가공하고자 하는 데이터로 만드는 과정을 기술 할 수 있다. 

필터(`fileter`)는 스트림내 요소들을 하나씩 검증하며 거르는 작업이다. 
인자로 받은 `Predicate` 타입의 람다식의 인자로 스트림의 요소가 입력으로 들어가고, 
스트림에서 필터링할 조건을 기술해서 `bool` 값을 리턴하는 방식으로 사용한다.  

```java
@Test
public void filtering_By_Custom() {
    // given
    List<String> strList = Arrays.asList("a", "b", "aa", "bb");

    // when
    Stream<String> actual = strList.stream()
            .filter(str -> str.startsWith("a"));

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(2));
    assertThat(listActual, contains(
            "a", "aa"
    ));
}
```  

맵(`map`)은 스트림내 요소들을 특정 값으로 변환(매핑)하는 작업이다. 
`Function` 타입의 람다식을 인자로 스트림 요소가 입력으로 들어가고, 
변환에 대한 규칙을 작성해서 리턴하는 방식으로 사용한다. 

```java
@Test
public void mapping_By_Custom() {
    // given
    List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);

    // when
    Stream<Integer> actual = intList.stream()
            .map(num -> num * 2);

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, contains(
            2, 4, 6, 8, 10
    ));
}

@Test
public void mapping_By_DefinedMethod() {
    // given
    List<String> strList = Arrays.asList("a", "b", "aa", "bb");

    // when
    Stream<String> actual = strList.stream()
            .map(String::toUpperCase);

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(4));
    assertThat(listActual, contains(
            "A", "B", "AA", "BB"
    ));
}

@Test
public void mapping_By_CustomObject_To_field() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    Stream<Integer> actual = list.stream()
            .map(MyClass::getNum);

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(3));
    assertThat(listActual, contains(
            1, 2, 3
    ));
}
```  

`flatMap` 은 중첩 배열, 중첩 리스트, 객체 배열과 같이 중첩된 구조를 한단계 제거하는 `flattening` 작업이다. 
`Function` 타입의 람다식의 인자로 스트림의 요소가 입력으로 들어가고, 
중첩구조에서 스트림으로 추가할 스트림을 리턴하는 방식으로 사용한다.  

```java
@Test
public void flatMap_By_DoubleList_To_SingleList() {
    // given
    List<List<Integer>> doubleList = Arrays.asList(
            Arrays.asList(1, 2, 3),
            Arrays.asList(4, 5, 6)
    );

    // when
    Stream<Integer> actual = doubleList.stream()
            .flatMap(Collection::stream);

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(6));
    assertThat(listActual, contains(
            1, 2, 3, 4, 5, 6
    ));
}

@Test
public void flatMap_By_DoubleList_To_UpperCaseStringList() {
    // given
    List<List<String>> doubleList = Arrays.asList(
            Arrays.asList("a", "b"),
            Arrays.asList("c", "d")
    );

    // when
    Stream<String> actual = doubleList.stream()
            .flatMap(list -> list.stream())
            .map(String::toUpperCase);

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(4));
    assertThat(listActual, contains(
            "A", "B", "C", "D"
    ));
}

@Test
public void flatMap_By_CustomObjectList_To_UpperCaseStrList() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    Stream<String> actual = list.stream()
            .flatMap(obj -> Stream.of(obj.getStr()))
            .map(String::toUpperCase);

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(3));
    assertThat(listActual, contains(
            "A", "B", "C"
    ));
}
```  









































































### Terminal Operation





















































---
## Reference
[Package java.util.stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)  
[Stream (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)  
