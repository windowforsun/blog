--- 
layout: single
classes: wide
title: "[Java 개념] Stream API"
header:
  overlay_image: /img/java-bg.jpg
excerpt: '람다를 사용해서 데이터 집합을 간결하고 깔끔하게 처리 및 가공할 수 있는 스트림에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Stream
    - Lamda
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


![그림 1]({{site.baseurl}}/img/java/concept_stream_1.png)

1. 생성(`Create`) : 배열, 컬렉션을 기반으로 스트림을 생성한다. 
1. 중간 연산(`Intermediate`) : `filter`, `sorted` 와 같은 중간 연산은 새로운 스트림을 반환한다. 
그러므로 체이닝 방식으로 사용해서 여러 동작, 질의를 연결해서 구성할 수 있다. 
중간 연산은 최종연산 호출 전까지는 수행되지 않는다. 
중간 연산을 합친 다음에 합쳐진 중간 연산을 최종 연산으로 한번 처리하는 방식으로 
이전에 설명했던 스트림의 `Lazy` 특정은 중간연산에 해당하는 내용이다. 
1. 최종연산(`Terminal`) : 최종 연산은 스트림 파이프라인에서 결과를 도출하는 연산이다. 
일반적으론 최종 연산의 결과값으로 컬렉션이나 `Integer`,`void` 를 반환한다. 
또는 `System.out::println` 을 통해 연산의 결과를 출력해 볼 수도 있다.  



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

#### Filtering
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

#### Mapping
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
`Stream.flatMap` 메소드는 `Function` 타입의 람다식의 인자로 스트림의 요소가 입력으로 들어가고, 
중첩구조에서 스트림에 추가할 스트림을 리턴하는 방식으로 사용한다.  

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

#### Sorting
`Sorting` 은 스트림의 요소를 정렬하는 작업이다. 
`Stream.sorted` 메소드는 오버로딩으로 돼있어, 
파라미터가 없는 메소드와 `Comparator` 를 파라미터로 갖는 메소드가 있다. 
파라미터가 없는 메소드는 스트림 요소를 내림차순으로 정렬하고, 
`Comparator` 파라미터를 갖는 메소드를 사용하면 사용자 정의에 따라 정렬 조건을 정의할 수 있다.  

```java
@Test
public void sorting_IntList_Asc() {
    // given
    List<Integer> list = Arrays.asList(
            5, 1, 2, 4, 3
    );

    // when
    Stream<Integer> actual = list.stream()
            .sorted();

    // then
    List<Integer> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(5));
    assertThat(listActual, contains(
            1, 2, 3, 4, 5
    ));
}

@Test
public void sorting_StringList_Desc() {
    // given
    List<String> list = Arrays.asList(
            "One", "Two", "Three", "Four"
    );

    // when
    Stream<String> actual = list.stream()
            .sorted(Comparator.reverseOrder());

    // then
    List<String> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(4));
    assertThat(listActual, contains(
            "Two", "Three", "One", "Four"
    ));
}

@Test
public void sorting_CustomObjectList_Comparator() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(4, "a"),
            new MyClass(1, "b"),
            new MyClass(10, "c")
    );

    // when
    Stream<MyClass> actual = list.stream()
            .sorted((first, second) -> {
                return first.getNum() - second.getNum();
            });

    // then
    List<MyClass> listActual = actual.collect(Collectors.toList());
    assertThat(listActual, hasSize(3));
    assertThat(listActual, contains(
            hasProperty("num", is(1)),
            hasProperty("num", is(4)),
            hasProperty("num", is(10))
    ));
}
```  

#### Iterating
`peek` 은 스트림내 요소를 방문하며 연산을 수행하는 메소드이다. 
`Stream.peek` 은 `Comsumer` 타입의 람다를 파라미터로 갖는데, 
이는 람다에서 인자는 받지만 리턴하는 하지 않는다. 
그러므로 스트림 파이프라인 결과에 영향을 미치는 작업이 아닌 `System.out::print` 과 같이 중간 처리 결과를 확인하는 용도로 사용할 수 있다. 

```java
@Test
public void peek_IntList_Stdout() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    list.stream()
            .peek(System.out::println)
            .collect(Collectors.toList());

    // then
    // is output as follows
    // 1
    // 2
    // 3
    // 4
    // 5
}
```  


### Terminal Operation
스트림을 생성하고 스트림 요소에 대한 연산과정을 정의했다면, 
최종적으로 필요한 결과값을 만드는 단계이다.  



#### Calculating
스트림의 종료작업 중 합, 평균, 최대, 최소등 기본적인 숫자형 타입에 대한 계산작업 결과를 만들어 낼 수 있다. 
스트림이 비었을 경우 `sum`, `count` 는 0을 리턴하고 그외 메소드는 `Optional` 타입을 리턴 하는 방법으로 이를 해결한다. 

```java
@Test
public void count_IntList() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    long actual = list.stream().count();

    // then
    assertThat(actual, is(5L));
}

@Test
public void sum_IntList() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    long actual = list.stream()
            .mapToInt(Integer::intValue)
            .sum();

    // then
    assertThat(actual, is(15L));
}

@Test
public void min_IntList() {
    // given
    IntStream intStream = IntStream.of(1, 2, 3, 4, 5);

    // when
    OptionalInt actual = intStream.min();

    // then
    assertThat(actual.isPresent(), is(true));
    assertThat(actual.getAsInt(), is(1));
}

@Test
public void max_LongList() {
    // given
    LongStream longStream = LongStream.of(1L, 2L, 3L, 4L, 5L);

    // when
    OptionalLong actual = longStream.max();

    // then
    assertThat(actual.isPresent(), is(true));
    assertThat(actual.getAsLong(), is(5L));
}

@Test
public void average_DoubleList() {
    // given
    DoubleStream doubleStream = DoubleStream.of(1d, 2d, 3d, 4d, 5d);

    // when
    OptionalDouble actual = doubleStream.average();

    // then
    assertThat(actual.isPresent(), is(true));
    assertThat(actual.getAsDouble(), is(3d));
}

@Test
public void sum_Empty_LongList() {
    // given
    LongStream longStream = LongStream.empty();

    // when
    long actual = longStream.sum();

    // then
    assertThat(actual, is(0L));
}

@Test
public void average_Empty_IntList() {
    // given
    IntStream intStream = IntStream.empty();

    // when
    OptionalDouble actual = intStream.average();

    // then
    assertThat(actual.isPresent(), is(false));
}

@Test
public void orElse_IntList() {
    // given
    IntStream intStream = IntStream.empty();

    // when
    int actual = intStream.max().orElse(-1);

    // then
    assertThat(actual, is(-1));
}

@Test
public void ifPresent_IntList() {
    // given
    IntStream intStream = IntStream.of(1, 2, 3, 4, 5);

    // when
    List<Integer> actual = new ArrayList<>();
    intStream.max().ifPresent(actual::add);

    // then
    assertThat(actual, hasSize(1));
    assertThat(actual, hasItem(5));
}
```  

#### Reduction
`Reduction` 관련 연산은 `Stream.reduce` 메소드를 사용해서, 
스트림내 요소에 대한 사용자 정의가 가능한 결과를 만들어 낼 수 있다. 
`Stream.reduce` 메소드는 오버로딩 구조로 아래와 같이 구성돼있다.  
- `Optional<T> reduce(BinaryOperator<T> accumulator);`
- `T reduce(T identity, BinaryOperator<T> accumulator);`
- `<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);`

`Stream.reduce` 의 파라미터에는 아래와 같은 종류로 구성된다. 
- `accumulator` : 스트림 요소를 파라미터로 받아 로직에 따른 결과를 리턴하는 람다식을 작성한다. 
- `identity` : 결과 계산을 위한 초기값을 의미하고, 스트림이 비어있다면 초기값이 결과값이 된다. 
- `combiner` : 병렬 스트림에서 나눠 계산한 결과를 하나로 합치는 로직을 람다식으로 작성한다. 

`Stream.reduce` 는 메소드 정의 부분에서 알 수 있듯이, 
스트림내 요소와 같은 타입의 결과를 만들어 내는데 사용할 수 있다. 

```java
@Test
public void reduce_Accumulator_IntList_Sum() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    Optional<Integer> actual = list.stream()
            .reduce((a, b) -> {
                return Integer.sum(a, b);
            });

    // then
    assertThat(actual.isPresent(), is(true));
    assertThat(actual.get(), is(15));
}

@Test
public void reduce_Identity_Accumulator_IntList_Sum() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    int actual = list.stream()
            .reduce(10, Integer::sum);

    // then
    assertThat(actual, is(25));
}

@Test
public void reduce_Identity_Accumulator_Combiner_IntList_Sum_Only_ParallelStream1() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    int actual = list.parallelStream()
            .reduce(
                    10,
                    Integer::sum,
                    (partialA, partialB) ->  partialA + partialB
            );

    // then
    assertThat(actual, is(65));
}

@Test
public void reduce_Identity_Accumulator_Combiner_IntList_Sum_Only_ParallelStream2() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    int actual = list.parallelStream()
            .reduce(
                    10,
                    (identity, num) ->  identity + num,
                    Integer::sum
            );

    // then
    assertThat(actual, is(65));
}

@Test
public void reduce_Identity_Accumulator_Combiner_IntList_Sum_Only_ParallelStream3() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    int actual = list.parallelStream()
            .reduce(
                    10,
                    (identity, num) -> num,
                    Integer::sum
            );

    // then
    assertThat(actual, is(15));
}

@Test
public void reduce_Identity_Accumulator_Combiner_IntList_Sum_Only_ParallelStream4() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    int actual = list.parallelStream()
            .reduce(
                    0,
                    Integer::sum,
                    Integer::sum
            );

    // then
    assertThat(actual, is(15));
}
```  

`parallelStream` 부분을 다시한번 확인해보자. 
`[1, 2, 3, 4, 5]` 배열에서 `identity` 에 10을 주고, `accumulator`, `combinder` 에 모두 `Integer::sum` 동작을 수행하면 
결과는 65가 나온다. 
먼저 `accumulator` 에서 아래와 같은 동작으로 초기값과 연산을 수행한다. 
- `10 + 1 = 11`
- `10 + 2 = 12`
- `10 + 3 = 13`
- `10 + 4 = 14`
- `10 + 5 = 15`

그리고 `conbiner` 에서는 초기값과 연산한 결과를 합치는 동작을 아래와 같이 수행한다. 
실제 동작과정과 순서는 상황에 따라 다를 수 있지만 결과 값은 동일하다. 
- `11 + 12 = 23`
- `13 + 14 = 27`
- `27 + 15 = 42`
- `23 + 42 = 65` 

#### Collecting
`Collecting` 또한 앞서 설명한 `Reduction` 과 많은 부분에서 비슷한 연산을 할 수 있지만 몇가지 차이점이 있다. 
먼저 `Collecting` 연산을 수행하는 `Stream.collect` 메소드는 아래와 같다. 
- `<R> R collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner);`
- `<R, A> R collect(Collector<? super T, A, R> collector);`

주로 `Collector` 를 리턴하는 `Collectors` 클래스의 전역 메소드를 사용해서 관련처리를 수행할 수 있고, 
`Stream.reduce` 와 비슷하게 `supplier`, `accumulator`, `combinder` 를 정의해서 결과를 만들어 낼 수도 있다.  

`Stream.collect` 는 기본적으로 `Stream.reduce` 와 달리 스트림내 요소와 다른 타입의 결과를 만들어 낼 수 있는, 
`mutable reduction` 이라고 부를 수 있다. 
간단한 예로 아래 코드를 참고한다. 

```java
@Test
public void reduce_Concat_String() {
    // given
    List<String> list = Arrays.asList("a", "b", "c");

    // when
    String actual = list.stream()
            .reduce(new String(),
                    (str, el) -> str.concat(el),
                    (str1, str2) -> str1.concat(str2));

    // then
    assertThat(actual, is("abc"));
}

@Test
public void collect_Concat_String() {
    // given
    List<String> list = Arrays.asList("a", "b", "c");

    // when
    StringBuilder actual = list.stream()
            .collect(StringBuilder::new,
                    (sb, str) -> sb.append(str),
                    (sb1, sb2) -> sb1.append(sb2.toString()));

    // then
    assertThat(actual.toString(), is("abc"));
}
```  

위 두가지 `TC` 모두 주어진 문자열 리스트를 하나의 문자열로 합치는 연산을 위해 스트림을 사용하고 있다. 
하지만 `Stream.reduce` 를 사용하는 경우는 `String` 객체를 사용하고, 
`Stream.collect` 를 사용하는 경우에는 `StringBuilder` 객체를 사용해서 동작을 수행하고 있다. 
우선 `Stream.collect` 를 사용할 때는 `String`, `StringBuilder` 모두 사용가능하지만, 
`Stream.reduce` 는 현재 스트림의 요소와 타입이 일치하는 `String` 만 사용가능하다. 
만약 `Stream.reduce` 를 사용하면서 `StringBuilder` 를 통해 문자열을 합치고 싶다면, 
스트림 요소 타입이 `StringBuilder` 이거나 `map` 을 사용해서 한번 `String` 타입을 `StringBuilder` 로 만들어야 한다.  

```java
StringBuilder actual = list.stream()
        .map((str) -> new StringBuilder(str))
        .reduce(new StringBuilder(),
                (str, el) -> str.append(el),
                (str1, str2) -> str1.append(str2));
```  

실제로 위 2케이스를 비교한다면, `Stream.collect` 를 사용하는 경우가 더욱 이득일 것이다. 
그 이유는 `String` 은 `Immutable` 한 성질로 계속해서 새로운 인스턴스를 만들어내지만, 
`StringBuilder` 는 `Mutable` 한 성질로 문자열을 조작하기 때문이다.  

`Stream.collect` 은 아래와 같이 사용해서 스트림 파이프라인에 대한 결과를 만들어 낼 수 있다. 

```java
@Test
public void collect_CustomObject_To_FieldList() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    List<String> actual = list.stream()
            .map(MyClass::getStr)
            .collect(Collectors.toList());

    // then
    assertThat(actual, hasSize(3));
    assertThat(actual, contains(
            "a", "b", "c"
    ));
}

@Test
public void collect_CustomObject_To_StringFieldJoin() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    String actual = list.stream()
            .map(MyClass::getStr)
            .collect(Collectors.joining());

    // then
    assertThat(actual, is("abc"));
}

@Test
public void collect_CustomObject_To_StringFieldJoin_Args() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    String actual = list.stream()
            .map(MyClass::getStr)
            .collect(Collectors.joining(",", "{", "}"));

    // then
    assertThat(actual, is("{a,b,c}"));
}

@Test
public void collect_CustomObject_To_NumericFieldAverage() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    double actual = list.stream()
            .collect(Collectors.averagingInt(MyClass::getNum));

    // then
    assertThat(actual, is(2d));
}

@Test
public void collect_CustomObject_To_NumericFieldSum() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    int actual = list.stream()
            .collect(Collectors.summingInt(MyClass::getNum));

    // then
    assertThat(actual, is(6));
}

@Test
public void collect_CustomObject_To_NumericFieldSummarizing() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c")
    );

    // when
    IntSummaryStatistics actual = list.stream()
            .collect(Collectors.summarizingInt(MyClass::getNum));

    // then
    assertThat(actual.getSum(), is(6L));
    assertThat(actual.getAverage(), is(2d));
    assertThat(actual.getCount(), is(3L));
    assertThat(actual.getMin(), is(1));
    assertThat(actual.getMax(), is(3));
}

@Test
public void collect_CustomObject_To_GroupingBy() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c"),
            new MyClass(1, "aa"),
            new MyClass(2, "bb"),
            new MyClass(3, "cc")
    );

    // when
    Map<Integer, List<MyClass>> actual = list.stream()
            .collect(Collectors.groupingBy(MyClass::getNum));

    // then
    assertThat(actual.size(), is(3));
    assertThat(actual, hasEntry(is(1), hasSize(2)));
    assertThat(actual, hasEntry(is(1), contains(
            hasProperty("str", is("a")), hasProperty("str", is("aa"))
    )));
    assertThat(actual, hasEntry(is(2), hasSize(2)));
    assertThat(actual, hasEntry(is(2), contains(
            hasProperty("str", is("b")), hasProperty("str", is("bb"))
    )));
    assertThat(actual, hasEntry(is(3), hasSize(2)));
    assertThat(actual, hasEntry(is(3), contains(
            hasProperty("str", is("c")), hasProperty("str", is("cc"))
    )));
}

@Test
public void collect_CustomObject_To_PartitioningBy() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c"),
            new MyClass(1, "aa"),
            new MyClass(2, "bb"),
            new MyClass(3, "cc")
    );

    // when
    Map<Boolean, List<MyClass>> actual = list.stream()
            .collect(Collectors.partitioningBy(myClass -> myClass.getStr().length() == 1));

    // then
    assertThat(actual.size(), is(2));
    assertThat(actual, hasEntry(is(true), hasSize(3)));
    assertThat(actual, hasEntry(is(true), contains(
            hasProperty("str", is("a")), hasProperty("str", is("b")), hasProperty("str", is("c"))
    )));
    assertThat(actual, hasEntry(is(false), hasSize(3)));
    assertThat(actual, hasEntry(is(false), contains(
            hasProperty("str", is("aa")), hasProperty("str", is("bb")), hasProperty("str", is("cc"))
    )));
}

@Test
public void collect_IntList_To_CollectingAndThen_SynchronizedList() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    List<Integer> actual = list.stream()
            .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::synchronizedList));

    // then
    assertThat(actual, hasSize(5));
    assertThat(actual, contains(
            1, 2, 3, 4, 5
    ));
}

@Test
public void collect_IntList_To_CollectingAndThen_UnmodifiedSet() {
    // given
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // when
    Set<Integer> actual = list.stream()
            .collect(Collectors.collectingAndThen(Collectors.toSet(), Collections::unmodifiableSet));

    // then
    assertThat(actual, hasSize(5));
    assertThat(actual, contains(
            1, 2, 3, 4, 5
    ));
}

@Test
public void collect_CustomObject_To_Of_StringFieldList() {
    // given
    List<MyClass> list = Arrays.asList(
            new MyClass(1, "a"),
            new MyClass(2, "b"),
            new MyClass(3, "c"),
            new MyClass(1, "aa"),
            new MyClass(2, "bb"),
            new MyClass(3, "cc")
    );
    Collector<MyClass, ?, LinkedList<String>> customCollector =
            Collector.of(LinkedList::new,
                    (linkedList, myClass) -> linkedList.add(myClass.getStr()),
                    (first, second) -> {
                        first.addAll(second);
                        return first;
                    });

    // when
    LinkedList<String> actual = list.stream()
            .collect(customCollector);

    // then
    assertThat(actual, hasSize(6));
    assertThat(actual, contains(
            "a", "b", "c", "aa", "bb", "cc"
    ));
}
```  

#### Matching
`Matching` 은 몇가지 `Predicate` 타입 람다식을 사용해서 조건을 만족하는 요소가 존재하는지에 대한 `boolean` 값을 리턴한다. 
- `anyMatch` : 요소 중 조건을 만족하는 요소가 단 하나라도 존재하는지 
- `allMath` : 모든 요소가 조건을 모두 만족하는지
- `noneMath` : 모든 요소가 조건을 모두 만족하지 않는지

```java
@Test
public void matching_anyMatch() {
    // given
    List<String> list = Arrays.asList(
            "a", "b", "c", "ab", "ba"
    );

    // when
    boolean actual = list.stream()
            .anyMatch(str -> str.contains("c"));

    // then
    assertThat(actual, is(true));
}

@Test
public void matching_allMath() {
    // given
    List<String> list = Arrays.asList(
            "a", "b", "c", "ab", "ba"
    );

    // when
    boolean actual = list.stream()
            .allMatch(str -> !str.isEmpty());

    // then
    assertThat(actual, is(true));
}

@Test
public void matching_noneMatch() {
    // given
    List<String> list = Arrays.asList(
            "a", "b", "c", "ab", "bc"
    );

    // when
    boolean actual = list.stream()
            .noneMatch(str -> str.contains("d"));

    // then
    assertThat(actual, is(true));
}
```  

#### Iterating
`Iterating` 은 `Stream.forEach` 메소드를 사용해서 메소드 참조를 넘겨, 
스트림 전체 요소에 대한 처리를 수행하는 작업을 수행할 수 있다. 
중간 작업에 해당하는 `Stream.peek` 과 비슷한 동작을 수행하지만 시점에서 차이가 있다. 

```java
@Test
public void forEach_Stdout() {
    // given
    List<Integer> list = Arrays.asList(
            1, 2, 3, 4, 5
    );

    // when
    list.stream()
            .forEach(System.out::println);

    // then
    // is output as follows
    // 1
    // 2
    // 3
    // 4
    // 5
}
```  

---
## Reference
[Package java.util.stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)  
[Stream (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)  
[What are Java 8 streams?](https://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/stream-api-intro.html)  
[Java 8 Streams - Lazy evaluation](https://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/lazy-evaluation.html)  
