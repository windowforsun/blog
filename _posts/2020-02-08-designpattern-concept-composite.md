--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Composite Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '내용물을 포함하는 상자와 내용물을 동일시해서 재귀적인 구조로 다룰 수 있도록하는 Composite 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Composite
use_math : true
---  

## Composite 패턴이란
- `Composite` 는 합성, 혼합물 등의 의미를 가진 것과 같이 무언가를 포함해서 같은 종류로 다룰 수 있도록 하는 패턴을 뜻한다.
- 컴퓨터 파일 시스템의 `디렉토리` 는 아래와 같은 특징을 가지고 있다.
	- 디렉토리는 디렉토리를 포함할 수 있다.
	- 디렉토리는 파일을 포함할 수 있다.
- 디렉토리와 파일을 합쳐 `디렉토리 엔트리` 라고 불러 디렉토리 안에 들어갈 수 있는 것을 공통 지을 수 있다.
- 한 디렉토리의 용량을 알고 싶을 때, 해당 디렉토리에 포함된 디렉토리 엔트리들의 용량을 모두 합하면 된다.
- 디렉토리, 파일, 디렉토리 엔트리 처럼 내용물을 담는 상자와 내용물을 동일시하여 재귀적인 구조로 만드는 것을 `Composite` 패턴이라고 한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_composite_1.png)

- 패턴의 구성요소
	- `Leaf` : 내용물(파일) 역할을 하며, 내부에 다른 것을 넣을 수 없다.
	- `Composite` : 내용물을 포함하는 상자(디렉토리) 역할을 하며, 내부에 `Leaf`, `Composite` 를 넣을 수 있다.
	- `Component` : `Leaf` 와 `Composite` 를 동일시하는 역할과 두 클래스의 부모 클래스로 공통적은 부분에 대한 정의나 구현이 있다.
- 앞서 `Composite` 패턴은 내용물을 담는 상자와 내용물을 동일시하는 패턴이라고 했지만, 복수와 단수 관계 또한 동일시 할 수 있다.
	- API 테스트를 할때 요청 타입에 대한 테스트 1, 2, 3 을 묶어 API 타입 테스트라고 할 수 있다.
	- API 요청 값에 따른 로직적인 테스트 1, 2, 3 을 묶어 API 로직 테스트라고 할 수 있다.
	- API 요청에 따른 응답에 대한 테스트 1,2, 3 을 묶어 API 응답 테스트라고 할 수 있다.
	- 이런 API 에 대한 테스트를 모두 묶어 API 테스트라고 할 수 있다.
	
## 트리구조 만들기
- 트리는 노드를 통해 구성되고, 트리안에 트리가 있는 구조를 뜻한다.
- 트리는 아래와 같은 재귀적인 특징을 가지고 있다.
	- 트리는 트리를 포함할 수 있다.
	- 트리는 노드를 포함을 수 있다.
- 노드의 종류(노드 엔트리)는 자식을 가질 수 있느 노드(Node)와 자식을 가질수 없는 노드(Leaf)로 구분된다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_composite_2.png)

### Node

```java
public abstract class Entry {
    protected int num;

    public Entry(int num) {
        this.num = num;
    }

    public abstract int getCount();
    public abstract int getSum();

    public Entry add(Entry entry) {
        throw new RuntimeException("Need to override");
    }
}
```  

- `Entry` 클래스는 자식을 가질 수 있는 노드(Node)와 자식을 가질 수 없는 노드(Leaf) 를 공통으로 표현하는 추상 클래스이다.
- `Composite` 패턴에서 `Component` 역할을 수행한다.
- 모든 노드 엔트리는 `num` 이라는 숫자 값을 가진다.
- 각 하위 클래스에서는 포함된 노드 엔트리의 수를 반환하는 `getCount()` 와 포함된 노드 엔트리의 `num` 을 합을 반환하는 `getSum()` 의 추상 메소드를 구현해야 한다.
- `add()` 클래스는 노드 엔트리에 다른 노트 엔트리는 추가하는 메소드로 노드 엔트리에 따라 구현사항이 다를 수 있다.
	- 현재 작성된 코드와 같이 하위 클래스에서 Override 하지 않으면 강제적으로 예외를 발생시킨다.
	- 메소드에 대한 구현은 있지만, 아무 것도 실행하지 않는다.
	- 메소드에 대한 선언만 해서 하위 클래스에서 구현하도록 한다.
		- 불필요한 하위 클래스에서도 구현이 필요해 진다.
	- 상위 클래스의 스펙에 메소드 스펙을 포함시키지 않고, 필요한 하위 클래스에서 메소드를 따로 구현한다.
		- 상위 클래스의 타입으로 공통 지을 수 없다.

### Leaf

```java
public class Leaf extends Entry {
    public Leaf(int num) {
        super(num);
    }

    @Override
    public int getCount() {
        return 1;
    }

    @Override
    public int getSum() {
        return this.num;
    }
}
```  

- `Leaf` 는 자식을 가질 수 없는 노드를 표현하는 `Entry` 의 하위 클래스이다.
- `Composite` 패턴에서 `Leaf` 역할을 수행한다.
- 포함된 노드 엔트리가 자신 밖에 없기 때문에 `getCount()` 메소드에서는 1을 리턴하고, `getSum()` 메소드에서는 자신의 `num` 값을 리턴한다.

### Node

```java
public class Node extends Entry {
    private List<Entry> children;

    public Node(int num) {
        super(num);
        this.children = new LinkedList<>();
    }

    @Override
    public int getCount() {
        int result = 1;

        for(Entry entry : this.children) {
            result += entry.getCount();
        }

        return result;
    }

    @Override
    public int getSum() {
        int result = this.num;

        for(Entry entry : this.children) {
            result += entry.getSum();
        }

        return result;
    }

    @Override
    public Entry add(Entry entry) {
        this.children.add(entry);
        return this;
    }
}
```  

- `Node` 클래스는 자식을 포함하는 노드를 표현하는 `Entry` 의 하위 클래스이다.
- `Composite` 패턴에서 `Composite` 역할을 수행한다.
- `children` 필드에 자신의 노트 엔트리에서 포함하는 노드 엔트리를 저장한다.
- `add()` 메소드를 `Entry` 클래스에서 Override 해서 `children` 필드에 추가하도록 구현한다.
- 포함된 노드 엔트리(children) 을 순회하며 `getCount()` 와 `getSum()` 을 호출해서 자신의 값까지 포함해서 리턴한다.
- 자신에게 포함된 노드 엔트리의 `getCount()`, `getSum()` 을 구할때, 해당 노드 엔트리가 `Node` 인지 `Leaf` 인지 확인할 필요없이 해당하는 메소드만 호출해 주면된다.

### 테스트
- 테스트에서 사용할 트리의 구조는 아래 그림과 같다.

	![그림 1]({{site.baseurl}}/img/designpattern/2/concept_composite_2.png)

```java
public class CompositeTest {
    @Test
    public void nodeCountTest() {
        // given
        Node root = new Node(0);
        Node child1 = new Node(1);
        Node child2 = new Node(2);
        Node child3 = new Node(3);
        root.add(child1);
        root.add(child2);
        root.add(child3);

        Node child11 = new Node(11);
        Node child12 = new Node(12);
        child1.add(child11);
        child1.add(child12);

        Leaf leaf111 = new Leaf(111);
        child11.add(leaf111);

        Leaf leaf121 = new Leaf(121);
        child12.add(leaf121);

        Leaf leaf21 = new Leaf(21);
        child2.add(leaf21);

        Node child31 = new Node(31);
        child3.add(child31);

        Leaf leaf311 = new Leaf(311);
        Leaf leaf312 = new Leaf(312);
        child31.add(leaf311);
        child31.add(leaf312);

        // when
        int actual = root.getCount();

        // then
        assertThat(actual, is(12));
    }

    @Test
    public void nodeSumTest() {
        // given
        Node root = new Node(0);
        Node child1 = new Node(1);
        Node child2 = new Node(2);
        Node child3 = new Node(3);
        root.add(child1);
        root.add(child2);
        root.add(child3);

        Node child11 = new Node(11);
        Node child12 = new Node(12);
        child1.add(child11);
        child1.add(child12);

        Leaf leaf111 = new Leaf(111);
        child11.add(leaf111);

        Leaf leaf121 = new Leaf(121);
        child12.add(leaf121);

        Leaf leaf21 = new Leaf(21);
        child2.add(leaf21);

        Node child31 = new Node(31);
        child3.add(child31);

        Leaf leaf311 = new Leaf(311);
        Leaf leaf312 = new Leaf(312);
        child31.add(leaf311);
        child31.add(leaf312);

        // when
        int actual = root.getSum();

        // then
        assertThat(actual, is(0 + 1 + 2 + 3 + 11 + 12 + 21 + 31 + 111 + 121 + 311 + 312));
    }
}
```  

---
## Reference

	