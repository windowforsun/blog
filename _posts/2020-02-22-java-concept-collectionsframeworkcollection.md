--- 
layout: single
classes: wide
title: "[Java 개념] Collections Framework - Collection"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Collections Framework 중 Collection 부분에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Collection
    - Java Collection Framework
toc: true
use_math: true
---  

# Java Collection 

![그림 1]({{site.baseurl}}/img/java/concept_collectionsframeworkcollection_1.png)

- Java 에서 Collection 은 다수의 데이터를 쉽고 효과적으로 처리 할 수 있는 표준화된 방법을 제공하는 클래스의 집합을 의미한다.
- 데이터 집합을 다양한 자료구조를 통해 효과적으로 처리할 수 있도록 처리하는 알고리즘을 구조화해 각 클래스에 구현해 놓았다.
- 위 그림은 Java Collections Framework 중 Collection 부분의 구조를 표현한 간략한 다이어그램이다.
- Java Collection 을 이루는 모든 클래스들은 `Iterable` 인터페이스를 구현한다.
- `Iterable` 의 하위 인터페이스는 보다 구체적인 자료구조의 종류에 따라 나눠져 있다.
- 인터페이스의 하위에는 자료구조 종류에 따라 추상클래스에서 전체적인 알고리즘이나, 공통적인 부분을 구현한다.
- 인터페이스나, 추상 클래스의 하위에는 각 클래스들이 실제로 해당 자료구조에 필요한 알고리즘을 구현하고 있다.

# Iterable
- `Iterable` 인터페이스를 구현하면 객체가 `for-each loop` 문에 사용이 가능하다.
- 앞서 설명한 것처럼 Collection 을 구성하는 모든 인터페이스와 클래스는 해당 인터페이스의 하위에 있다.
- `Iterable` 인터페이스는 Java Collection 에서만 쓰이는 인터페이스가 아닌 `for-each loop` 이 필요하면 해당 인터페이스를 구현하기 때문에 다양한 곳에서 사용된다.

# Collection 
- Java Collection 의 루트 인터페이스로, Collection 동작에 공통적으로 필요한 메소드가 정의돼 있다.
- 하위 인터페이스에서는 보다 구체적인 자료구조의 인터페이스를 제공한다. (List, Set, Queue)

## List
- `Collection` 의 하위 인터페이스로 순서(`sequence`)가 있는 데이터 구조의 인터페이스이다.
- 다수의 데이터에서 위치(`index`) 를 통해 삽입, 삭제, 검색 등을 수행할 수 있다.
- 중복을 허용한다.
- 하위 클래스에서는 각 다른 구조로 순서가 있는 데이터를 컨트롤하는 구현체가 있다.(LinkedList, ArrayList, Vector, Stack)
- 일부 하위 구현체에서는 `null` 값을 추가할 경우 예외가 발생한다.

### AbstractList
- `List` 를 구현하는 추상 클래스로 랜덤 접근, 인덱스 기반 접근이 가능한 데이터 구조의 구현체이다.
- 랜덤 접근이 가능한 데이터 구조 구현에 필요한 기본적인 구현체를 제공하는 추상 클래스이다.

#### ArrayList
- `AbatractList` 의 하위 클래스로 가변적인 배열(Array)의 구현체이다.
- 가변적인 배열인 만큼 내부적으로 고정 크기의 배열을 사용하고 이를 `ArrayList` 단에서 조정해주는 처리가 들어간다.
- 가변적인 배열은 `null` 값을 원소를 허용한다.
- 배열의 크기를 명시해서도 사용 가능하다.
- 동기화에 대한 처리가 돼있지 않으므로, 동기화 처리가 필요한 경우 `Vector` 를 사용한다.
- 배열의 크기가 다차면 자동으로 조정하고, 명시적으로 배열의 크기를 조정할 수도 있다.
- 랜덤 접근 동작의 비율이 많은 경우 사용하기 좋다.
- `size`, `isEmpty`, `get`, `set`, `iterator` 는 상수시간인 $O(1)$ 을 보장한다.
- `add` 의 경우 `amorized constant time` 으로 할당된 배열의 공간이 충분할 경우 $O(1)$, 충분하지 않을 경우 $O(n)$ 이 소요 된다.
	>- Amortized Constant time(분활 상환 상수 시간)
	>   - 동적 배열(ArrayList)에서 발생할 수 있는 시간 복잡도 이다.
	>   - 배열의 공간이 있을 떄 원소를 추가하면 $O(1)$ 이지만, 공간이 다찼다면 확장하고 복사하는데 $O(n)$ 이 걸린다.
	>   - 배열 크기를 늘릴 때는 고비용이 발생하지만, 이후에는 비용이 거의 들지 않으므로 이부분을 분산시키면 성능은 상수 시간과 비슷해 진다.
- 이외의 다른 메소드들은 $O(n)$ 의 시간복잡도를 갖는다.
- `ArrayList` 에서 `Iterator` 를 생성한 후에 기존 `ArrayList` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)

#### Vector
- `AbstractList` 의 하위 클래스로 가변적인 배열이면서 동기화에 대한 처리가 추가된 구현체이다.
- `Vector` 는 삽입, 삭제 연산에 따라 배열의 크기가 늘어나거나 줄어드는 방식으로 사용 중인 배열의 전체 크기를 최적화 한다.
- 가변적인 배열인 `ArrayList` 와 위의 내용을 제외하면 기능적 내용은 거의 동일하다.

#### Stack
- `Vector` 의 하위 클래스로 `LIFO` 동작의 구현체이다.
- `Vector` 에서 `LIFO` 동작을 위해 5가지 메소드를(`empty`, `peek`, `pop`, `push(e)`, `search(e)`) 확장 한다.
- `Deque` 인터페이스의 구현체인 `ArrayDequeue` 가 보다 완벽하고 일관된 기능을 제공한다.

### AbstractSequentialList
- `AbstractList` 의 하위 추상 클래스로, 순차 접근하는 데이터 구조의 구현체이다.
- 순차 접근 데이터 구조 구현을에 필요한 기본적인 구현체를 제공하는 추상 클래스이다.

#### LinkedList
- `AbstractSequentialList` 와 `Deque` 의 하위 클래스로 이중 연결 리스트(Linked List)의 구현체이다.
- `Deque` 를 구현하고 있기 때문에, 이준 연결 리스트에서 양 끝쪽에 대한 삽입, 삭제, 조회 연산이 가능하다.
- 이중 연결 리스트는 `null` 값을 원소로 허용한다.
- 동기화에 대한 처리가 돼있지 않다.
- 순차적인 동작에 대해 최적화 돼있다.
- `getFirst`, `getLast`, `removeFirst`, `removeLast`, `offer`, `offerFirst`, `offerLast`, `pop`, `popFirst`, `popLast`, `peek`, `peekFirst`, `peekLast`, `size` 등과 같이 순서에 따른 메소드는 상수시간 $O(1)$ 의 시간복잡도를 갖는다.
- `add(i,e)`, `contains(e)`, `indexOf(e)`, `get(i)`, `remove(i)`, `remove(e)` 와 같은 순서 혹은 특정 원소에 해당하는 메소는 선형시간 $O(n)$ 의 시간복잡도가 소요된다.
- `LinkedList` 에서 `Iterator` 를 생성한 후에 기존 `LinkedList` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)


## Queue
- `Collection` 의 하위 인터페이스로 데이터의 삽입, 삭제에서 항상 순서가 유지되는 데이터 구조의 인터페이스이다.
- `Collection` 인터페이스에서 제공하는 삽입, 삭제, 검색 외에 별도의 기능을 제공한다.
- 기본적으로 `FIFO` 방식으로 요소를 컨트롤이 가능하고, 하위 구현체를 사용할 경우 `LIFO` 도 가능하다.
- 삽입의 경우 `tail`(배열의 마지막) 에 추가되고, 삭제는 `head`(배열의 처음) 에서 수행된다.
- `Queue` 인터페이스에서 직접 제공하는 데이터 조작 기능은 예외가 발생하지 않고 특정 값을 반환한다.

	.|예외 발생|특정값 반환
	---|---|---
	추가|add(e)|offer(e)
	삭제|remove()|poll()
	조회|element()|peek()

- 동기화가 필요한 데이터의 경우 `BlockingQueue` 를 사용한다.
- 일부 하위 구현체에서는 `null` 값을 추가할 경우 예외가 발생한다.

### AbstractQueue
- `Queue` 를 구현하는 추상 클래스로 Queue 데이터 구조의 구현체이다.
- Queue 데이터 구조 구현에 필요한 기본적인 구현체를 제공하는 추상 클래스이다.

### PriorityQueue
- `AbstractQueue` 의 하위 클래스로 우선순위 큐(Priority Heap)의 구현체이다.
- 우선순위는 `Comparator` 에 의해 순서대로 정렬된다.
- 우선순위 큐는 `null` 원소를 허용하지 않는다.
- `PriorityQueue` 의 `head` 는 우선순위 중 가장 가장 적은 원소를 가리킨다.
- `PriorityQueue` 는 내부적으로 배열을 사용하고, 배열의 크기는 자동으로 조정된다.
- 동기화에 대한 처리가 돼있지 않기 때문에, 동기화 처리가 필요한 경우 `PriorityBlockingQueue` 를 사용해야 한다.
- 각 메소드가 가지는 시간 복잡도는 아래와 같다.

	메소드명|시간복잡도
	---|---
	offer(e)|$O(\log_n)$
	poll()|$O(\log_n)$
	remove()|$O(\log_n)$
	add(e)|$O(\log_n)$
	remove(e)|$O(n)$
	contains(e)|$O(n)$
	peek()|$O(1)$
	element()|$O(1)$
	size()|$O(1)$	
	
## Deque
- `Queue` 인터페이스의 하위 인터페이스로 배열의 양쪽 끝에서 데이터 추가, 삭제가 가능한 데이터 구조의 인터페이스이다.
- `double ended queue` 의 약자이고 `deck` 이라고 불리기도 한다.
- 용량에 제한이 없지만, 필요한 경우 제한도 가능하다.
- 인덱스를 기반으로 데이터를 조작하는 연산은 지원하지 않는다.
- `Queue` 의 메소드에서 양쪽 끝의 동작에 대한 아래와 같은 메소드를 제공한다.
	- 첫 번째 요소(head)
		
		.|예외 발생|특정값 반환
		---|---|---
		추가|addFirst(e)|offerFirst(e)
		삭제|removeFirst()|pollFirst()
		조회|getFirst()|peekFirst()
		
	- 마지막 요소(tail)
		
		.|예외 발생|특정값 반환
		---|---|---
		추가|addLast(e)|offerLast(e)
		삭제|removeLast()|pollLast()
		조회|getLast()|peekLast()

- `Queue` 에서 확장된 `Deque` 를 `Queue` 와 같이 `FIFO` 에 대응되는 메소드는 아래와 같다.

	Queue|Deque
	---|---
	add(e)|addLast(e)
	offer(e)|offerLast(e)
	remove()|removeFirst()
	poll()|pollFirst()
	element()|getFirst()
	peek()|peekFirst()
	
- `Stack` 과 같은 `LIFO` 에 대응되는 메소드는 아래와 같다.

	Stack|Dequeue
	---|---
	push(e)|addFirst(e)
	pop()|removeFirst()
	peek()|peekFirst()

#### ArrayDeque
- `Dequeue` 를 구현한 클래스로, 배열을 사용한 `deck`(Linked List) 의 구현체이다.
- 내부에서 사용한 배열의 크기는 자동으로 조정 된다.
- 동기화에 대한 처리가 돼있지 않다.
- `null` 원소를 허용하지 않는다.
- 스택 자료구조처럼 사용될때는 `Stack` 구현체 클래스 보다 빠르고, 큐 자료구조로 사용될 때는 `LinkedList` 구현체 클래스보다 빠른 성능을 보인다.
- `ArrayDeque` 에서 제공하는 메소드들은 대부분 `Amortized Constant time` 임을 보장한다. 제외 되는 메소드는 removeFirstOccurrence, removeLastOccurrence, contains, iterator, remove 들이 있고, 대부분의 모든 메소드들도 선형시간을 보장한다.
	>- Amortized Constant time(분활 상환 상수 시간)
	>   - 동적 배열(ArrayList)에서 발생할 수 있는 시간 복잡도 이다.
	>   - 배열의 공간이 있을 떄 원소를 추가하면 $O(1)$ 이지만, 공간이 다찼다면 확장하고 복사하는데 $O(n)$ 이 걸린다.
	>   - 배열 크기를 늘릴 때는 고비용이 발생하지만, 이후에는 비용이 거의 들지 않으므로 이부분을 분산시키면 성능은 상수 시간과 비슷해 진다.
- `ArrayDeque` 에서 `Iterator` 를 생성한 후에 기존 `ArrayDeque` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)
	
## Set
- `Collection` 의 하위 인터페이스로 중복을 허용하지 않는 데이터 구조의 인터페이스이다.
- 수학의 집합을 추상화해 모델링한것이다
- 중복에 대한 판별은 `e1.eqauls(e2)` 를 사용한다.
- 일부 하위 구현체에서는 `null` 값을 추가할 경우 예외가 발생한다.

### AbstractSet
- `Set` 을 구현하는 추상클래스로 집합 데이터 구조의 구현체이다.
- 집합 데이터 구조 구현에 필요한 기본적인 구현체를 제공하는 추상 클래스이다.

#### EnumSet
- `AbstractSet` 의 하위 클래스로 Enum 을 사용하는 집합의 구현체이다.
- `EnumSet` 은 하나의 각 원소를 비트 벡터(Bit Vector)로 표현해서 저장하고 이런 저장방식은 성능적으로나 공간적으로 매우 효율적이다.
- `null` 원소는 허용하지 않는다.
- `Iterator` 로 순차 접근을 할때 순서는 사용하는 Enum 상수값에 의존한다.
- 동기화에 대한 처리는 돼있지 않다.
- 제공하는 모든 메소드는 상수시간 $O(1)$ 에 처리된다.
- `HashSet` 구현체 보다 빠를 수도 있다.(보장은 하지 않음)

#### HashSet
- `AbstractSet` 의 하위 클래스로 해시 테이블(Hash Table)을 사용하는 집합의 구현체이다.
- `Iterator` 로 순차 접근할 할때 순서는 보장되지 않는다.
- `null` 원소를 허용한다.
- 제공하는 모든 메소드는 해시 함수의 값이 해시 테이블에 잘 분배된다면 상수시간 $O(1)$ 안에 처리된다. (add, remove, contains, size)
- 해시 함수의 값이 잘 분배되기 위해서는 `capacity` 값이 중요한데, 너무 크지도 너무 작지도 않게 설정해야 계속해서 해시 함수를 돌리며 빈 버킷을 찾는 일을 줄일 수 있다.
- 동기화에 대한 처리는 돼있지 않다.
- `HashSet` 에서 `Iterator` 를 생성한 후에 기존 `HashSet` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)

#### LinkedHashSet
- `HashSet` 의 하위 클래스로 순서를 예측가능한 집합의 구현체이다.(Hash Table + Linked List)
- `HashSet` 과 다른점은 집합의 구조에서 원소간 이중 링크를 유지한다는 점에 있다. 이 링크를 통해 순차 접근시에 순서를 보장한다.
- 순서는 집합에 원소를 추가한 순서로 결정된다.
- `HashSet` 의 성능은 `capacity` 에만 의존했다면, `LinkedHashSet` 은 `capacity` 와 `loadfactor` 두 가지에 의존한다.
- 순서를 유저하는데 사용되는 목록이 추가되고, 이를 관리하는 비용이 추가되어 `HashSet` 와 비교해 약간의 성능저하는 발생 할 수 있다.
- 위의 특징외에는 `HashSet` 과 동일한 스펙을 가지고 있다.

## SortedSet
- `Set` 인터페이스의 하위 인터페이스로 중복이 없는 정렬된 데이터 구조의 인터페이스이다.
- `Comparator` 으로 정렬 기준의 변경할 수 있다.
- 기본적으로 오름차순으로 데이터를 순회한다.
- `SortedSet` 에 추가되는 데이터(객체)는 모두 `Comparable` 을 구현해야 한다.

## NavigableSet
- `SortedSet` 인터페이스의 하위 인터페이스로 정렬이라는 부분을 보다 확장한 데이터 구조의 인터페이스이다.
- `SortedSet` 은 오름차순으로 조회, 순회만 가능 했다면, 정렬 기준에서 오름차순, 내림차순으로 조회, 순회가 가능하다.
- 조회 하려는 원소와 값이 완전히 같지 않더라도, 가장 인접한 원소를 조회할 수 있다.
- 범위 검색을 통해 범위에 해당하는 `SortedSet` 을 검색 할 수 있다.

#### TreeSet
- `AbstractSet` 과 `NavigableSet` 의 하위 클래스로 임의의 정렬 집합의 구현체이다.(Red-black Tree)
- `Comparable` 의 기본 정렬 기준으로 사용이 가능하고, `Comparator` 를 통해 특정 정렬 기준을 정의할 수도 있다.
- `TreeSet` 에서 제공하는 기본 메소드(add, remove, contains)는 $O(\log_n)$ 의 시간복잡도를 보장한다.
- 동기화에 대한 처리는 돼있지 않다.
- `TreeSet` 에서 `Iterator` 를 생성한 후에 기존 `TreeSet` 를 삭제 및 변경하게 될경우 에러가 발생한다. (`fast-fail`)(`Iterator` 의 `remove` 메소드는 제외된다)


---
## Reference
[Hierarchy For Package java.util](https://docs.oracle.com/javase/8/docs/api/java/util/package-tree.html)  
[Java Collections – Performance (Time Complexity)](http://infotechgems.blogspot.com/2011/11/java-collections-performance-time.html)  
