--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Abstract Factory Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '추상적인 부품을 사용해서 추상적인 제품을 만드는 Abstract Factory 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Abstract Factory
---  


## Abstract Factory 패턴이란
- `Abstract Factory` 는 '추상적인 공장' 이라는 의미를 가지고 있는 것처럼, 추상적인 무언가를 만들어내는 공장과 같다.
- 추상적인 공장에서는 추상적인 부품을 사용해서 추상적인 제품을 만들어 낸다.
- 추상적인 부품과 제품이라는 것은 구체적인 구현을 생각하기 전에, 어떠한 구성으로 돼있는지 먼저 간략하게 정의하는 것을 뜻한다.
- 위와 같은 개념은 
[Template Method]({{site.baseurl}}{% link _posts/designpattern/2020-01-05-designpattern-concept-templatemethod.md %})
와 
[Factory Method]({{site.baseurl}}{% link _posts/designpattern/2020-01-06-designpattern-concept-factorymethod.md %}) 
등에서 사용해 왔다. 추상 클래스를 사용해서 인스턴스를 생성한다는 점이 `Factory Method` 패턴과 비슷해 보일 수 있는데 차이점은 `Abstract Factory` 패턴은 서로 관련이 있는 객체들을 부품으로 조합해 하나의 큰 객체인 제품을 만드는 점이다.
- `Abstract Factory` 패턴을 구성하면서 주의해야 할점은 `if else` 혹은 `switch` 문을 통해 객체를 판별하는 코드는 OOP 적인 디자인이 아니기 때문에 지양해야 한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_abstractfactory_1.png)

- 패턴의 구성요소
	- `AbstractProduct` : `AbstractFaoctry` 에 의해 만들어지는 추상적인 부품, 제폼을 정의한다.
	- `AbstractFactory` : `AbstractProduct` 인스턴스를 만들어 내기 위한 인터페이스를 정의한다.
	- `ConcreteProduct` : `AbstractProduct` 를 구현한 하위 클래스이다. 만들고자하는 부품, 제품이 이 클래스에 해당된다.
	- `ConcreteFactory` : `AbstractFactory` 를 구현한 클래스이다. `ConcreteProduct` 의 인스턴스를 만들어낸다.
- `AbstractFactory` 패턴을 보면 추상적인 부분인 `factory` 패키지와 실제 구현 부분인 `concretefactory` 패키지로 나눠져 구성되어 있다.
- `AbstractFactory` 패턴에서 새로운 `concretefacotry` 패키지 즉, 새로운 제품을 만들어내는 공장을 추가하는 것은 간단하다.
	- `factory` 패키지에 있는 것들을 구체화 하면 된다.
- 반대로 `AbstractFactory` 패턴에서 새로운 부품 즉, `concretefaoctry` 패키지 내에 있는 `ConcreteProduct` 를 추가하는 것은 어려울 수 있다.
	- `concretefactory` 패키지에 새로운 `ConcreteProduct` 를 추가한다는 것은, `factory` 패키지부터 `concretefacotry` 패키지 까지 구조 수정 뿐만 아니라, 클래스내 메소드까지 모두 수정해야 할 수 있기 때문이다.


## 컴퓨터 공장 만들기
- 한 회사에서 Good 이라는 컴퓨터 제품 라인이 있다.
- Good 컴퓨터를 만들기위해서 `factory` 를 구성하고 컴퓨터를 만들기 위한 부품을 통해 최종적으로 컴퓨터라는 제품을 만들어 낸다.


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_abstractfactory_2.png)

- 위 그림과 같이 크게 `factory` 패키지와, `good` 패키지로 구성돼 있다.
	- `factory` 패키지는 컴퓨터와 컴퓨터에 필요한 제품을 만들기위한 추상적인 내용으로 구성돼 있다.
	- `good` 패키지는 Good 제품 컴퓨터와 필요한 제품을 만들기위한 구체적인 내용으로 구성돼 있다.

### factory.Part

```java
public abstract class Part {
    public abstract Part makePart();

    public abstract float calculateSpec();
}
```  

- `Part` 는 컴퓨터의 부품을 나타내는 추상 클래스이다.
- `Abstract Factory` 패턴에서 `AbstractProduct` 역할을 수행한다.
- 컴퓨터의 부품 클래스들은 해당 클래스를 상속받아 구현된다.
- 하위 클래스에서는 부품을 실제로 생성하는 `makePart()` 추상 메소드와 부품의 스펙을 계산하는 `calculateSpec()` 추상 메소드를 구현해야 한다.

### factory.Cpu

```java
public abstract class Cpu extends Part {
    protected float coreClock;
    protected int coreCount;

    @Override
    public float calculateSpec() {
        return this.coreClock * coreCount;
    }

	// getter, setter
}
```  

- `Cpu` 는 컴퓨터의 부품 CPU 를 나타내는 추상 클래스이다.
- `Abstract Factory` 패턴에서 `AbstractProduct` 역할을 수행한다.
- 컴퓨터의 부품이기 때문에 `Part` 클래스를 상속한다.
- 다양한 CPU 를 만들 수 있도록 추상 클래스로 두고, 하위에서 보다 구체적인 CPU 를 정의하고, 실제로 생성하는 `makePart()` 추상 메소드를 구현하도록 했다.
	- Intel CPU, AMD CPU
- CPU 의 클럭 수와 코어 수가 필드에 있다.
- CPU 의 스펙은 클럭 수 * 코어 수로 계산한다.

### factory.Ram

```java
public abstract class Ram extends Part {
    protected int size;

    @Override
    public float calculateSpec() {
        return this.size;
    }
    
	// getter, setter
}
```  

- `Ram` 은 컴퓨터의 부품 RAM 을 나타내는 추상 클래스이다.
- `Abstract Factory` 패턴에서 `AbstractProduct` 역할을 수행한다.
- 컴퓨터의 부품이기 때문에 `Part` 클래스를 상속한다.
- 다양한 RAM 을 만들 수 있도록 추상 클래스로 두고, 하위에서 보다 구체적인 RAM 을 정의하고, 실제로 생성하는 `makePart()` 추상 메소드를 구현 해야 한다.
	- Samsung RAM, SK Hynix RAM
- RAM 의 용량이 필드에 있다.
- RAM 의 스펙은 용량으로 계산한다.

### factory.Computer

```java
public abstract class Computer {
    protected Cpu cpu;
    protected List<Ram> ram;

    public Computer() {
        this.ram = new ArrayList<>();
    }

    public abstract Computer makeComputer(Cpu cpu, Ram ram);

    public float calculateSpec() {
        float result = this.cpu.calculateSpec();

        for(Ram ram : this.ram) {
            result += ram.calculateSpec();
        }

        return result;
    }

    public void addRam(Ram ram) {
        this.ram.add(ram);
    }
	// getter, setter
}
```  

- `Computer` 는 컴퓨터는 나타내는 추상 클래스이다.
- `Abstract Factory` 패턴에서 `AbstractProduct` 역할을 수행한다.
- CPU 부품, RAM 부품 배열을 필드로 가지고 있다.
- 하위 클래스에서 각 제조사에(공장) 맞는 컴퓨터를 정의하고, 실제로 생성하는 `makeComputer()` 추상 메소드를 구현을 해야 한다.
- Computer 의 스펙은 CPU 스펙 + RAM 스펙으로 계산한다.

### factory.Factory

```java
public abstract class Factory {
    public static Factory getFactory(Class factoryClass) {
        Factory factory = null;

        try {
            factory = (Factory)factoryClass.newInstance();
        } catch(Exception e) {
            e.printStackTrace();
        }

        return factory;
    }

    public abstract Computer createComputer();
    public abstract Cpu createCpu();
    public abstract Ram createRam();
}
```  

- `Factory` 는 인스턴스를 생성하는 메소드가 정의된 추상 클래스이다.
- `Abstract Factory` 패턴에서 `AbstractFactory` 역할을 수행한다.
- 전역 메소드인 `getFactory()` 는 `Factory` 하위 클래스의 이름을 받아 동적으로 인스턴스를 생성해 리턴한다.
- `createComputer()`, `createCpu()`, `createRam()` 추상 메소드는 `Factory` 에서 만들고자 하는 부품이나 제품을 생성할 때 사용하는 메소드이다.
- 위 추상클래스의 구체적인 구현은 `Factory` 의 하위 클래스에서 맡는다.

### good.GoodFactory

```java
public class GoodFactory extends Factory {
    @Override
    public Computer createComputer() {
        return new GoodComputer();
    }

    @Override
    public Cpu createCpu() {
        return (Cpu)new GoodCpu().makePart();
    }

    @Override
    public Ram createRam() {
        return (Ram)new GoodRam().makePart();
    }
}
```  

- `GoodFactory` 는 `Factory` 의 하위 클래스로 실제 인스턴스를 생성하는 구체적인 구현의 내용이 있는 클래스이다.
- `Abstract Factory` 패턴에서 `ConcreteFactory` 역할을 수행한다.
- `Factory` 의 추상 메소드인 `createComputer()`, `createCpu()`, `createRam()` 의 세부 구현은 모두 Good 컴퓨터와 관련 부품 클래스를 이용한다.

### good.GoodCpu

```java
public class GoodCpu extends Cpu {
    @Override
    public Part makePart() {
        this.coreClock = 3.2f;
        this.coreCount = 1;

        return this;
    }
}
```  

- `GoodCpu` 는 `Cpu` 의 하위 클래스로 실제 인스턴스를 생성하는 구체적인 구현의 내용이 있는 클래스이다.
- `Abstract Factory` 패턴에서 `ConcreteProduct` 역할을 수행한다.
- `Part`(`Cpu` 의 상위 클래스)의 추상 메소드인 `makePart()` 를 Good 컴퓨터의 CPU 에 맞는 내용으로 구현하고 있다.

### good.GoodRam

```java
public class GoodRam extends Ram {
    @Override
    public Part makePart() {
        this.size = 8;

        return this;
    }
}
```  

- `GoodRam` 은 `Ram` 의 하위 클래스로 실제 인스턴스를 생성하는 구체적인 구현의 내용이 있는 클래스이다.
- `Abstract Factory` 패턴에서 `ConcreteProduct` 역할을 수행한다.
- `Part`(`Ram` 의 상위 클래스)의 추상 메소드인 `makePart()` 를 Good 컴퓨터의 RAM 에 맞는 내용으로 구현하고 있다.

### good.GoodComputer

```java
public class GoodComputer extends Computer {
    @Override
    public Computer makeComputer(Cpu cpu, Ram ram) {
        this.cpu = cpu;
        this.addRam(ram);

        return this;
    }
}
```  

- `GoodComputer` 은 `Computer` 의 하위 클래스로 실제 인스턴스를 생성하는 구체적인 구현의 내용이 있는 클래스이다.
- `Abstract Factory` 패턴에서 `ConcreteProduct` 역할을 수행한다.
- `Computer` 의 추상 메소드인 `makeComputer()` 를 인자 값으로 받은 CPU 와 RAM 을 통해 컴퓨터를 조립한다.
- 코드에서 알 수 있듯이 Good 컴퓨터의 RAM 은 한개만 있다.

### 테스트

```java
public class GoodComputerTest {
    @Test
    public void goodComputer() {
        // given
        Factory factory = Factory.getFactory(GoodFactory.class);
        Cpu cpu = factory.createCpu();
        Ram ram = factory.createRam();
        Computer computer = factory.createComputer();

        // when
        Computer actual = computer.makeComputer(cpu, ram);

        // then
        assertThat(actual.getCpu().getCoreClock(), is(3.2f));
        assertThat(actual.getCpu().getCoreCount(), is(1));
        assertThat(actual.getRam().size(), is(1));
        assertThat(actual.getRam().get(0).getSize(), is(8));
        assertThat(actual.calculateSpec(), is(3.2f * 1 + 8));
    }
}
```  

## Excellent 컴퓨터 공장 라인 신설하기
- Good 컴퓨터 보다 더 성능적으로 좋은 Excellent 컴퓨터를 만들기 위해 기존 `factory` 패키지를 사용해서 `excellent` 패키지 추가로 새로운 부품 및 제품을 만든다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_abstractfactory_3.png)

### excellent.ExcellentFactory

```java
public class ExcellentFactory extends Factory {
    @Override
    public Computer createComputer() {
        return new ExcellentComputer();
    }

    @Override
    public Cpu createCpu() {
        return (Cpu)new ExcellentCpu().makePart();
    }

    @Override
    public Ram createRam() {
        return (Ram)new ExcellentRam().makePart();
    }
}
```  

- `good.GoodFactory` 와 같은 역할을 수행한다.
- `Factory` 클래스 추상 메소드의 세부 구현을 Excellent 컴퓨터에 맞게 구성 한다.

### excellent.ExcellentCpu

```java
public class ExcellentCpu extends Cpu {
    @Override
    public Part makePart() {
        this.coreClock = 3.4f;
        this.coreCount = 2;

        return this;
    }
}
```  

- `good.GoodCpu` 와 같은 역할을 수행한다.
- `Cpu` 의 스펙을 `makePart()` 추상 메소드 세부 구현을 통해 Excellent 컴퓨터에 맞춰 설정한다.

### excellent.ExcellentRam

```java
public class ExcellentRam extends Ram {
    @Override
    public Part makePart() {
        this.size = 16;

        return this;
    }
}
```  

- `good.GoodRam` 과 같은 역할을 수행한다.
- `Ram` 의 스펙을 `makePart()` 추상 메소드 세부 구현을 통해 Excellent 컴퓨터에 맞춰 설정한다.

### excellent.ExcellentComputer

```java
public class ExcellentComputer extends Computer {
    @Override
    public Computer makeComputer(Cpu cpu, Ram ram) {
        this.cpu = cpu;
        this.addRam(ram);
        this.addRam(ram);

        return this;
    }
}
```  

- `good.GoodComputer` 와 같은 역할을 수행한다.
- `makeComputer()` 추상 메소드를 통해 Excellent 컴퓨터 조립에 대한 세부 구현 내용을 작성한다.
- Good 컴퓨터와 비교해서 램이 2개인 것을 확인 할 수 있다.

### 테스트

```java
public class ExcellentComputerTest {
    @Test
    public void excellentComputer() {
        // given
        Factory factory = Factory.getFactory(ExcellentFactory.class);
        Cpu cpu = factory.createCpu();
        Ram ram = factory.createRam();
        Computer computer = factory.createComputer();

        // when
        Computer actual = computer.makeComputer(cpu, ram);

        // then
        assertThat(actual.getCpu().getCoreClock(), is(3.4f));
        assertThat(actual.getCpu().getCoreCount(), is(2));
        assertThat(actual.getRam().size(), is(2));
        assertThat(actual.getRam().get(0).getSize(), is(16));
        assertThat(actual.getRam().get(1).getSize(), is(16));
        assertThat(actual.calculateSpec(), is(3.4f * 2 + 16 + 16));
    }
}
```  

---
## Reference

	