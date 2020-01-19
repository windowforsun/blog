--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Prototype Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '클래스로 인스턴스를 만들지 않고, 인스턴스에서 새로운 인스턴스를 만드는 Prototype 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Prototype
---  


## Prototype 패턴이란
- 일반적으로 인스턴스를 생성할때 `new SomeClass()` 와 같은 식으로 클래스 이름을 지정해서 생성해야 한다.(Java 의 경우)
- 아래는 클래스 이름을 바탕으로 인스턴스를 생성하지 않고, 인스턴스를 복사해 새로운 인스턴스를 만들어야 하는 경우 이다.
	- 아주 많은 종류가 있어 클래스로 정의가 힘든 경우
		- 반지름이 1인 원, 반지름이 2인원, 반지름이 3인 원 ...
	- 인스턴스 생성에 큰 비용이 들때
		- 인스턴스 생성을 할때 많은 요청의 응답을 사용하는 경우
	- 인스턴스 생성이 복잡할 경우
		- 사용자 인터페이스를 통해 그린 원을 복사해서 원 인스턴스를 만들어야 하는 경우
	- 같은 속성을 가진 객체를 빈번하게 복사해야 할 경우
- 여기서 인스턴스를 참조하지 않고 복사해서 사용해야 하는 이유는, `Prototype` 에서 생성하는 인스턴스들은 속성이 같더라도 독립적으로 다른 인스턴스로 구성이 필요하기 때문이다.
	- `a` 라는 인스턴스를 복사해서 `b` 라는 인스턴스를 만들었는데, `b` 의 필드 값을 변경한다고해서 `a` 인스턴스의 필드 값도 같이 변경돼면 안된다.
- `Prototype` 패턴은 원형, 모범 이라는 의미를 가진 것처럼, 만들어야 하는 인스턴스의 원형, 모범이 되는 인스턴스를 기반으로 인스턴스를 만드는 패턴이다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_prototype_1.png)

- 패턴의 구성요소
	- `Prototype` : 인스턴스를 복사해서 새로운 인스턴스를 만들기 위한 메소드가 정의된 인터페이스이다. (추상 클래스도 가능하다)
	- `ConcretePrototype` : `Prototype` 을 구현 또는 상속하는 클래스로, 인스턴스를 만드는 메소드를 실제로 구현한다.
	- `Client` : `Prototype` 의 인스턴스를 만드는 메서드를 사용해서 새로운 인스턴스를 만드는 클래스이다.
- `Prototype` 패턴은 인스턴스의 생성을 `framework`(`prototype`)과, 실제 생성하는(`ConcretePrototype`) 부분을 분리해서 구성한다.
	- 이후 예제에서 확인 할 수 있겠지만, 인스턴스를 생성하기 위한 메소드가 선언된 `Prototype` 은 클래스 이름을 사용하지 않고 문자열을 통해 인스턴스를 생성한다.
	- 위 특징을 통해 OOP 의 목표중 하나인 부품으로써 재사용이라는 점을 실현 할 수 있다.
	- 클래스 안에 어떤 클래스의 이름이 있다는 것은 강한 의존성을 가지고 있다는 의미로, 분리해서 재사용하기가 힘들어 진다.

## 여러 크기의 도형을 나타내는 인스턴스 만들기
- `Prototype` 패턴을 사용해서 여러가지 도형과 크기가 각 다른 도형을 만들어본다.

### java.lang.Cloneable
- 앞서 설명한 것과 같이 `Prototype` 패턴에서는 `new` 키워드를 통해 인스턴스를 생성하지 않고, 만들어진 인스턴스를 복사하는데 복사할 때 Java 의 `Cloneable` 인스터스을 통해 이를 구현한다.
- 인스턴스를 복사하는 `Cloneable` 인터페이스는 `java.lang.Cloneable` 로 기본으로 Java 에서 기본으로 포함된 인터페이스이다.
- 인스턴스의 복사는 `java.lang.Object` 에 있는 `clone()` 메소드를 사용한다.
- `Cloneable` 인터페이스를 구현하는 이유는 `clone()` 메서드는 `Cloneable` 인터페이스를 명시적으로 구현한 클래스에서만 사용가능 하기 때문이다.
- `Cloneable` 인터페이스에는 어떤 메소드도 선언돼 있지 않은데, 이는 `clone()` 메소드에 의해 복사 될 수 있다 라는 것을 명시하는 용도로 쓰이는 인터페이스이고, 이런 인터페이스를 `Maker interface` 라고 부른다.
- `clone()` 메소드는 기본으로 얕은 복사(shallow copy) 를 수행하는 메소드이다.
- `clone()` 메소드로 깊은 복사(deep copy) 를 수행하기 위해서는 직접 Override 해서 구현해야 한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_prototype_2.png)

### Figure

```java
public interface Figure extends Cloneable{
    int getArea();
    int getPerimeter();
    Figure createClone();
}
```  

- `Figure` 은 생성하는 인스턴스인 도형을 만들기 위한 메소드가 정의된 인터페이스 이다.
- `Prototype` 패턴에서 `Prototype` 역할을 수행한다.
- 객체의 인스턴스 복사하기 `clone()` 메소드로 복사하기 위해 `Cloneable` 인터페이스를 구현한다.
- 도형을 넓이를 구하는 `getArea()` 메소드, 도형의 둘레를 구하는 `getPerimeter()` 와 만들어진 객체를 복사하는 `createClone()` 메소드를 구성돼 있다.
- `Figure` 인터페이스는 인스턴스를 복사하는 `Manager` 클래스와 도형을 실제로 구현한 하위 클래스들간의 다리 역할을 수행한다.

### Manager

```java
public class Manager {
    private Map<String, Figure> figureMap = new HashMap<>();

    public void register(String name, Figure figure) {
        this.figureMap.put(name, figure);
    }

    public Figure create(String name) {
        Figure figure = this.figureMap.get(name);

        return figure.createClone();
    }
}
```  

- `Manager` 는 `Figure` 인터페이스를 사용해서 인스턴스의 복사를 수행하는 클래스이다.
- `Prototype` 패턴에서 `Client` 역할을 수행한다.
- `figureMap` 필드에 복사할 이름과 인스턴스로 저장한다.
- `register()` 메소드는 인자값으로 전달받은 이름과 인스턴스를 `figureMap` 에 저장한다.
- `create()` 메소드는 `figureMap` 에 만들어져 있는 인스턴스 중 이름과 매칭되는 인스턴스를 리턴한다.
- `Manager` 클래스는 실제 도형을 구현한 하위 클래스들의 이름은 사용하지 않고, 도형의 나타내는 인터페이스로만 구현해서 실제 도형들과의 의존성을 없애서 독립적인 수정과 확장이 가능하다.


### Circle

```java
public class Circle implements Figure {
    private int radius;

    public Circle(int radius) {
        this.radius = radius;
    }

    @Override
    public int getArea() {
        return (int)(this.radius * this.radius * Math.PI);
    }

    @Override
    public int getPerimeter() {
        return (int)(this.radius * 2 * Math.PI);
    }

    @Override
    public Figure createClone() {
        Figure figure = null;

        try {
            figure = (Figure)this.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }

        return figure;
    }
}
```  

- `Circle` 은 `Figure` 인터페이스를 구현하는 실제 도형인 원을 나타내는 클래스이다.
- `Prototype` 패턴에서 `ConcretePrototype` 역할을 수행한다.
- `radius` 필드는 원의 반지름을 의미한다.
- 원이라는 도형에 맞게 넓이는 구하는 `getArea()`, 둘레를 구하는 `getPerimeter()` 메소드를 구현하고 있다. 
- `createClone()` 메소드에서는 `clone()` 메소드를 사용해서 인스턴스를 복사하는데, `clone()` 메소드는 `CloneNotSupportedException` 이 발생할 수 있기 때문에 `try .. catch` 문으로 감싸고 있다.

### Rectangle

```java
public class Rectangle implements Figure {
    private int width;
    private int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() {
        return this.width * height;
    }

    @Override
    public int getPerimeter() {
        return (this.width * 2) + (this.height * 2);
    }

    @Override
    public Figure createClone() {
        Figure figure = null;

        try {
            figure = (Figure)this.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }

        return figure;
    }
}
```  

- `Rectangle` 은 `Figure` 인터페이스를 구현하는 실제 도형인 사각형을 나타내는 클래스이다.
- `Prototype` 패턴에서 `ConcretePrototype` 역할을 수행한다.
- `width`, `height` 필드는 사각형의 너비와 높이를 의미한다.
- 사각형이라는 도형에 맞게 넓이는 구하는 `getArea()`, 둘레를 구하는 `getPerimeter()` 메소드를 구현하고 있다. 
- `createClone()` 메소드에서는 `clone()` 메소드를 사용해서 인스턴스를 복사하는데, `clone()` 메소드는 `CloneNotSupportedException` 이 발생할 수 있기 때문에 `try .. catch` 문으로 감싸고 있다.

### 테스트

```java
public class PrototypeTest {
    @Test
    public void rectangle() {
        // given
        Manager manager = new Manager();
        Rectangle oneTwo = new Rectangle(1, 2);
        Rectangle threeFour = new Rectangle(3, 4);
        manager.register("oneTwo", oneTwo);
        manager.register("threeFour", threeFour);

        // when
        Figure actualOneTwo = manager.create("oneTwo");
        Figure actualThreeFour = manager.create("threeFour");

        // then
        assertThat(actualOneTwo.hashCode(), not(oneTwo.hashCode()));
        assertThat(actualOneTwo.getArea(), is(oneTwo.getArea()));
        assertThat(actualOneTwo.getPerimeter(), is(oneTwo.getPerimeter()));
        assertThat(actualThreeFour.hashCode(), not(threeFour.hashCode()));
        assertThat(actualThreeFour.getArea(), is(threeFour.getArea()));
        assertThat(actualThreeFour.getPerimeter(), is(threeFour.getPerimeter()));
    }

    @Test
    public void circle() {
        // given
        Manager manager = new Manager();
        Circle one = new Circle(1);
        Circle two = new Circle(2);
        manager.register("one", one);
        manager.register("two", two);

        // when
        Figure actualOne = manager.create("one");
        Figure actualTwo = manager.create("two");

        // then
        assertThat(actualOne.hashCode(), not(one.hashCode()));
        assertThat(actualOne.getArea(), is(one.getArea()));
        assertThat(actualOne.getPerimeter(), is(one.getPerimeter()));
        assertThat(actualTwo.hashCode(), not(two.hashCode()));
        assertThat(actualTwo.getArea(), is(two.getArea()));
        assertThat(actualTwo.getPerimeter(), is(two.getPerimeter()));
    }
}
```  

## 얕은 복사(shallow copy) 와 깊은 복사(deep copy)
- 앞서 `clone()` 메소드는 얕은 복사를 수행하는 메소드라고 했었다.
- `Circle`, `Rectangle` 클래스의 필드는 모두 원시(primitive) 타입이기 때문에 얕은 복사를 하더라도 문제가 발생하지 않는다.
- `ConcretePrototype` 클래스에 참조(reference) 타입의 필드가 있다면 기본으로 제공하는 `clone()` 메소드를 사용하면 참조로 인한 문제가 발생하게 된다.

### UnknownShallow

```java
public class UnknownShallow implements Figure {
    private List<Integer> lengthList;

    public UnknownShallow(List<Integer> lengthList) {
        this.lengthList = lengthList;
    }

    public List<Integer> getLengthList() {
        return lengthList;
    }

    @Override
    public int getArea() {
        return this.lengthList.stream().reduce(1, (a, b) -> a * b);
    }

    @Override
    public int getPerimeter() {
        return this.lengthList.stream().reduce(0, Integer::sum);
    }

    @Override
    public Figure createClone() {
        Figure figure = null;

        try {
            figure = (Figure)this.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }

        return figure;
    }
}
```  

- `UnknownShallow` 는 `Figure` 인터페이스를 상속 받고 참조 타입 필드를 가진 도형을 나타내는 클래스이다.
- `createClone()` 메소드에서는 기존 `Circle`, `Rectangle` 과 동일한 방식으로 인스턴스를 복사한다.

### UnknownDeep

```java
public class UnknownDeep implements Figure {
    private List<Integer> lengthList;

    public UnknownDeep(List<Integer> lengthList) {
        this.lengthList = lengthList;
    }

    public List<Integer> getLengthList() {
        return lengthList;
    }

    @Override
    public int getArea() {
        return this.lengthList.stream().reduce(1, (a, b) -> a * b);
    }

    @Override
    public int getPerimeter() {
        return this.lengthList.stream().reduce(0, Integer::sum);
    }

    @Override
    public Figure createClone() {
        List<Integer> list = new LinkedList<>();
        this.lengthList.forEach(list::add);

        return new UnknownDeep(list);
    }
}
```  

- `UnknownDeep` 은 `Figure` 인터페이스를 상속 받고 참조 타입 필드를 가진 도형을 나타내는 클래스이다.
- `createClone()` 메소드에서는 기존 `ConcretePrototype` 클래스들과 달리 직접 인스턴스의 필드를 복사해서 인스턴스를 복사하고 있다.

### 테스트

```java
public class PrototypeTest {
    @Test
    public void unknown_ShallowCopy() {
        // given
        Manager manager = new Manager();
        UnknownShallow one = new UnknownShallow(new ArrayList<>(Arrays.asList(1, 1, 1, 1)));
        UnknownShallow two = new UnknownShallow(new ArrayList<>(Arrays.asList(2, 2, 2, 2)));
        manager.register("one", one);
        manager.register("two", two);

        // when
        Figure actualOne = manager.create("one");
        Figure actualTwo = manager.create("two");
        one.getLengthList().add(100);
        two.getLengthList().add(200);

        // then
        assertThat(actualOne.hashCode(), not(one.hashCode()));
        assertThat(actualOne.getArea(), is(one.getArea()));
        assertThat(actualOne.getPerimeter(), is(one.getPerimeter()));
        assertThat(actualTwo.hashCode(), not(two.hashCode()));
        assertThat(actualTwo.getArea(), is(two.getArea()));
        assertThat(actualTwo.getPerimeter(), is(two.getPerimeter()));
    }

    @Test
    public void unknown_DeepCopy() {
        // given
        Manager manager = new Manager();
        UnknownDeep one = new UnknownDeep(new ArrayList<>(Arrays.asList(1, 1, 1, 1)));
        UnknownDeep two = new UnknownDeep(new ArrayList<>(Arrays.asList(2, 2, 2, 2)));
        manager.register("one", one);
        manager.register("two", two);

        // when
        Figure actualOne = manager.create("one");
        Figure actualTwo = manager.create("two");
        one.getLengthList().add(100);
        two.getLengthList().add(200);

        // then
        assertThat(actualOne.hashCode(), not(one.hashCode()));
        assertThat(actualOne.getArea(), not(one.getArea()));
        assertThat(actualOne.getPerimeter(), not(one.getPerimeter()));
        assertThat(actualTwo.hashCode(), not(two.hashCode()));
        assertThat(actualTwo.getArea(), not(two.getArea()));
        assertThat(actualTwo.getPerimeter(), not(two.getPerimeter()));
    }
}
```  

- `UnknonwShallow` 도형의 경우 복사 하는 인스턴스의 필드를 변경하게 될 경우, 복사한 인스턴스의 필드까지 변경되는 것을 확인 할 수 있다.
- `UnkownDeep` 도형의 경우 복사 하는 인스턴스의 필드를 변경하더라도, 복사한 인스턴스의 필드는 변경되지 않아 독립적으로 수정 및 사용이 가능한 것을 확인 할 수 있다.

---
## Reference

	