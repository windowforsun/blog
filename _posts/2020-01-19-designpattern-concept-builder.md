--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Builder Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '부품과 조립 과정을 통해 인스턴스를 만드는 Builder 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Builder
---  

## Builder 패턴이란
- `Builder` 패턴은 이름과 같이, 건축물을 세우는 것처럼 인스턴스를 만들어가는 방식을 패턴화 한 것이다.
- 하나의 인스턴스를 만들때 일련의 과정이 있고 그 과정에 따라 복잡하게 만들어 진다고 했을 때, 이런 인스턴스를 한번에 만드는 것을 복잡하고 어려울 수 있다.
- 이런 인스턴스를 만들 때 복잡한 인스턴스의 생성을 부품화 하고, 이 부품들을 모아 하나의 인스턴스를 만드는 것이 `Builder` 패턴이다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_builder_1.png)

- 패턴의 구성요소
	- `Builder` : 인스턴스를 생성할 떄 필요한 부품을 만드는 메소드들이 선언된 인터페이스이다.
	- `ConcreteBuilder` : `Builder` 를 실제로 구현하는 클래스로 각 부품을 만드는 메소드를 실제로 구현하고 부품들을 통해 만들어진 인스턴스를 반환하는 메소드도 선언돼 있다.
	- `Director` : `Builder` 인터페이스를 사용해서 인스턴스를 생성하는 클래스이다. `ConcreteBuilder` 에 의존하지 않고, `Builder` 인터페이스만으로 인스턴스 생성을 수행한다.
- `Builder` 패턴이 인스턴스를 만드는 과정을 시퀀스 다이어그램으로 그리면 아래와 같다.

	![그림 1]({{site.baseurl}}/img/designpattern/2/concept_builder_2.png)

- `Builder` 패턴은 인스턴스를 생성에 필요한 부품을 정의하는 부분(`Builder`)과 부품을 조립해서 인스턴스를 만드는(`Director`)로 나눠 구성돼 있다.
	- `Director` 클래스는 `Builder` 인터페이스의 하위 클래스는 알지 못하지만 `Builder` 인터페이스 만으로 인스턴스를 조립한다.
	- 이런 구조적 특징으로 `Director` 클래스는 `Builder` 인터페이스만 구현한다면, 어떠한 인스턴스도 만들 수 있다.
	- 이는 `Director` 클래스가 `Builder` 인터페이스의 하위 클래스들과 의존성이 없기 때문에 독립적으로 확장 가능한 구조가 될 수 있다.
- `Builder` 패턴에서 소개하는 예제는 아래와 같은 2가지로 분류된다.
	- GoF 에서 설명하는 방식의 예제
	- Java 에서 보편적으로 사용되는 방식의 예제

## 컴퓨터 조립하기
- 먼저 소개할 예제는 GoF 에서 설명하는 방식의 `Builder` 패턴의 예제이다.
- 컴퓨터 엔지니어가 컴퓨터의 부품들을 사용해서 컴퓨터를 조립하는 것을 `Builder` 패턴으로 구성한다.
- 컴퓨터 엔지니어는 컴퓨터 조립에 필요한 부품들만 알고 있고, 부품들을 이용해서 컴퓨터를 조립만 수행한다.
- 조립하는 컴퓨터의 종류는 PC 와 슈퍼컴퓨터가 있다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_builder_3.png)

### Computer

```java
public class Computer {
    protected float coreClock;
    protected int coreCount;
    protected int ramSize;
    protected int ramCount;
    protected int ssdSize;
    protected int ssdCount;
    protected int hddSize;
    protected int hddCount;

	// getter, setter
}
```  

- `Computer` 는 컴퓨터를 나태내는 클래스이다.
- 패턴의 구성요소에는 나와있지 않지만, `Builder` 인터페이스는 `Computer` 객체에 필요한 부품을 정의하고 `ConcreateBuilder` 클래스는 `Computer` 객체에 필요한 부품으로 실제로 구성한다.
- `Computer` 를 만들기 위한 부품들은 필드로 정의돼 있다.

### ComputerBuilder

```java
public abstract class ComputerBuilder {
    protected Computer computer;

    public ComputerBuilder() {
        this.computer = new Computer();
    }

    public abstract void buildCpu();
    public abstract void buildRam();
    public abstract void buildSsd();
    public abstract void buildHdd();
    public Computer getComputer() {
        return this.computer;
    }
}
```  

- `ComputerBuilder` 는 컴퓨터에 필요한 부품들을 만드는 메소드들이 선언돼있는 추상클래스이다.
- `Builder` 패턴에서 `Builder` 역할을 수행한다.
- 필드에는 컴퓨터를 나태내는 `Computer` 객체가 있고, 생성자에서 인스턴스를 생성한다.
- 컴퓨터를 만들기 위해 `buildCpu()`, `buildRam()`, `buildSdd()`, `buildHdd()` 메소드를 사용한다.
- `getComputer()` 메소드는 만들어진 `Computer` 의 인스턴스를 리턴한다.

### ComputerEngineer

```java
public class ComputerEngineer {
    private ComputerBuilder builder;

    public ComputerEngineer(ComputerBuilder builder) {
        this.builder = builder;
    }

    public void construct() {
        this.builder.buildCpu();
        this.builder.buildRam();
        this.builder.buildSsd();
        this.builder.buildHdd();
    }
}
```  

- `ComputerEngineer` 는 `Builder` 에 정의된 메소드를 사용해서 컴퓨터를 조립하는 클래스이다.
- `Builder` 패턴에서 `Director` 역할을 수행한다.
- 생성자에서 `Builder` 인스턴스를 받고 있는데 이는 실제로 `ConcreteBuilder` 이기 때문에, 하위 클래스에 따라 만들어지는 컴퓨터가 결정된다.
- `construct()` 메소드에서는 `Builder` 에 있는 부품들을 통해 실제로 컴퓨터를 조립하는 과정을 구현했다.


### PersonalComputerBuilder

```java
public class PersonalComputerBuilder extends ComputerBuilder {
    @Override
    public void buildCpu() {
        this.computer.setCoreClock(3.4f);
        this.computer.setCoreCount(4);
    }

    @Override
    public void buildRam() {
        this.computer.setRamSize(4);
        this.computer.setRamCount(2);
    }

    @Override
    public void buildSsd() {
        this.computer.setSsdSize(128);
        this.computer.setSsdCount(1);
    }

    @Override
    public void buildHdd() {
        this.computer.setHddSize(512);
        this.computer.setHddCount(1);
    }
}
```  

- `PersonalComputerBuilder` 는 PC 에 들어가는 부품을 만드는 클래스이다.
- `Builder` 패턴에서 `ConcreteBuilder` 역할을 수행한다.
- `Builder` 에 정의된 각 부품들에 PC 에 맞는 각 부품을 실제로 만들고 있다.

### SuperComputerBuilder

```java
public class SuperComputerBuilder extends ComputerBuilder {

    @Override
    public void buildCpu() {
        this.computer.setCoreClock(4);
        this.computer.setCoreCount(64);
    }

    @Override
    public void buildRam() {
        this.computer.setRamSize(16);
        this.computer.setRamCount(32);
    }

    @Override
    public void buildSsd() {
        this.computer.setSsdSize(1000);
        this.computer.setSsdCount(8);
    }

    @Override
    public void buildHdd() {
        this.computer.setHddSize(10000);
        this.computer.setHddCount(16);
    }
}
```  

- `SuperComputerBuilder` 는 슈퍼컴퓨터에 들어간느 부품을 만드는 클래스이다.
- `Builder` 패턴에서 `ConcreteBuilder` 역할을 수행한다.
- `Builder` 에 정의된 각 부품에 슈퍼컴퓨터에 맞는 각 부품을 실제로 만들고 있다.

### 테스트

```java
public class GoFBuilderTest {
    @Test
    public void personalComputer() {
        // given
        PersonalComputerBuilder builder = new PersonalComputerBuilder();
        ComputerEngineer engineer = new ComputerEngineer(builder);

        // when
        engineer.construct();

        // then
        Computer actual = builder.getComputer();
        assertThat(actual.getCoreClock(), is(3.4f));
        assertThat(actual.getCoreCount(), is(4));
        assertThat(actual.getRamSize(), is(4));
        assertThat(actual.getRamCount(), is(2));
        assertThat(actual.getSsdSize(), is(128));
        assertThat(actual.getSsdCount(), is(1));
        assertThat(actual.getHddSize(), is(512));
        assertThat(actual.getHddCount(), is(1));
    }

    @Test
    public void superComputer() {
        // given
        SuperComputerBuilder builder = new SuperComputerBuilder();
        ComputerEngineer engineer = new ComputerEngineer(builder);

        // when
        engineer.construct();

        // then
        Computer actual = builder.getComputer();
        assertThat(actual.getCoreClock(), is(4f));
        assertThat(actual.getCoreCount(), is(64));
        assertThat(actual.getRamSize(), is(16));
        assertThat(actual.getRamCount(), is(32));
        assertThat(actual.getSsdSize(), is(1000));
        assertThat(actual.getSsdCount(), is(8));
        assertThat(actual.getHddSize(), is(10000));
        assertThat(actual.getHddCount(), is(16));
    }
}
```  

## 유연하게 인스턴스 생성하기

---
## Reference

	