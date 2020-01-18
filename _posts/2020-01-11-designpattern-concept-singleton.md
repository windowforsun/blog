--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Singleton Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '인스턴스가 단 1개만 생성되는 것을 보장하는 Singleton 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Singleton
---  

## Singleton 패턴이란
- 하나의 프로그램이 실행될 때 수 많은 인스턴스가 생성되는데, 프로그램에서 하나의 인스턴스만 존재해야 하는 경우가 있다. 
	- 게임을 구현한다면 게임 전체를 관리하는 `GameManager` 의 인스턴스는 단 하나만 존재해야 한다.
	- 서버를 구현한다면 서버의 설정, 상태를 관리하는 인스턴스는 단 하나만 존재해야 한다.
- `Singleton` 패턴은 하나의 프로그램(메모리를 공유하는 프로세스)에서 인스턴스가 단 1개만 존재하는 것을 보장해주는 패턴이다.


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_singleton_1.png)

- 패턴의 구성요소
	- `instance` 필드 : `static` 필드로 인스턴스가 단 하나임을 보장한다.
	- `Singleton` 생성자 : `private` 접근제어자를 통해 외부에서 생성자 호출을 막아, 외부 생성을 못하게 한다.
	- `getInstance()` : `Singleton` 클래스의 인스턴스를 생성 혹은 가져올 수 있는 메서드이다.
	
## 인스턴스가 단 1개인 클래스 만들기

### Singleton

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_singleton_1.png)

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```  

- `Singleton` 은 인스턴스가 단 1개만 보장하는 내용이 정의된 클래스이다.
- `instance` 필드는 `static` 키워드와 `private` 접근 제어자를 통해 인스턴스의 유일성과 외부 접근 제한을 보장한다.
- `Singleton()` 생성자는 `private` 접근제어자를 통해 외부 접근을 막아, 외부에서 생성자를 통해 `Singleton` 클래스의 인스턴스를 생성하는 것을 막는다.
- `getInstance()` 은 `public` 접근제어자로 외부에서 `Singleton` 의 인스턴스가 필요할 때 호출 하는 메서드이다.
	- `instance` 필드가 `null` 일 경우 즉, 아직 인스턴스가 생성되지 않았을 때만 인스턴스를 생성한다.
	- 이미 인스턴스가 생성된 경우에는 그 인스턴스를 리턴한다.

### 테스트

```java
public class SingletonTest {
    @Test
    public void getInstance() {
        // given
        Singleton singleton1 = Singleton.getInstance();
        Singleton singleton2 = Singleton.getInstance();

        // when
        boolean actual = singleton1.hashCode() == singleton2.hashCode();

        // then
        assertThat(actual, is(true));
    }
}
```  

## Multi Thread 인 상황에서 인스턴스가 단 1개인 클래스 만들기
- 현재 `Singleton` 클래스는 Multi Thread 의 환경에서 인스턴스의 유일성을 보장하지 못한다.(Thread-Safe 하지 못하다.)
	- 1번, 2번 쓰레드가에서 동시에 `getInstance()` 메서드를 호출한다.
	- `instance` 필드가 `null` 인상태에서 두 쓰레드 모두 `if` 안으로 들어가게 되면 `Singleton` 클래스의 인스턴스는 2개가 생성된다.

### Lazy Initialization

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {

    }

    public static synchronized Singleton getInstance() {
        if(instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```  

- `getInstance()` 메서드에 Java 의 동기화 키워드인 `syncchronized` 를 사용해서 Thread-Safe 를 보장하는 방식이다.
- `synchronized` 키워드 특성상 성능저하가 발생될 수 있다.
	- `getInstance()` 메서드가 호출 될때 마다 `Lock` 이 걸린다.

### Lazy Initialization + Double checked locking

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if(instance == null) {
            synchronized (Singleton.class) {
                if(instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```  

- `Lazy Initialization` 의 방법과 동일하게 `synchronized` 키워드를 사용했지만, 보다 성능적으로 유리한 방법이다.
- `getInstance()` 가 호출 때마다 `Lock` 이 걸리지 않게 하기 위해, `instance` 필드가 `null` 일 경우(처음 생성할 때만) `Lock` 을 걸어 성능과, Thread-Safe 를 보장한다.


### Initialization-on-demand holder idiom

```java
public class Singleton {
    private Singleton() {

    }

    public static class LazyHolder {
        public static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```  

- `Lazy Hodler` 방식이라고도 불린다.
- 중첩 클래스(`Lazy Holder`) 을 이용한 방법으로, `getInstance()` 가 호출 되기 전까지 인스턴스를 생성되지 않고, 메서드 안에 `LazyHolder` 클래스가 참조 될때 인스턴스가 생생된다. 또한 `final` 키워드를 통해 단 한번만 인스턴스 값을 할당한다.
- JVM 의 클래스의 초기화 가정에서 보장되는 원자적 특성을 이용해서 Thread-Safe 를 보장하는 방식이다.
- `synchronized` 키워드도 사용하지 않기 때문에 성능저하에 대한 이슈도 없다.


### Enum

```java
public enum Singleton {
    INSTANCE;

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```  

- Java 의 `enum` 을 활용한 방식으로, Thread-Safe 와 Serialization 을 보장해준다.
- 언어적 특성을 활용했기 때문에 Reflection 에도 안전하다.


---
## Reference

	