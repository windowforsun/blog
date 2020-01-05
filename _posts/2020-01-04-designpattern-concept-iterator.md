--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Iterator Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '반복자 역할을 하는 Iterator 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - DesignPattern
tags:
  - DesignPattern
  - Iterator
---  

## Iterator 패턴이란
- 반복자라는 의미에서 유추할 수 있듯이, 여러개의 데이터에 대해서 순새대로 지정하면서 전체를 검색할 수 있도록 하는 패턴이다.
- 아래와 같은 전체 탐색 반복문에서 인덱스 역할을 하는 `i` 를 좀 더 추상화해서 일반화한 디자인 패턴이다.

	```java
	for(int i = 0; i < array.length; i++) {
		// do something by array[i]
	}
	```  


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_iterator_1.png)

- 패턴의 구성 요소
	- `Iterator` : 요소를 순서대로 검색하는 인터페이로 관련 메서드가 선언되어 있다.
	- `ConcreteIterator` : `Iterator` 인터페이스의 구현체로, 검색에 필요한 데이터를 통해 실제 검색 동작을 수행한다.
	- `Aggregate` : `Iterator` 인터페이스를 만들어내는 메서드가 선언되어 있는 인터페이스이다.
	- `ConcreteAggregate` : `Aggregate` 인터페이스의 구현체로, 검색하고자 하는 데이터들의 `ConcreteIterator` 를 만든다.

## 전화번호부 전체 검색
- 전화번호부(PhoneBook)에 등록된 모든 전화번호(Phone)를 탐색하는 프로그램이다.


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_iterator_2.png)

### Aggregate

```java
public interface Aggregate {
    Iterator iterator();
}
```  

- `Aggregate` 는 `모으다`, `종합하다` 라는 의미를 가진 것처럼 데이터의 집합체를 의미하는 인터페이스이다.
- `Aggregate` 에는 `iterator()` 라는 하나의 메서드만 선언되어 있고, 이 메서드에서는 탐색하고자 하는 집합의 `Iterator` 를 생성해서 반환하는 구현체가 된다.


### Iterator

```java
public interface Iterator {
    boolean hasNext();
    Object next();
}
```  

- `Iterator` 는 다량의 데이터에서 하나씩 나열하면서 루프 변수(i)와 같은 역할을 수행하는 인터페이스이다.
- `Iterator` 에는 아래와 같은 2개의 메서드가 선언되어 있다.
	- `next()` : 다음 데이터를 반환하고, 내부적으로 사용하는 루프 변수(i)와 같은 것을 반환한 다음 데이터의 값으로 변경한다.
	- `hasNext()` : 다음 데이터가 존재하는 지 검사하는 메서드이다. `next()` 호출 전에 다음 데이터가 있는지 검사할 떄 주로 사용된다.

### Phone

```java
@Getter
@Setter
@Builder
@ToString
public class Phone {
    private String name;
    private String phoneNumber;
}
```  

- `Phone` 은 하나의 전화번호의 정보를 담는 클래스이다.
- `Lombok` 을 사용해서 `Getter`, `Setter` 등을 구현했다.

### PhoneBook

```java
public class PhoneBook implements Aggregate {
    private Phone[] phones;
    private int lastIndex;
    private int maxSize;

    public PhoneBook(int maxSize) {
        this.maxSize = maxSize;
        this.lastIndex = 0;
        this.phones = new Phone[this.maxSize];
    }

    public Phone getByIndex(int index) {
        return this.phones[index];
    }

    public void addPhone(Phone phone) {
        this.phones[this.lastIndex++] = phone;
    }

    public int getLastIndex() {
        return this.lastIndex;
    }

    public Iterator iterator() {
        return new PhoneBookIterator(this);
    }
}
```  

- `PhoneBook` 은 여러 전화번호(`Phone`) 을 담는 전화번호부를 나타내는 클래스이다.
- 여러 전호번화를 담기 때문에 집합체의 역할을 하기 위해 `Aggregate` 인터페이스를 구현하고 있다.
- `iterator()` 메서드에서는 `Iterator` 인터페이스의 구현체(`PhoneBookIterator`)를 생성해서 리턴하고 있는 것을 확인 할 수 있다.
- 전화번호 추가(`addPhone()`), 탐색(`getByIndex()`)에 필요한 메서드들이 구현되어 있다.

### PhoneBookIterator

```java
public class PhoneBookIterator implements Iterator {
    private PhoneBook phoneBook;
    private int index;

    public PhoneBookIterator(PhoneBook phoneBook) {
        this.phoneBook = phoneBook;
        this.index = 0;
    }

    public boolean hasNext() {
        boolean result = false;

        if(this.index < this.phoneBook.getLastIndex()) {
            result = true;
        }

        return result;
    }

    public Object next() {
        return this.phoneBook.getByIndex(this.index++);
    }
}
```  

- `PhoneBookIterator` 는 `PhoneBook` 을 전체 검색할 수 있도록 `Iterator` 인터페이스를 구현한 클래스 이다.
- 현재 검색 위치를 나타내는 `index` 필드를 사용해서 `hasNext()`, `next()` 메서드가 동작한다.
- `hasNext()` 메서드는 현재 전화번호부에 검색하지 않은 남은 데이터가 있는지 검사해서, 다음 데이터가 있는지 여부를 리턴한다.
- `next()` 는 현재 검색 위치의 데이터를 리턴하고, 검색 위치를 다음 위치로 변경시킨다.

### 테스트

```java
public class IteratorTest {
    @Test
    public void Iterator_One() {
        // given
        PhoneBook phoneBook = new PhoneBook(1);
        phoneBook.addPhone(Phone.builder()
                .name("a")
                .phoneNumber("01012341234")
                .build());

        // when
        Iterator it = phoneBook.iterator();
        List<Phone> actual = new LinkedList<>();
        while(it.hasNext()) {
            actual.add((Phone)it.next());
        }

        // then
        assertThat(actual, hasSize(1));
        assertThat(actual, hasItem(hasProperty("name", is("a"))));
        assertThat(actual, hasItem(hasProperty("phoneNumber", is("01012341234"))));
    }

    @Test
    public void Iterator_Two() {
        // given
        PhoneBook phoneBook = new PhoneBook(10);
        phoneBook.addPhone(Phone.builder()
                .name("a")
                .phoneNumber("01012341234")
                .build());
        phoneBook.addPhone(Phone.builder()
                .name("b")
                .phoneNumber("010123412345")
                .build());

        // when
        Iterator it = phoneBook.iterator();
        List<Phone> actual = new LinkedList<>();
        while(it.hasNext()) {
            actual.add((Phone)it.next());
        }

        // then
        assertThat(actual, hasSize(2));
        assertThat(actual, hasItem(hasProperty("name", is("a"))));
        assertThat(actual, hasItem(hasProperty("phoneNumber", is("01012341234"))));
        assertThat(actual, hasItem(hasProperty("name", is("b"))));
        assertThat(actual, hasItem(hasProperty("phoneNumber", is("010123412345"))));
    }
}
```  

---
## Reference
