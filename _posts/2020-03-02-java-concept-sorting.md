--- 
layout: single
classes: wide
title: "[Java 개념] 정렬 Comparable, Comparator"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java 에서 정렬과 Comparable, Comparator 의 차이에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Sorting
    - Comparable
    - Comparator
toc: true
use_math: true
---  

## Java Sorting
- Java 에서는 Array 나 Collection 을 정렬을 위해 `Comparable` 과 `Comparator` 인터페이스를 제공한다.
- 두 인터페이스 모두 정렬의 기준을 명시하는 인터페이스이지만, 서로다른 역할을 가지고 있다.
- Java 에서 제공하는 정렬관련 메소드를 사용하기 위해서는 위 2개 인터페이스 중 하나는 구현해야 한다.
- 배열 정렬은 `java.util.Arrays`  패키지의 `sort` 메소드를 사용한다.
- Collection 정렬은 `java.util.Collections` 패키지의 `sort` 메소드를 사용한다.

### Comparable

```java
public interface Comparable<T> {
    public int compareTo(T o);
}
```  

- `Comparable` 인터페이스는 객체에 대한 기본적인 정렬기준을 하위 클래스에서 구현한다.
- Java 문서에서는 `Comparable` 의 정렬기준을 `natural ordering` 이라고 한다. 
- Java 정렬 메소드에서는 `Comparable` 의 `compareTo` 메소드를 바탕으로 정렬을 수행한다.
- `String`, `Integer` 등의 Java 에서 제공되는 클래스는 `Comparable` 을 구현하고 있다.
- `comparaTo` 메소드는 자신과 인자로 넘어온 객체를 비교할 수 있다.
	- 자신이 더 크다면 양수값, 작다면 음수 값, 같다면 0 을 리턴하면 된다.
	- 인자값 보다 자신이 더 클때 양수를 리턴하면 오름차순 정렬, 음수를 리턴하면 내림차순 정렬이 된다.
- `Arrays`, `Collections` 의 `sort` 메서드에서 해당 객체가 구현한 구현체를 사용한다.

### Comparator

```java
public interface Comparator<T> {
	int compare(T o1, T o2);
	
	// 생략
}
```  

- `Comparator` 인터페이스는 기본 정렬기준이 아닌 다른 정렬 기준을 하위 클래스에서 구현한다.
- Java 정렬 메소드에서는 `Comparator` 의 `compare` 메소드를 바탕으로 정렬을 수행한다.
- `compare` 메소드는 인자값으로 주어진 2개의 객체를 비교할 수 있다.
	- `o1`, `o2` 두 객체가 모두 같다면 0 을 리턴한다.
	- `o2` 보다 `o1` 객체가 더 클때 양수를 리턴하면 오름차순 정렬, 음수를 리턴하면 내림차순 정렬이 된다.
- `Arrays`, `Collections` 의 `sort` 메서드의 인자값으로 구현체를 전달한다.
- 구현체를 `Lamda` 함수로 대체 가능하다.

### 테스트

```java
public class Computer implements Comparable<Computer> {
    private String serialCode;
    private int benchmarkScore;
    private int price;

    public Computer(String serialCode, int benchmarkScore, int price) {
        this.serialCode = serialCode;
        this.benchmarkScore = benchmarkScore;
        this.price = price;
    }

    // Comparable 인터페이스의 기본 Computer 객체에 대한 기본 정렬 기준(benchmarkScore 의 오름차순)
    @Override
    public int compareTo(Computer o) {
        return this.benchmarkScore - o.benchmarkScore;
    }
    
    // getter, setter
}
```  

```java
public class SortingTest {
    private Computer[] computerArray;
    private Computer a, b, c, d;

    @Before
    public void setUp() {
        this.a = new Computer("a", 10, 10000);
        this.b = new Computer("b", 9, 20000);
        this.c = new Computer("c", 20, 100000);
        this.d = new Computer("d", 5, 8000);

        this.computerArray = new Computer[4];
        this.computerArray[0] = this.a;
        this.computerArray[1] = this.b;
        this.computerArray[2] = this.c;
        this.computerArray[3] = this.d;
    }

    @Test
    public void 기본정렬기준_Comparable_Array_benchmarkScore기준오름차순() {
        // when
        Arrays.sort(this.computerArray);

        // then
        assertThat(this.computerArray[0], is(this.d));
        assertThat(this.computerArray[1], is(this.b));
        assertThat(this.computerArray[2], is(this.a));
        assertThat(this.computerArray[3], is(this.c));
    }

    @Test
    public void 기본정렬기준_Comparable_Collection_benchmarkScore기준오름차순() {
        // given
        List<Computer> computerList = new ArrayList<>(Arrays.asList(this.computerArray));

        // when
        Collections.sort(computerList);

        // then
        assertThat(computerList.get(0), is(this.d));
        assertThat(computerList.get(1), is(this.b));
        assertThat(computerList.get(2), is(this.a));
        assertThat(computerList.get(3), is(this.c));
    }

    @Test
    public void 다른정렬기준_Comparator_Array_price기준오름차순() {
        // when
        Arrays.sort(this.computerArray, new Comparator<Computer>() {
            @Override
            public int compare(Computer o1, Computer o2) {
                return o1.getPrice() - o2.getPrice();
            }
        });

        // then
        assertThat(this.computerArray[0], is(this.d));
        assertThat(this.computerArray[1], is(this.a));
        assertThat(this.computerArray[2], is(this.b));
        assertThat(this.computerArray[3], is(this.c));
    }

    @Test
    public void 다른정렬기준_Comparator_Collection_price기준오름차순() {
        // given
        List<Computer> computerList = new ArrayList<>(Arrays.asList(this.computerArray));

        // when
        Collections.sort(computerList, new Comparator<Computer>() {
            @Override
            public int compare(Computer o1, Computer o2) {
                return o1.getPrice() - o2.getPrice();
            }
        });

        // then
        assertThat(computerList.get(0), is(this.d));
        assertThat(computerList.get(1), is(this.a));
        assertThat(computerList.get(2), is(this.b));
        assertThat(computerList.get(3), is(this.c));
    }

    @Test
    public void 다른정렬기준_Comparator_Collection_price기준내림차순_Lamda() {
        // given
        List<Computer> computerList = new ArrayList<>(Arrays.asList(this.computerArray));

        // when
        Collections.sort(computerList, (a, b) -> b.getPrice() - a.getPrice());

        // then
        assertThat(computerList.get(0), is(this.c));
        assertThat(computerList.get(1), is(this.b));
        assertThat(computerList.get(2), is(this.a));
        assertThat(computerList.get(3), is(this.d));
    }

    @Test
    public void 다른정렬기준_Comparator_Collection_price기준내림차순_Stream() {
        // given
        List<Computer> computerList = new ArrayList<>(Arrays.asList(this.computerArray));

        // when
        List<Computer> actual = computerList.stream().sorted((a, b) -> b.getPrice() - a.getPrice()).collect(Collectors.toList());

        // then
        assertThat(actual.get(0), is(this.c));
        assertThat(actual.get(1), is(this.b));
        assertThat(actual.get(2), is(this.a));
        assertThat(actual.get(3), is(this.d));
    }
}
```  

---
## Reference
[Comparable (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/lang/Comparable.html)
[Comparator (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html)