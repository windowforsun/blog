--- 
layout: single
classes: wide
title: "[Java 개념] Java8 특징 및 변경 사항 "
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java8의 특징과 변경 사항에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Java
    - Java8
---  

## Java8
- Java8은 2014년 3월 쯤 정식으로 발표 되었다.
- Java 는 2가지 버전 업으로 구분할 수 있다.
	- Evolution : 기능적인 부분의 개선이 주를 이루는 버전업
	- Revolution : 언어 자체의 형태에 변화가 오는 버전업
- Java 6, 7 은 Evolution 에 속하고 Java 5, 8은 Revolution 에 속한다.
- 그 만큼 Java8 에서는 많은 부분들이 변경되었고 하나씩 알아보도록 한다.

## Java8 의 추가 및 변경 사항
- Lambda 표현식
	- Functional Interface
	- Stream API
	- 인터페이스의 Default method
	- Method Reference
- 병렬 배열 처리
- 새로운 날짜 API
- StringJoiner
- Reflection 및 Annotation 개선
- Nashorn 자바스크립트엔진
- Collections API 확장
- Concurrency API 확장
- IO/NIO API 확장

### Lambda 표현식
- 익명 클래스를 사용할 때 가독성이 떨어지는 점을 보완하기 위해 람다 표현식이 만들어졌다.
- 람다 표현식은 인터페이스에 메소드가 하나인 인터페이스에만(Functional Interface) 사용할 수 있다.(default method 와 관련있음)
- 람다 표현식은 익명 클래스로 전환이 가능하고, 익명 클래스는 람다 표현식으로 전환 가능하다.
- 대표적인 Functional Interface 인 Runnable 인터페이스를 사용하는 예시이다.

```java
Runnable run = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};
```  

- 이를 람다 표현식으로 작성하면 아래와 같다.

```java
Runnable run2 = () -> {
    System.out.println("Hello");
};
```  

- 일반적인 Collection 을 정렬 코드는 아래와 같다.

```java
ArrayList<My> myList = new ArrayList<>();

Collections.sort(myList, new Comparator<My>() {
    @Override
    public int compare(My o1, My o2) {
        return o1.getValue() - o2.getValue();
    }
});
```  

- 람다 표현식을 사용하면 아래와 같다.

```java
ArrayList<My> myList = new ArrayList<>();

myList.sort((My m1, My m2) -> {
    return m1.getValue() - m2.getValue();
});


Collections.sort(myList, (My m1, My m2) -> {
    return m1.getValue() - m2.getValue();
});
```  

- 위의 예시와 같이 람다 표현식을 툥해 더욱 간결하고 직관적인 코드로 동일한 로직을 작성할 수 있다.
- 위에서 언급했던 것과 같이 람다식은 **Functional Interface** 라는 메서드가 하나뿐인 함수형 인터페이스에만 사용할 수 있다.

### Stream API
- Java8 은 람다 표현식의 도입과 함께 람다식을 효율적으로 사용할 수 있도록 기존 API 에 람다식을 적용했다.
- 대표적인 API 가 Stream API 이다.
- Stream 은 Collection 을 다루는 새로운 방법이다.
- Collection 을 파이프식으로 처리하도록 하면서 고차함수로 구조를 추상화 한다.
- 지연 연산, 병렬 처리 등이 인터페이스로 제공된다.
- 이런 Stream API 와 같은 방식을 함수형이라고 일컫는다.
- Stream API 을 사용하지 않을 때는 아래와 같은 코드로 Collection 원소에 특정 조건일 때의 합을 구할 수 있다.

```java
int sum = 0;
for(My m : myList) {
	if(m.getValue() < 5) {
		sum += m.getValue();
	}
}
```  

- Stream API 를 사용한 예시는 아래와 같다.

```java
int sum = myList.stream().filter(m -> m.getValue() < 5).mapToInt(My::getValue).sum();
```  

- 아래와 같이 병렬 처리도 가능하다.

```java
int sum = myList.parallelStream().filter(m -> m.getValue() < 5).mapToInt(My::getValue).sum();
```  

### Method Reference
- Method Reference 는 이미 이름이 있는 메소드를 대상으로 한 람다식의 간략형을 뜻한다.
- 메소드 참조를 나타내는 예약어로 `::` 를 사용한다.
- 메소드 참조의 예와 대응하는 람다식은 아래와 같다.

```java
String::valueOf
x -> String.valueOf(x)

Object::toString
x -> x.toString()

x::toString
() -> x.toString()

ArrayList::new
() -> new ArrayList<>();
```  

### 인터페이스의 Default Method
- Java 에서 Interface 에 새로운 메서드가 추가되면 하위 호환성이 깨지기 때문에 해당 인터페이스를 구현하는 모든 클래스에서 해당 메서드를 구현해야 한다.
- Java8 에서는 Default Method 라는 개념이 추가되어 하위 호환성 문제 없이 Interface 에 새로운 메서드를 추가할 수 있다.
- Default Method 란 Interface 에 구현된 메서드로 Interface 를 구현하는 클래스가 Override 하지 않을 경우 기본 구현체로 적용 된다.

### 배열 병렬 처리
- Java8 에서는 배열 처리 관련 정적 메서드가 있는 Arrays 클래스에 벙렬 처리 메서드들이 추가 되었다.
- 여러 배열을 정렬 등의 처리를 해야 할때, 성능적으로 항샹을 기대할 수 있다.
- 일반적으로 배열을 정렬하는 코드는 아래와 같다.

```java
Arrays.sort(numArray);
```  

- 병렬 처리로 배열을 정렬하면 아래와 같다.

```java
Arrays.parallelSort(numArray);
```  

### 새로운 날짜 API
- Java 에서 날짜 관련 연산을 위해서는 Date, Calendar 클래스를 사용해야 하지만 많은 문제 점과 불편함을 가지고 있었다.
- 이러한 문제점들로 인해 JodaTime 이라는 별도의 라이브러리를 많이 사용했었다.
- Java8 에서는 JodaTime 라이브러리 참고한 새로운 java.time 이라는 패키지가 추가되었다.
- java.time 패키지에는 Instant, LocalDate, LocalDateTime, ZonedDate 등이 추가 되었다.

### StringJoiner
- Java8 부터 문자열 관련 클래스는 StringJoiner 이 추가되어 아래와 같다.
	- String
	- StringBuilder
	- StringBuffer
	- Formatter
	- StringJoiner
- StringJoiner 는 순차적으로 나열되는 문자열 사이에 특정 문자열을 넣어줘야 할때 사용한다.

### Reflection 및 Annotation 개선


### Nashorn 자바스크립트 엔진
- 자바의 기본 자바스크립트 엔진은 모질라의 Rhino 였다.
- 시간이 지나며 최산의 자바 개선 사항을 활용하지 못하는 문제가 있어, Rhino 의 후속작인 Nashorn 엔진을 도입하게 되었다.
- 성능, 메모리관리 등이 개선 되었다.

### Collections API 확장
- Interface 가 Default Method 를 가질 수 있게 되면서 Java8 부터 Collections API 에 다양한 메소드가 추가 되었다.
- Interface 는 모두 Default Method 가 구현되었으며 새롭게 추가된 메소드는 아래와 같은 것들이 있다.
	- Iterable.forEach(Comsunmer)
	- Collection.remove(Predicate)
	- List.sort(Comparator)
	- Map.merge(K, V BiFunction)
	- .....

### Concurrency API 확장









---
## Reference
[Java 8 개선 사항 관련 글 모음](https://blog.fupfin.com/?p=27)   
[Java8 8가지 특징](http://pigbrain.github.io/java/2016/04/04/Java8_on_Java)   
[Java 8에 추가된 것들](https://medium.com/inhyuck/java-8%EC%97%90-%EC%B6%94%EA%B0%80%EB%90%9C-%EA%B2%83%EB%93%A4-8c66023cbbae)   
[JAVA8 변경 사항](http://tcpschool.com/java/java_intro_java8)   
[자바 8 살펴보기](http://www.moreagile.net/2014/04/AllAboutJava8.html)   

