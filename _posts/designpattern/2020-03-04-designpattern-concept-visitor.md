--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Visitor Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '데이터 구조와 구분되어 처리를 수행하는 Visitor 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Visitor
use_math : true
---  

## Visitor 패턴이란
- 방문자 라는 의미를 가진것과 같이, 구성된 데이터 구조와 분리된 상태에서 데이터 구조를 방문하며 특정 동작을 수행하는 패턴을 뜻한다.
- 데이터를 구성하는 클래스에 데이터 구조를 처리하는 로직을 넣는것이 아니라, 데이터의 구조와 처리를 분리한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_visitor_1.png)

- 패턴의 구성요소
	- `Visitor` : 데이터 구조를 방문하는 방문자 역할로, 데이터 구조의 구체적인 요소마다 대응하는 메소드를 정의한다.
	- `ConcreteVisitor` : `Visitor` 를 구현한 클래스로, 실제로 데이터 구조를 방문하며 처리할 내용으로 메소드를 구현한다.
	- `Element` : `Visitor` 가 방문 가능한 요소를 의미하는 역할로, `Visitor` 를 받아들이는(accept) 메소드를 선언한다.
	- `ConcreteElement` : `Element` 를 구현한 클래스로, 데이터를 구성하는 요소에 맞게 클래스를 구성한다.
	- `ObjectStructure` : `Element` 의 집합을 의마한다. (`Directory`, `Node`)
- 더블 디스패치(`double dispatch`) 를 통해 `Visitor` 패턴은 구성된다.
	- `ConcreteVisitor` 클래스는 `ConcreteElement` 를 `visit(ConcreteElement)` 메소드로 방문한다.
	- `ConcreteElement` 클래스는 `ConcreteVisitor` 를 `accept(Visitor)` 메소드로 받아들인다.
	- `ConcreteVisitor` 와 `ConcreteElement` 는 한쌍의 관계로 실제 처리가 구성되고, 이를 더블 디스패치라고 한다.
- The Open-Closed Principle(OCP) 원칙을 바탕으로 데이터의 구조와 처리를 분리해, 데이터의 구조와 별개로 데이터 처리부분에 대한 확장성을 높인다.
	- OCP 는 확장에는 열려있지만, 수정에 대해서는 닫혀 있는 원칙을 뜻한다.
	- 기존 클래스를 수정하지 않고 확장할 수있도록 하는 것을 뜻한다.
	- 분리가 돼있지 않다면, 처리부분이 수정 됐을 때 데이터의 구조 또한 수정해야 한다.
- `ConcreteVisitor` 의 확장에 대해서는 유연하다.
	- 새로운 처리 방식(`ConcreteVisitor`) 의 추가는 `ConcreteElement` 의 수정없이 새로운 `Visitor` 클래스만 추가해주면 된다.
- `ConcreteElement` 의 추가는 비교적 복잡하다.
	- 새로운 데이터 구조인 `ConcreteElementC` 가 추가 되면, `Visitor` 에 `visit(ConcreteElementC)` 를 하위 클래스도 포함해서 추가해야 한다.
- `Visitor` 패턴을 구성하기 위해서는 방문하는 데이터 구조의 원소들이 방문할 수 있는 충분한 정보를 공개해 주어야 한다.

## 트리구조 원소수 카운트하기
- [Composite 패턴]({{site.baseurl}}{% link _posts/designpattern/2020-02-08-designpattern-concept-composite.md %})
에서 사용했었던 트리 구조를 그대로 사용한다.
- 구성된 트리구조는 자식을 가질 수 있는 `Node` 와 자식을 가질수 없는 `Leaf` 로 구성된다.
- 트리 구조에서 `Visitor` 패턴은 이런 트리라는 구조를 구성하는 각 원소를 별도의 처리 방식으로 카운트하는 역할을 수행한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_visitor_2.png)

### Visitor

```java
public abstract class Visitor {
    public abstract int visit(Node node);
    public abstract int visit(Leaf leaf);
}
```  

- `Visitor` 는 방문자로 데이터 구조의 각 원소에 대한 처리 메소드가 선언돼 있는 추상클래스이다.
- `Visitor` 패턴에서 `Visitor` 역할을 수행한다.
- `visit()` 메소드는 오버로딩해서 데이터 구조의 원소의 종류만큼 선언돼 있다.

### Element

```java
public interface Element {
    int accept(Visitor visitor);
}
```  

- `Element` 는 방문자인 `Visitor` 를 받아들이는 메소드가 선언된 인터페이스이다.
- `Visitor` 패턴에서 `Element` 역할을 수행한다.
- `accept(Visitor)` 메소드를 하위 클래스에서 구현해 `Visitor` 를 받아들이는 처리를 구현한다.

### Entry

```java
public abstract class Entry implements Element {
    protected int num;

    public Entry(int num) {
        this.num = num;
    }

    public Entry add(Entry entry) {
        throw new RuntimeException("Need to Override");
    }

    public abstract int getCount();
    public abstract int getSum();
}
```  

- `Entry` 는 `Element` 의 하위 이면서 자식을 가질 수 있는 노드와 가질 수 없는 노드를 공통으로 표현하는 추상 클래스이다.
- `Composite` 패턴에서 구성된 것과 동일하고, `Element` 구현을 통해 `Entry` 로 구성되는 데이터 구조는 `Visitor` 의 방문이 가능하도록 했다.
- 모든 노드 엔트리는 `num` 이라는 숫자 값을 가진다.
- 각 하위 클래스에서는 포함된 노드 엔트리의 수를 반환하는 `getCount()` 와 포함된 노드 엔트리의 `num` 을 합을 반환하는 `getSum()` 의 추상 메소드를 구현해야 한다.
	
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

    @Override
    public int accept(Visitor visitor) {
        return visitor.visit(this);
    }
}

```  

- `Leaf` 는 자식을 가질 수 없는 노드를 표현하는 `Entry` 의 하위 클래스이다.
- `Visitor` 패턴에서 `ConcreteElement` 역할을 수행한다.
- `accept(Visitor)` 메소드를 통해 `Visitor` 의 `visit(Leaf)` 메소드를 호출해서 처리에 대한 부분을 위임한다.
	
### Node

```java
public class Node extends Entry {
    private List<Entry> children;

    public Node(int num) {
        super(num);
        this.children = new LinkedList<>();
    }

    public Iterator<Entry> getIterator() {
        return this.children.iterator();
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
    public int accept(Visitor visitor) {
        return visitor.visit(this);
    }

    @Override
    public Entry add(Entry entry) {
        this.children.add(entry);
        return this;
    }
}
```  

- `Node` 는 자식을 가질 수 있는 노드를 표현하는 `Entry` 의 하위 클래스이다.
- `Visitor` 패턴에서 `ConcreteElement` 와 `ObjectStructure` 역할을 수행한다.
- `getIterator()` 메소드를 통해 `Visitor` 가 `Node` 의 하위 클래스를 방문하며 처리 할 수 있도록 정보를 제공한다.
- `accept(Visitor)` 메소드를 통해 `Visitor` 의 `visit(Node)` 메소드를 호출해서 처리에 대한 부분을 위임한다.
	
### AllCountVisitor, NodeCountVisitor, LeafCountVisitor

```java
public class AllCountVisitor extends Visitor {
    @Override
    public int visit(Node node) {
        return node.getCount();
    }

    @Override
    public int visit(Leaf leaf) {
        return leaf.getCount();
    }
}
```  

```java
public class NodeCountVisitor extends Visitor {
    @Override
    public int visit(Node node) {
        int count = 1;
        Iterator<Entry> iterator = node.getIterator();
        Entry entry;

        while(iterator.hasNext()) {
            entry = iterator.next();
            count += entry.accept(this);
        }

        return count;
    }

    @Override
    public int visit(Leaf leaf) {
        return 0;
    }
}
```  

```java
public class LeafCountVisitor extends Visitor {
    @Override
    public int visit(Node node) {
        int count = 0;
        Iterator<Entry> iterator = node.getIterator();
        Entry entry;

        while(iterator.hasNext()) {
            entry = iterator.next();
            count += entry.accept(this);
        }

        return count;
    }

    @Override
    public int visit(Leaf leaf) {
        return leaf.getCount();
    }
}
```  

- `AllCountVisitor`, `NodeCountVisitor`, `LeafCountVisitor` 는 `Visitor` 의 구현체로 실제 처리구현 사항이 있는 클래스이다.
- `Visitor` 패턴에서 `ConcreteVisitor` 역할을 수행한다.
- `AllCountVisitor` 는 구성된 트리에서 모든 노드의 수를 카운트, `NodeCountVisitor` 는 자식을 가질 수 있는 노드만 카운트, `LeafCountVisitor` 는 자식을 가질 수 없는 노드만 카운트한다.
- 서로다른 3개의 클래스들은 각 처리에 맞게 `visit(Node)`, `visit(Leaf)` 메소드를 구현한다.
- `visit()` 메소드의 구현은 주로 `ConcreteElement` 에서 제공해주는 정보를 기반으로 처리된다.
- `ConcreteElement` 의 `accept()` 메소드는 `ConcreteVisitor` 의 `visit()` 메소드를 호출하고, 반대로 `ConcreteVisitor` 의 `visit()` 메소드는 `ConcreteElement` 의 `accept()` 메소드를 호출하는 상대가 상대를 호출하는 구조로 처리가 수행된다.
	
### Visitor 처리의 흐름
- 처리의 흐름은 `LeafCountVisitor` 를 통해 알아본다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_visitor_3.png)

1. `LeafCountVisitor` 인스턴스를 생성한다.
1. `Node` 인스턴스에 대해 `accept(Visitor)` 메소드를 `LeafCountVisitor` 인스턴스를 인자값으로 호출 한다.
1. `Node` 인스턴스는 인자값으로 전달 된 `LeafCountVisitor` 의 `visit(Node)` 메소드를 호출 한다.
1. `LeafCountVisitor` 인스턴스는 `Node` 의 자식을 순회하면서 첫번째  `Leaf` 인스턴스의  `accept(Visitor)` 메소드를 호출한다.
1. 첫번째 `Leaf` 인스턴스는 인자값으로 전달 된 `LeafCountVisitor` 의 `visit(Leaf)` 메소드를 호출 한다.
1. 첫번째 `Leaf` 인스턴스의 `visit(Leaf)` 와 `accept(Visit)` 가 반환되면, `LeafCountVisitor` 는 두번째 `Leaf` 인스턴스의 `accept(Visit)` 메소드를 호출 한다.
1. 두번째 `Leaf` 인스턴스도 첫번째와 동일하게 인자값으로 전달 된 `LeafCountVisitor` 의 `visit(Leaf)` 메소드를 호출 한다.
	
### 테스트
- 테스트에서 사용할 트리의 구조는 `Composite` 패턴과 동일하게 아래와 같다.

	![그림 1]({{site.baseurl}}/img/designpattern/2/concept_composite_3.png)
	
```java
public class VisitorTest {
    private Node root;

    @Before
    public void setUp() throws Exception {
        this.root = new Node(0);
        Node child1 = new Node(1);
        Node child2 = new Node(2);
        Node child3 = new Node(3);
        this.root.add(child1);
        this.root.add(child2);
        this.root.add(child3);

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
    }

    @Test
    public void AllCountVisitor() {
        // given
        Visitor visitor = new AllCountVisitor();

        // when
        int actual = this.root.accept(visitor);

        // then
        assertThat(actual, is(12));
    }

    @Test
    public void LeafCountVisitor() {
        // given
        Visitor visitor = new LeafCountVisitor();

        // when
        int actual = this.root.accept(visitor);

        // then
        assertThat(actual, is(5));
    }

    @Test
    public void NodeCountVisitor() {
        // given
        Visitor visitor = new NodeCountVisitor();

        // when
        int actual = this.root.accept(visitor);

        // then
        assertThat(actual, is(7));
    }
}
```  

	
	
---
## Reference

	