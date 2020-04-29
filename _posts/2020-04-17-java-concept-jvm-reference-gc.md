--- 
layout: single
classes: wide
title: "[Java 개념] Reference 와 GC"
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
    - JVM
    - Strong Reference
    - Weak Reference
    - Soft Reference
    - Phantom Reference
    - Reachability
toc: true
use_math: true
---  

## java.lang.ref
- [GC]()
의 역할은 `Heap` 내의 객체중 `Garbage` 를 찾아서 메모리를 회수하는 역할을 수행하고, 이는 사용자 애플리케이션과는 독립된 영역이면서 역할이였다.
- `JDK 1.2` 부터는 애플리케이션에서 `java.lang.ref` 패키지로 `GC` 와 상호작용을 통해 어느정도 관여할 수 있게 되었다.
- `java.lang.ref` 패키지에는 사용자 애플리케이션에서 일반적으로 사용하는 `strong reference` 와 `soft`, `weak`, `phantom` 참조 방식을 클래스로 제공한다.
- 해당 패키지를 사용해서 `LRU(Least Recently Used)` 캐시와 같은 특별한 동작에 대해 더욱 쉽게 구현가능하다.


## Java Reference 의 Reachability

[runtime data area 참조 그림]

- `GC` 는 `Heap` 에 있는 객체 중 `Garbage` 를 판별하기 위해 `reachability` 라는 개념을 사용한다.
- 객체에 유효한 참조가 있으면 `reachable` 없으면, `unreachable` 로 판별한다.
- 객체에 유효한 참조가 없는 `unreachable` 객체를 `Garbage` 로 취급한다.
- `root set` 을 통해 `Heap` 내에서 객체간 참조 사슬로 구성된 상태에서 참조의 유/무효를 판별 한다.
- `Heap` 영역의 객체의 참조는 아래 4가지로 구분된다.
	- `Heap` 내의 다른 객체에 의한 참조
	- `Java Stack`(메소드 실행) 에서 사용하는 변수에 의한 참조
	- `Native Stack`(JNI) 에서 사용하는 객체에 의한 참조
	- `Method Area` 의 전적 변수에 의한 참조
	
[root set 참조 그림]
	
- 위 4가지 참조 중 `Heap` 내의 다른 객체에 의한 참조를 제외한 3가지가 `root set` 에 대한 참조로, `reachability` 의 판정 기준이 된다.
	- `root set` 에서 참조 사슬이 이어지는 객체를 `reachable` 객체라고 한다.
	- `root set` 에서 참조 사슬이 이어지지 않은 객체를 `unreachable` 객체라고 한다.

## Reference 클래스
- `java.lang.rev` 패키지에서는 `Reference` 의 하위 클래스로 `soft`, `weak`, `phantom` 레퍼런스를 사용할 수 있는 클래스를 제공한다.
	- `soft reference` : `SoftReference` 클래스
	- `weak reference` : `WeakReference` 클래스
	- `phantom reference` : `PhantomReference` 클래스
- `Reference` 클래스의 하위 클래스는 다른 객체와 달리 `GC` 에서 특별하게 관리한다.

### Reference 클래스에 의한 참조
- `Reference` 의 하위 클래스인 `WeakReference` 의 인스턴스는 아래와 같이 생성할 수 있다.

	```java
	// weak reference 생성
	WeakReference<SimpleClass> weak = new WeakReference<SimpleClass>(new Simple());
	// strong reference 생성
	Simple simple = weak.get();
	```  
	
	[root set weak reference, strong reference 참조 그림]
	
	- `weak` 은 참조는 `weak reference` 의 참조이고, `simple` 의 참조는 `strong reference` 의 참조이다.
	- `root set` 에는 `weak` 과 `simple` 이 존재한다.
	- 생성한 `Simple` 객체의 인스턴스는 `weak` 과 `simple` 에 의해 참조 된다.

- `strong reference` 인 `simple` 참조에 `null` 을 대입하면 아래와 같다.

	```java	
	// weak reference 생성
	WeakReference<SimpleClass> weak = new WeakReference<SimpleClass>(new Simple());
	// strong reference 생성
	Simple simple = weak.get();
	simple = null;
	```  
	
	[null 대입 그림]
	
	- `simple` 참조에 `null` 을 대입하게 되면 생성한 `Simple` 객체의 인스턴스는 `weak` 에 의해서만 참조 된다.
	- 위와 같이 `weak`(`WeakReference`) 에 대해서만 참조되는 객체를 `weakly reachable` 이라고 한다.
- `Reference` 의 하위 클래스를 통해 생성된 객체를 `reference object`(`weak`) 라고 한다. 이는 일반적으로 사용되고 있는(`simple` 참조) `strong reference` 의 참조와는 무관하게 사용되는 용어이다.
- `reference object`(`weak`) 에 의해 참조되는 객체를 `referent`(`Simple` 인스턴스) 라고 한다.

### Refernce 클래스의 Reachability
- `Reference` 클래스를 이용한 참조에 대한 `Reachability` 에 대해 알아본다.
- `WeakReference` 의 참조를 추가해서 `root set` 과 `Heap` 의 참조를 그리면 아래와 같다.

[root set, heap, weak reference 참조 그림]

- 그림에서 참조는 3가지 종류로 구성돼 있다.
	- `root set` 에 의한 참조되는 객체 (Strong Reachable)
	- `weak reference` 에 의한 참조되는 객체 (Weakly Reachable)
	- `root set` 으로 부터 참조되지 않은 객체 (Unreachable)
- 다음 `GC` 가 수행될때 `Garbage` 의 대상은 `Unreachable` 과 `Weakly reachable` 이 된다.
- `WeakReference` 객체의 경우 `root set` 에서 참조하고 있는 `Strong reachable` 이기 때문에 `Garbage` 의 대상이 되지 않는다.
- `WeakReference` 에 참조되면서 `root set` 에 참조되는 객체 또한 `Strong reachable` 이기 때문에 `Garbage` 대상이 되지 않는다.
- `GC` 에 의해 `Garbage` 대상이 되면 `WeakReference` 객체에서 참조하고 있는 `weakly reachable` 에 `null` 을 설정하고, 이후 메모리 회수 대상이 된다.

## Reachability 의 종류
- `java.lang.ref` 패키지의 `Reference` 클래스를 통해 다양한 방식의 참조를 생성할 수 있다.
- `GC` 가 수행되면 `root set` 을 시작으로 구성 된 참조 사슬을 탐색해서 `reference object` 를 검사하고, `reachability` 를 결정하게 된다.
- 여기서 결정되는 `reachability` 는 같은 5가지 종류가 있다.
- 객체가 생성되고 `GC` 를 통해 메모리가 해제되기까지 `Reachability` 에 의한 생명 주기는 아래와 같다.

	[reachability life cycle 그림]

[strengths reachability 그림]

### Strong reachable
- `root set` 으로 시작해서 참조 사슬에 `reference object` 가 존재하지 않는 객체를 의미한다.
- `strong reference` 로만 구성된 참조 사슬에 참조 되는 객체이다.

### Softly reachable
- `strong reachable` 객체를 제외하고 남은 객체 중, `soft reference` 가 하나라도 있는 참조 사슬에 참조되는 객체를 의미한다.

### Weakly reachable
- `strongly reachable`, `softly reachable` 객체를 제외하고 남은 객체 중, `weak reference` 가 하나라도 있는 참조 사슬에 참조되는 객체를 의미한다.

### Phantom reachable
- `strong reachable`, `softly reachable`, `weakly reachable` 객체를 제외하고 남은 객체 중, `phantom reference` 가 하나라도 있는 참조 사슬에 참조되거나 `finalize()` 되었지만 아직 메모리 회수가 되지 않은 객체를 의미한다.

### Unreachable
- `root set` 으로 부터 어떠한 참조 사슬에도 참조되지 않은 객체를 의미한다.

## ReferenceQueue
- `ReferenceQueue` 는 `java.lang.ref` 패키지에서 제공하는 클래스이다.
- `ReferenceQueue` 는 `Reference` 하위 클래스의 생성자를 통해 설정해서 사용할 수 있다.
- `Reference` 하위 클래스의 객체가 참조하는 객체가 `GC` 대상이 되면, 해당 참조는 `null` 로 설정되고 `ReferenceQueue` 에 주입된다.
- `ReferenceQueue` 에 주입되는 동작은 `GC` 에서 자동으로 수행한다.
- 이후 `ReferenceQueue` 를 통해 `GC` 대상이 된 객체를 `poll()`, `remove()` 메소드를 통해 가져와 확인하거나 후처리 작업을 수행 할 수 있다.
- 대표적으로 `WeakHashMap` 이 `ReferenceQueue` 와 `WeakReference` 를 사용해서 구현되었다.

```java
ReferenceQueue<Object> rQueue = new ReferenceQueue<Object>();
SoftReference<Object> weak = new SoftReference<Object>(new Object(), rQueue);
WeakReference<Object> weak = new WeakReference<Object>(new Object(), rQueue);
PhantomReference<Object> weak = new PhantomReference<Object>(new Object(), rQueue);
```  
	
## StrongReference(strongly reachable)
	
## SoftReference(softly reachable)

## WeakReference(weakly reachable)

## PhantomReference(phantom reachable)

## LRU 구현











































---
## Reference
[Java Reference와 GC](https://d2.naver.com/helloworld/329631)  
[Package java.lang.ref](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/compact2-package-summary.html)  
[Strong, Soft, Weak and Phantom References (Java)](http://neverfear.org/blog/view/150/Strong_Soft_Weak_and_Phantom_References_Java)  
[Weak, Soft, and Phantom References in Java (and Why They Matter)](https://dzone.com/articles/weak-soft-and-phantom-references-in-java-and-why-they-matter)  
[Difference between WeakReference vs SoftReference vs PhantomReference vs Strong reference in Java](https://javarevisited.blogspot.com/2014/03/difference-between-weakreference-vs-softreference-phantom-strong-reference-java.html)  
[Java Garbage Collection - Understanding Phantom Reference with examples](https://www.logicbig.com/tutorials/core-java-tutorial/gc/phantom-reference.html)  
[Class Object finalize](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Object.html#finalize())  