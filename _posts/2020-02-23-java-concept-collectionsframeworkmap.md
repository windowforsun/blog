--- 
layout: single
classes: wide
title: "[Java 개념] Collections Framework - Map"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Collections Framework 중 Map 부분에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Map
    - Java Collection Framework
toc: true
use_math: true
---  

# Java Map

![그림 1]({{site.baseurl}}/img/java/concept_collectionsframeworkmap_1.png)

- Java 에서 Map 은 `key-value` 형식으로 데이터를 저장하는 구조에서 효과적으로 처리할 수 있는 표준화된 방법을 제공하는 클래스의 집합니다.
- 위 그림은 Java Collections Framework 에서 Map 부분을 구성하고 있는 다이어그램이다.
- Map 을 구성하는 모든 인터페이스와 클래스는 Map 의 하위에 있다.
- Map 을 상속하는 하위 인터페이스들은 보다 구체적인 Map 구조의 인터페이스를 정의한다.
- 인터페이스를 구현하는 추상클래스들은 해당 Map 구조에서 공통적인 부분을 구현하거나, 보다 구체적으로 정의한다.
- 인터페이스 혹은 추상 클래스의 하위 클래스는 실질적인 Map 구조를 구현하는 클래스이다.
- 시간 복잡도에서 `n` 은 원소의 수, `h` 는 HashTable 의 용량을 의미한다.

# Map
- Java Map 의 루트 인터페이스로 Map 동작에 필요한 공통적인 메소드가 정의돼 있다.
- Map 구조에서 `key` 는 중복된 값을 가질 수 없고, 각 `key` 는 대응되는 `value` 를 갖는다.
- Java 의 Map 구조는 Dictionary 구조를 대체할 수 있다.
- Map 은 기본적으로 데이터 조회를 위해 세가지 방식의 Collection 을 제공한다.
	- `Set` 형식의 `key` 집합을 받아, `value` 를 조회한다.
	- `Collection` 형식의 `value` 를 받는다.
	- `Set` 형식의 `key-value` 구조를 받는다.
- `Collection` 의 순서는 `iterator` 에서 정의된 것을 따르고, 몇가지(`TreeMap`)는 별도의 순서를 가질 수 있다.
- Map 에서 `key`, `value` 에 대한 값 제한은 하위 구현체에 따라 상이 할 수 있다.
	- 어떤 구현체에서는 `null` 값을 허용하지 않는다.
	- 어떤 구현체에서는 `key` 타입에 제한이 있다.

## AbstractMap
- `Map` 을 구현하는 추상 클래스로 Map 데이터 구조의 구현체이다.
- Map 데이터 구조 구현에 필요한 기본적이면서 공통적인 구현체를 제공한다.

### EnumMap
- `AbstractMap` 의 하위 클래스로 `key` 가 `Enum` 형식인 Map 의 구현체이다.(Array)
- `key` 가 `Enum` 형식이기 때문에 다른 Map 구현체보다 `key` 에 있어서는 작고 효율적이다.
- `EnumMap` 은 내부적으로 배열을 통해 표현한다. (`key` 가 `Enum` 이기때문에 순서값이 존재)
- Map 에 있는 `key` 를 `Set` 구조로 가져오면 정렬은 원래 `Enum` 의 정렬(순서값)기준으로 정렬된다.
- `key` 에 `null` 은 허용하지 않는다.
- 동기화 처리가 돼있지 않다.
- `HashMap` 의 구현체보다 성능적 이점이 있을 수 있다.(보장하진 않음)
- 대부분은 메소드는 상수시간 $O(1)$ 의 시간복잡도를 갖는다.

메소드|시간 복잡도
---|---
get(key)|$O(1)$
containsKey(key)|$O(1)$
for-each next|$O(1)$

### HashMap
- `AbstractMap` 의 하위 클래스로 `key` 가 다양한 형태를 가질 수 있는 Map 의 구현체이다.(Hash Table)
- `key` 는 `Set` 구현체에 저장되기 때문에 정렬 순서는 정해진 기준이 없다.
- `key`, `value` 에 모두 `null` 값을 허용한다.
- `HashMap` 은 내부적으로 `key` 값의 저장을 `Set` 구현체를 사용하는 만큼 HashTable 에 의존한다.
- 내부적으로 사용하는 `HashTable` 의 버킷의 수가 $bucketCount * loadfactor$ 보다 같거나 클 때, `rehash` 를 통해 다시 구축하는데 기존의 2배의 공간을 할당한다.
- 크기를 예측할 수 있다면 적절한 크기를 설정해 주는 것이 효과적이다. 초기 크기를 너무 크게 설정하면 반복자와 같은 연산에서 느려질 수 있고, 너무 작게 설정하면 `rehash` 작업으로 인해 느려 질 수 있기 때문이다.
- 동기화 처리가 돼있지 않다.
- `HashMap` 에서 `Iterator` 를 생성한 후에 기존 `HashMap` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)
- `add`, `remove`, `contains`, `size` 메소드는 상수시간 $O(1)$ 의 시간복잡도를 갖는다.
- 반복자인 `Iterator` 관련 동작의 경우 `HashMap` 을 구성하는 전체 원소의 수와 할당된 `capacity`(버킷 수) 를 더 한만큼의 시간이 필요하다.
- `n` 은 원소의 수, `h` 는 HashTable 의 용량을 의미힌다.

메소드|시간 복잡도
---|---
get(key)|$O(1)$
containsKey(key)|$O(1)$
for-each next|$O(h/n)$

#### LinkedHashMap
- `HashMap` 의 하위 클래스로 순서 예측이 가능한 Map 의 구현체이다.(Hash Table + Linked List)
- 내부적으론 `HashMap` 의 구조에서 이중 연결 리스크가 `key` 의 순서를 관리한다.(insertion-order)
- 기존에 존재하던 `key` 가 다시 삽입되는 경우에는 순서에 영향을 끼치지 않는다.
- `LinkedHashMap` 구현체는 LRU(least-recently-used) 캐시와 같은 동작을 구현하는데 적합한다.
- `null` 값을 허용한다.
- 메소드 시간 복잡도의 경우 대부분 `HashMap` 과 비슷하거나, 리스트 관리 부분으로 인해 약간의 비용이 추가 될 수 있다.
- 반복자인 `Iterator` 관련 동작의 경우, `LinkedListHashMap` 을 구성하는 전체 원소의 수만큼 소요된다.
- 동기화 처리가 돼있지 않다.
- `LinkedHashMap` 에서 `Iterator` 를 생성한 후에 기존 `LinkedHashMap` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)

메소드|시간 복잡도
---|---
get(key)|$O(1)$
containsKey(key)|$O(1)$
for-each next|$O(1)$

### IdentityHashMap
- `AbstractMap` 의 하위 클래스로 `key` 의 비교가 `reference` 방식인 Map 의 구현체이다.(Array)
- `key` 비교가 `reference` 라는 것은 기존 `HashSet` 의 비교처럼 `k1==nul ? k2==null : k1.equals(k2)` 와 같이 수행하는 것이 아니라, `k1==k2` 와 같이 `key` 의 `reference` 만 비교하는 것을 뜻한다.
- `IdentityHashMap` 은 프로그래밍 로직상 객체의 값은 같지만 다른 객체를 `Map` 구조로 표현해야 할때 사용할 수 있다.
- `key`, `value` 에 모두 `null` 값을 허용한다.
- `Iterator` 와 같은 반복자를 수행할때 순서는 예측 할 수 없다.
- 크기를 예측할 수 있다면 적절한 크기를 설정해 주는 것이 효과적이다. 초기 크기를 너무 크게 설정하면 반복자와 같은 연산에서 느려질 수 있고, 너무 작게 설정하면 `rehash` 작업으로 인해 느려 질 수 있기 때문이다.
- 동기화 처리가 돼있지 않다.
- `IdentityHashMap` 에서 `Iterator` 를 생성한 후에 기존 `IdentityHashMap` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)
- `n` 은 원소의 수, `h` 는 HashTable 의 용량을 의미힌다.

메소드|시간 복잡도
---|---
get(key)|$O(1)$
containsKey(key)|$O(1)$
for-each next|$O(h/n)$

### WeakHashMap
- `AbstractMap` 의 하위 클래스로 `key` 의 관리가 JVM GC 의 참조 관리 중, 약간 참조를 사용하는 Map 의 구현체이다.(Hash Table)

>- JVM GC 3가지 참조 유형
>	1. 강한 참조(Strong Reference)
>		- `Integer strongNum = 1;` 과 같은 일반적인 참조 유형이다.
>		- `java.lang.ref` 패키지를 사용하지 않는 참조 이다.
>		- `strongNum` 변수는 `1` 인 객체에 대한 강한 참조를 가진다.
>		- 이런 강한 참조는 GC 의 대상이 되지 않는다.
>	1. 부드러운 참조(Soft Reference)
>		- `java.lang.ref.SoftReference` 를 이용해서 참조를 생성할 수 있다.			
>		- `SoftReference<Integer> softNum = new SoftReference<Integer>(strongNum)`
>		- `softNum` 변수가 가리키는 참조가 없을 경우(== null) GC 대상이 된다.
>		- 메모리가 부족하지 않을 때는 위의 경우라도 GC 대상이 되지 않는다.
>	1. 약한 참조(Weak Reference)
>		- `java.lang.ref.WeakReference` 를 이용해서 참조를 생성할 수 있다.
>		- `WeakReference<Integer> weakNum = new WeakReference<Integer>(strongNum)`
>		- `weakNum` 번슈가 가리키는 참조가 없을 경우(== null) GC 대상이 된다.
>		- 가용 메모리가 충분하더라도, GC 대상이 된다.

- `key` 의 관리가 약한 참조로 이뤄지기 때문에, 사용되지 않는(== null)인 `key` 가 있을 경우 해당 `key-value` 는 삭제(GC)된다. 
- `WeakHashMap` 에서 `key` 에 대한 비교는 `==` 를 통해 이뤄진다. (레퍼런스 비교)
- 동기화 처리가 돼있지 않다.
- `WeakHashMap` 의 `key` 에 `new` 가 아닌 리터럴(literal) 방식으로 생성할 경우(`String key = "key1"`) 약한 참조의 방식으로 `key` 가 관리되지 못한다.
- `WeakHashMap` 에서 `value` 는 강한 참조로 관리된다. `value` 객체가 `key` 객체를 강한 참조 하지 않도록 주의해야 한다.
	- `value` 가 `key` 를 참조해야 한다면 `value` 또한 약한 참조로 생성해야 한다. (`weakMap.put(weakKey, new WeakReference(value))`)
- 기타 기본적인 메소드의 동작과 시간복잡도는 `HashMap` 과 유사하다.

메소드|시간 복잡도
---|---
get(key)|$O(1)$
containsKey(key)|$O(1)$
for-each next|$O(h/n)$

## SortedMap
- `Map` 의 하위 인터페이스로 특정 기준으로 `key` 를 정렬한 Map 구조의 인터페이스이다.
- `SortedMap` 에서 사용되는 `key` 는 `Comparable` 의 하위 클래스여야 한다.
- 정렬에 대한 기준은 `Comparable` 의 `compareTo` 메소드를 오버라이드 해서 구현한다.
- 정렬에 대한 기준은 `Comparator` 을 통해 변경 할 수 있다.

## NavigableMap
- `SortedMap` 의 하위 인터페이스로 정렬이라는 부분을 보다 확장한 데이터 구조의 인터페이스이다.
- `SortedMap` 이 `key` 를 특정 기준으로 정렬해서 오름차순 조회, 순회만 가능했다면, 추가적으로 오름차순, 내림차순으로 조회, 순회가 가능하다.
- 조회하려면 `key` 와 같지 않더라도, 가장 인접한 `key` 를 조회 할 수 있다.
- `key` 에 대한 범위 검색을 통해 범위에 해당하는 `SortedMap` 을  검색 할 수 있다.

### TreeMap
- `AbstractMap`, `NavigableMap` 의 하위 클래스로, 임의의 정렬기준으로 `key` 를 관리할 수 있는 Map 구조의 구현체이다.(Red-black Tree)
- `key` 에서 구현한 `Comparable` 을 기본 정렬 기준으로 사용하고, 필요에 따라 `Comparator` 를 별도로 구현해 특정 정렬 기준을 정의 할 수도 있다.
- 동기화에 대한 처리는 돼있지 않다.
- TreeMap 에서 Iterator 를 생성한 후에 기존 TreeMap 를 삭제 및 변경하게 될경우 에러가 발생한다. (fast-fail)(Iterator 의 remove 메소드는 제외된다)
- `containsKey`, `get`, `put`, `remove` 메소드에 대해서 $\log n$ 을 보장한다.


메소드|시간 복잡도
---|---
get(key)|$O(\log n)$
containsKey(key)|$O(\log n)$
for-each next|$O(\log n)$

---
## Reference
[Hierarchy For Package java.util](https://docs.oracle.com/javase/8/docs/api/java/util/package-tree.html)  
[Java Collections – Performance (Time Complexity)](http://infotechgems.blogspot.com/2011/11/java-collections-performance-time.html)  
[Java HashMap은 어떻게 동작하는가?](https://d2.naver.com/helloworld/831311)  
[Java – Collection – Map – WeakHashMap (약한 참조 해시맵)](http://blog.breakingthat.com/2018/08/26/java-collection-map-weakhashmap/)  
