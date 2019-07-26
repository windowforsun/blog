--- 
layout: single
classes: wide
title: "[Java 개념]] Java Generic(제네릭)"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Generic 의 개념에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Generic
---  

## Generic 이란
- 컴파일 시에 구체적인 타입이 결정되는 기능이다.
	- 강한 타입 체크를 할 수 있다.
- 타입 변환(Casting)을 제거할 수 있다.
- Java 5 부터 추가되었다.
- Collections, Lamda (함수적 인터페이스), NIO 등에 많이 사용된다.

## Generic Type
- 제네릭 타입은 타입을 매개변수로 가지는 클래스와 인터페이스를 뜻한다.

	```java
	public class 클래스 <T> {
		// ...
	}
	```  
	
	```java
	public interface 인터페이스 <T> {
		// ...
	}
	```  

- 아래와 같은 `Parent` 클래스가 있다.

	```java
	public class Parent {
	    private Object parentObject;
	
	    public Object getParentObject() {
	        return parentObject;
	    }
	
	    public void setParentObject(Object parentObject) {
	        this.parentObject = parentObject;
	    }
	}
	```  
	
	- `parentObject` 에 대한 타입을 `Object` 로 가지고 있다.
	- 이 경우 `parentObject` 를 사용하기 위해 계속해서 타입 변환을 해주어야 한다.
	
- 아래는 위 `Parent` 클래스에 제네릭을 적용한 `ParentG` 클래스 이다.

	```java
	public class ParentG <T> {
	    T parentObject;
	
	    public T getParentObject() {
	        return parentObject;
	    }
	
	    public void setParentObject(T parentObject) {
	        this.parentObject = parentObject;
	    }
	}
	```  
	
- `parentObject` 의 타입이 되는 클래스는 아래와 같다.

	```java
	public class InnerChildA {
	    private String innerChildA;
	
	    public String getInnerChildA() {
	        return innerChildA;
	    }
	
	    public void setInnerChildA(String innerChildA) {
	        this.innerChildA = innerChildA;
	    }
	}
	
	public class InnerChildB {
	    private String innerChildB;
	
	    public String getInnerChildB() {
	        return innerChildB;
	    }
	
	    public void setInnerChildB(String innerChildB) {
	        this.innerChildB = innerChildB;
	    }
	}
	```  
	
- 제네릭 타입의 테스트 코드는 아래와 같다.

	```java
	public class GenericTypeTest {
	    private InnerChildA innerChildA;
	    private InnerChildB innerChildB;
	
	    @Before
	    public void init() {
	        this.innerChildA = new InnerChildA();
	        this.innerChildA.setInnerChildA("innerChildA");
	
	        this.innerChildB = new InnerChildB();
	        this.innerChildB.setInnerChildB("innerChildB");
	    }
	
	    @Test
	    public void parent() {
	        Parent parentA = new Parent();
	        parentA.setParentObject(this.innerChildA);
	
	        // Casting
	        InnerChildA resultA = (InnerChildA)parentA.getParentObject();
	        assertThat(resultA.getInnerChildA(), is("innerChildA"));
	
	        Parent parentB = new Parent();
	        parentB.setParentObject(this.innerChildB);
	
	        // Casting
	        InnerChildB resultB = (InnerChildB)parentB.getParentObject();
	        assertThat(resultB.getInnerChildB(), is("innerChildB"));
	
	        Parent parentString = new Parent();
	        parentString.setParentObject(new String("String"));
	
	        // Casting
	        String resultString = (String)parentString.getParentObject();
	        assertThat(resultString, is("String"));
	    }
	
	    @Test
	    public void parentG() {
	        ParentG<InnerChildA> parentA = new ParentG<>();
	        parentA.setParentObject(this.innerChildA);
	
	        InnerChildA resultA = parentA.getParentObject();
	        assertThat(resultA.getInnerChildA(), is("innerChildA"));
	
	        ParentG<InnerChildB> parentB = new ParentG<>();
	        parentB.setParentObject(this.innerChildB);
	
	        InnerChildB resultB = parentB.getParentObject();
	        assertThat(resultB.getInnerChildB(), is("innerChildB"));
	
	        ParentG<String> parentString = new ParentG<>();
	        parentString.setParentObject(new String("String"));
	
	        String resultString = parentString.getParentObject();
	        assertThat(resultString, is("String"));
	    }
	}
	```  
	
## Generic Type Naming Conventions
- `T` 와 같은 `<T>` 의 문자는 사용자 정의에 따라 이름을 정해 사용할 수 있다.
- 일반적으로 사용되는 이름들은 아래와 같다.

	Name|Desc
	---|---
	E|요소(Java Collections)
	K|키
	N|숫자
	T|타입
	V|값
	S,U,V|2번째, 3번째, 4번째
	
## Multi Type Parameter
- 제네릭 타입은 두 개 이상으로 멀티 파라미터로 사용할 수 있다.
- `ParentG` 클래스를 아래와 같이 수정한다.

	```java
	public class ParentG<F,S> {
	    private F first;
	    private S second;
	
	    public F getFirst() {
	        return first;
	    }
	
	    public void setFirst(F first) {
	        this.first = first;
	    }
	
	    public S getSecond() {
	        return second;
	    }
	
	    public void setSecond(S second) {
	        this.second = second;
	    }
	}
	```  
	
- 멀티 타입 파라미터의 테스트 코드는 아래와 같다.

	```java
	public class MultiTypeParameterTest {
	    private InnerChildA innerChildA;
	    private InnerChildB innerChildB;
	
	    @Before
	    public void init() {
	        this.innerChildA = new InnerChildA();
	        this.innerChildA.setInnerChildA("innerChildA");
	
	        this.innerChildB = new InnerChildB();
	        this.innerChildB.setInnerChildB("innerChildB");
	    }
	
	    @Test
	    public void multiTypeParameter_1() {
	        ParentG<InnerChildA, InnerChildB> parent = new ParentG<>();
	        parent.setFirst(this.innerChildA);
	        parent.setSecond(this.innerChildB);
	
	        InnerChildA first = parent.getFirst();
	        InnerChildB second = parent.getSecond();
	
	        assertThat(first.getInnerChildA(), is("innerChildA"));
	        assertThat(second.getInnerChildB(), is("innerChildB"));
	    }
	
	    @Test
	    public void multiTypeParameter_2() {
	        ParentG<InnerChildB, InnerChildA> parent = new ParentG<>();
	        parent.setFirst(this.innerChildB);
	        parent.setSecond(this.innerChildA);
	
	        InnerChildB first = parent.getFirst();
	        InnerChildA second = parent.getSecond();
	
	        assertThat(first.getInnerChildB(), is("innerChildB"));
	        assertThat(second.getInnerChildA(), is("innerChildA"));
	    }
	}
	```  
	
## Generic Method
- 제네릭 메소드는 매개타입과 리턴타입으로 타입 파라미터를 갖는 메소드를 의미한다.
		
	```java
	public class GenericMethod {
	    public <T> T getString(T object) {
	        return object;
	    }
	
	    public <K, V> HashMap<K, V> makeHashMap(K key, V value) {
	        return new HashMap<K, V>();
	    }
	}
	```  
	
- 제네릭 메서드 테스트 코드는 아래와 같다.

```java
public class GenericMethodTest {
    @Test
    public void genericMethod() {
        GenericMethod genericMethod = new GenericMethod();

        assertThat(genericMethod.getObject(new InnerChildA()), instanceOf(InnerChildA.class));
        assertThat(genericMethod.getObject(new InnerChildB()), instanceOf(InnerChildB.class));
        assertThat(genericMethod.getObject(new Parent()), instanceOf(Parent.class));
    }
}
```  

## Bounded Type Parameters (제한된 타입 파라미터)
- 제네릭 타입에 지정된 타입 파라미터의 제한을 두는 것을 의미한다.
	
	```java	
	public class 클래스 <T extends 상위클래스또는인터페이스> {
		// ...
	}
	```  
	
	```java
	public interface 인터페이스 <T extends 상위클래스또는인터페이스> {
		// ...
	}
	```  
	
	```java	
	public <T extends 상위클래스또는인터페이스> void 메서드명(T t){
		// ...
	}
	```  
	
- `ParentG` 클래스를 아래와 같이 수정한다.

	```java
	public class ParentG <T extends InnerParent> {
	    T parentObject;
	
	    public T getParentObject() {
	        return parentObject;
	    }
	
	    public void setParentObject(T parentObject) {
	        this.parentObject = parentObject;
	    }
	}
	```  
	
	- `ParentG` 클래스의 타입 파라미터를 `InnerParent` 클래스를 상속받는 하위 클래스로 제한한다.
	
- `InnerParent` 와 하위 클래스는 아래와 같다.

	```java
	public class InnerParent {
	    private String parentName;
	
	    public String getParentName() {
	        return parentName;
	    }
	
	    public void setParentName(String parentName) {
	        this.parentName = parentName;
	    }
	}
	
	public class InnerChildA extends InnerParent {
	    private String innerChildA;
	
	    public String getInnerChildA() {
	        return innerChildA;
	    }
	
	    public void setInnerChildA(String innerChildA) {
	        this.innerChildA = innerChildA;
	    }
	}

	public class InnerChildB extends InnerParent {
	    private String innerChildB;
	
	    public String getInnerChildB() {
	        return innerChildB;
	    }
	
	    public void setInnerChildB(String innerChildB) {
	        this.innerChildB = innerChildB;
	    }
	}
	```  
	
- 제한된 타입 파라미터 테스트 코드는 아래와 같다.

	```java
	public class BoundedTypeParameterTest {
	    @Test
	    public void boundedTypeParameter() {
	        InnerChildA innerChildA = new InnerChildA();
	        innerChildA.setParentName("innerChildAParent");
	        innerChildA.setInnerChildA("innerChildA");
	
	        InnerChildB innerChildB = new InnerChildB();
	        innerChildB.setParentName("innerChildBParent");
	        innerChildB.setInnerChildB("innerChildB");
	
	        ParentG<InnerChildA> resultA = new ParentG<>();
	        resultA.setParentObject(innerChildA);
	
	        assertThat(resultA.getParentObject().getParentName(), is("innerChildAParent"));
	        assertThat(resultA.getParentObject().getInnerChildA(), is("innerChildA"));
	
	        ParentG<InnerChildB> resultB = new ParentG<>();
	        resultB.setParentObject(innerChildB);
	
	        assertThat(resultB.getParentObject().getParentName(), is("innerChildBParent"));
	        assertThat(resultB.getParentObject().getInnerChildB(), is("innerChildB"));
	
	        // String 클래스는 InnerParent 클래스의 하위 클래스가 아니기 때문에
	        // 타입 파라미터로 사용이 불가능하다.
	        // ParentG<String> resultString = new ParentG<>();
	    }
	}
	```  
	
## Multiple Bounds
- 타입 파라미터에 여러개의 제한을 둘 수도 있다.

	```java
	<T extends class1 & interface1 & interface2>
	```  
	
- 위와 같이 `Class`, `Interface` 를 모두 사용하여 제한을 둬야 할때 `Class`가 앞으로 와야 한다.

## Generic, Inheritance and Subtypes
- 제네릭의 상속은 아래와 같다.

	```java
	public class ParentG <T extends InnerParent> {
	    T parentObject;
	
	    public T getParentObject() {
	        return parentObject;
	    }
	
	    public void setParentObject(T parentObject) {
	        this.parentObject = parentObject;
	    }
	}
	
	public class ChildGA <T extends InnerParent> extends ParentG<T> {
	    private String childGA;
	
	    public String getChildGA() {
	        return childGA;
	    }
	
	    public void setChildGA(String childGA) {
	        this.childGA = childGA;
	    }
	}
	
	public class ChildGB <T extends InnerParent> extends ParentG<T>{
	    private String childGB;
	
	    public String getChildGB() {
	        return childGB;
	    }
	
	    public void setChildGB(String childGB) {
	        this.childGB = childGB;
	    }
	}
	```  
	
- 제네릭 관련 상속 성질에 대한 예외상황에는 아래와 같은 상황이 있다.

	```java	
	public class GenericInheritance {
	    @Test
	    public void genericInheritance_TypeParameter() {
	        ChildGA<InnerParent> childGA = new ChildGA<>();
	
	        childGA.setParentObject(new InnerChildA());
	        assertThat(childGA.getParentObject(), instanceOf(InnerChildA.class));
	
	        childGA.setParentObject(new InnerChildB());
	        assertThat(childGA.getParentObject(), instanceOf(InnerChildB.class));
	
	        assertThat(this.getObject(childGA), instanceOf(ChildGA.class));
	    }
	
	    @Test
	    public void genericInheritance_GenericClass() {
	        ChildGA<InnerChildA> childGAA = new ChildGA<>();
	        childGAA.setParentObject(new InnerChildA());
	
	        ChildGA<InnerChildB> childGAB = new ChildGA<>();
	        childGAB.setParentObject(new InnerChildB());
	
        	// 타입 파라미터의 Invariant(무공변) 의 성질로 인해 호출 할 수 없다.
	        // this.getObject(childGAA);
	        // this.getObject(childGAB);
	    }
	
	    public ChildGA<InnerParent> getObject(ChildGA<InnerParent> object) {
	        return object;
	    }
	}
	```  
	
	![그림 1]({{site.baseurl}}/img/java/concept-generic-1.gif)
	
- `Collections` 에서의 상속 관계는 아래와 같다.

	```java
	ArrayList<String> list = new ArrayList<String>();
	```  
	
	![그림 2]({{site.baseurl}}/img/java/concept-generic-2.gif)
	
- `multiple tpye parameters` 를 사용할 경우 아래와 같은 상속 구조도 가능하다.


	```java	
	interface PayloadList<E,P> extends List<E> {
	  void setPayload(int index, P val);
	}
	```  
	
	![그림 3]({{site.baseurl}}/img/java/concept-generic-3.gif)

## Wildcards
- 제네릭을 사용하기 위해서는 하나의 타입을 지정해야 하지만 와일드카드를 사용할 경우 `?` 을 사용해서 타입을 지정할 수 있다.
- 와일드카드 타입을 타입 파라미터 부분에 하나 이상의 타입을 지정할 수 있다.
- 타입 파라미터의 Invariant(무공변) 성질로 인해 아래의 코드는 동작할 수 없다.

	```java
	List<Integer> list = new ArrayList<>();
	// compile error
	do(list);
	
	public void do(List<Number> list) 
	```  
	
- 위 문제를 와일드카드 타입을 사용해서 해결하면 아래와 같다.

	```java	
	List<Integer> listInteger = new ArrayList<>();
	List<Double> listDouble = new ArrayList<>();
	do(listInteger);
	do(listDouble);
	
	public void do(List<? extends Number> list) 
	```  
	
## Upper, Lower Bounded Wildcards
- 와일드카드 타입은 Upper, Lower 로 타입의 제한을 둘 수 있다.
	- Upper Bounded : `<? extends T>`
	- Lower Bounded : `<? super T>`
	
- 이를 다시 표현하면 아래와 같다.

	name|Desc|Ex
	---|---|---
	invariant(무공변)|자기 타입만 허용한다.|<T>
	convariant(공변)|구체적인 방향으로 타입 변환을 허용 한다.(자신과 자식 객체)|<? extends T>
	contravariant(반공변)|추상적인 방향으로 타입 변환을 허용한다.(자신과 부모 객체)|<? super T>
	
- `extends-bound`, `super-bound` 는 `PECS` 라는 개념으로 설명 하겠다.
	- PECS : Procuder-extends, Consumer-super

	```java	
	public class WildCardsProducerConsumer {
	    public void listCopy(List<? extends InnerParent> src, List<? super InnerParent> dest) {
	        for(InnerParent innerParent : src) {
	            dest.add(innerParent);
	        }
	    }
	    
	    // compile error
	    public void listCopyCompileError(List<? super InnerParent> src, List<? extends InnerParent> dest) {
	        for(InnerParent innerParent : src) {
	            dest.add(innerParent);
	        }
	    }
	}
	```  
	
	- Producer-extends 는 무언가를 제공하는, Consumer-super 는 무언가를 사용한다.
	- `listCopy` 의 인자값으로 설명하면 아래와 같다.
		- src : Producer-extends
		- dest : Consumer-super
	- 바꿔 말하면, Producer-extends 는 In 의 개념, Consumer-super 는 Out 의 개념으로 사용할 수 있다.
	- 생산인 In 의 개념일 경우 `<? extebds T>` 사용하고, 사용인 Out 의 개념일 경우 `<? super T>` 를 사용한다.
	
---
## Reference
[Lesson: Generics (Updated)](https://docs.oracle.com/javase/tutorial/java/generics/index.html)  
[[ Java] Java의 Generics](https://medium.com/@joongwon/java-java%EC%9D%98-generics-604b562530b3)  
