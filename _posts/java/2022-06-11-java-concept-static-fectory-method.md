--- 
layout: single
classes: wide
title: "[Java 개념] 정적 팩토리 메소드(Static Factory Method)"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: '생성자 이외에 객체의 인스턴스를 좀더 효율적으로 생성하고 제어까지 할 수 있는 정적 팩토리 메소드에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Effective Java
  - Static Factory Method
toc: true 
use_math: true
---  

## 정적 팩토리 메소드(Static Factory Method)
`Effective Java` 의 `Item 1. 생성자 대신 정적 팩토리 메소드를 고려하라` 를 보면 
객체의 인스턴스를 생성할때 일반적인 `public 생성자`도 있지만 `Static Factory Method` 의 
장점이 더 많기 때문에 이를 고려하라는 내용이 있다.  

물론 `public 생성자`, `Static Factory Method` 는 지니고 있는 장/단점이 모두 다르기 때문에, 
상황에 따라 선택해서 사용하는 편이 좋지만 전체적으로 보았을 때 `Static Factory Method` 의 장점이 더 많다는 내용이다.  

`Static Factory Method` 란 아래와 같은 `Boolean` 객체의 예시를 들 수 있다. 

```java
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);
}
```  

위 메소드는 `primitive type` 인 `boolean` 을 인자로 받아 `reference type` 인 `Boolean` 으로 반환하는 동작을 수행한다.  

> 이는 디자인 패턴의 `팩토리 메소드 패턴` 과는 다른 것을 의미한다. 

## 장점
### 이름을 가질 수 있다. 

아래와 같은 `Computer` 클래스가 있다고 가정하자. 

```java
public class Computer {
    private String cpu;
    private String ram;

    public Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public String boot() {
        return String.format("cpu : %s, ram : %s", this.cpu, this.ram);
    }
}

// WorkStation
new Computer("core-16", "64GB");
// Notebook
new Computer("core-8-u", "8GB");
```  

이제 `Computer` 클래스의 `public 생성자` 원하는 컴퓨터 스펙을 인자로 전달해 `Computer` 인스턴스를 생성할 수 있다.  

하지만 `public 생성자` 를 사용하는 경우 구체적으로 어떠한 컴퓨터가 생성되는지를 파악할 수 없다. 
만약 해당 인스턴스에 대한 설명이 필요하다면 주석을 사용해서 추가 설명이 필요하다. 
하지만 `Static Factory Method` 를 사용하는 경우 아래와 같이 생성하는 컴퓨터의 이름을 부여해 인스턴스를 생성할 수 있다.  

```java
public class Computer {
    private String cpu;
    private String ram;

    private Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public static Computer createWorkStation() {
        return new Computer("core-16", "64GB");
    }

    public static Computer createNotebook() {
        return new Computer("core-8-u", "8GB");
    }
}

Computer.createWorkStation();
Computer.createNotebook();
```  

위 예시에서는 단순히 컴퓨터 스펙에 따른 이름 부여를 했지만, 
이는 필요에 따라 다양한 방식으로 이름을 부여해 인스턴스의 특성을 구체화 할 수 있기 때문에 
상황에 맞는 알맞는 인스턴스를 생성해 사용할 수 있을 것이다.  


### 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다. 
`public 생성자` 사용해서 인스턴스를 생성하는 방법 밖에 없다면, 
매번 새로운 인스턴스를 생성해야 한다. 
하지만 정말로 새로운 인스턴스가 필요한 상황이 아니라면 이와 같은 방식은 단점이 더 많다.  

대표적으로 인스턴스를 생성하는 것 또한 시스템의 리소스를 소모하는 작업이기 때문에, 
동시에 많은 인스턴스를 생성한다거나 큰 비용이 들어가는 인스턴스의 경우에는 성능적으로 좋지 않다.  

위와 같은 부분을 `Static Factory Method` 를 사용해서 매번 새로운 인스턴스를 생성해 리턴하는 것이 아니라, 
불변 클래스(`immutable class`) 를 통해 인스턴스를 미리 만들어 두고 캐싱해서 재활용 하는 식으로 비용을 줄일 수 있다. 
이런 방식은 디자인 패턴의 [Flyweight pattern]({{site.baseurl}}{% link _posts/designpattern/2020-03-21-designpattern-concept-flyweight.md %})
과 유사한 기법이다.  

```java
public class Computer {
    private String cpu;
    private String ram;

    private static Map<String, Computer> COMPUTER_WAREHOUSE = Map.of(
            "workStation", new Computer("core-16", "64GB"),
            "notebook", new Computer("core-8-u",  "8GB")
    );

    private Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public static Computer rentWorkStation() {
        return COMPUTER_WAREHOUSE.get("workStation");
    }

    public static Computer rentNotebook() {
        return COMPUTER_WAREHOUSE.get("notebook");
    }
}

assertThat(Computer.rentWorkStation(), is(Computer.rentWorkStation()));
assertThat(Computer.rentNotebook(), is(Computer.rentNotebook()));
```  

위 예시와 같이 미리 인스턴스를 캐싱해두고 `Static Factory Method` 를 통해 캐싱된 인스턴스를 반환하는 방식으로 사용할 수 있다.  

이러한 방식은 반복되는 요청에 같은 인스턴스를 반환하는 식으로 해당 객체의 인스턴스를 통제할 수 있고, 
이를 인스턴스 통제(`instance-controlled`) 라고 한다. 
이렇게 인스턴스를 통제하면 단 하나의 인스턴스만 보장하는 싱글턴(`singleton`), 인스턴스화 불가(`noninstantiable`) 로 활용 할 수 있다. 


### 반환 타입의 하위 타입 객체를 반환할 수 있다. 
`Static Factory Method` 는 메소드를 통해 인스턴스를 리턴하는 방식이다. 
그러므로 `public 생성자` 에서는 수행할 수 업었던 다양한 추가 동작을 행할 수 있다. 
그 중 하나가 하위 타입의 인스턴스를 반환하는 것이다.  

이러한 동작은 필요에 따라 알맞는 하위 타입의 인스턴스를 반환하기 때문에 아주 큰 유연성을 지니고 있다고 할 수 있다. 
객체지향에 있어서 상속에 따른 하위 타입 정의는 객체에 있어서 속성, 기능의 확장과 그에 따른 다형성을 제공한다. 
이러한 객체지향 특징을 `Static Factory Method` 를 통해 활용도가 더 높아 지는 것이다.  

```java
public class Computer {
    private String cpu;
    private String ram;

    protected Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public static Computer createWorkStation() {
        return new WorkStation();
    }

    public static Computer createNotebook() {
        return new Notebook();
    }
}

class WorkStation extends Computer {
    public WorkStation() {
        super("cpu-16", "64GB");
    }
}

class Notebook extends Computer {
    public Notebook() {
        super("cpu-8-u", "8GB");
    }
}

Computer c1 = Computer.createWorkStation();
Computer c2 = Computer.createNotebook();
```  

위 예시와 같이 `Computer` 를 상속해 하위 타입으로 정의한 `WorkStation`, `Notebook` 을 `Static Factory Method` 에서 반환하고, 
두 하위 타입은 자신의 객체 특성에 따라 속성 및 기능적 확장이 가능하다.  


### 입력 매개변수에 따른 다른 클래스의 객체 반환
`public 생성자` 에서는 행할 수 없었던 동작 중 다른 하나는 매개변수에 따른 객체를 반환하는 것이다. 
다형성을 사용하면 하위 타입이라면 어떤 클래스를 리턴하더라도 문제가 되지 않는다. 
우리는 이러한 특성을 활용해 매개변수에 따라 알맞는 객체의 인스턴스를 리턴할 수 있도록 할 수 있다.  

```java
public class Computer {
    private String cpu;
    private String ram;

    enum Spec {
        PORTABLE,
        HIGH_END
    }

    protected Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public static Computer createComputer(Spec spec) {
        if(Spec.HIGH_END.equals(spec)) {
            return new WorkStation();
        } else if(Spec.PORTABLE.equals(spec)) {
            return new Notebook();
        } else {
            return new Computer("cpu-8", "8GB");
        }
    }

}

class WorkStation extends Computer {
    public WorkStation() {
        super("cpu-16", "64GB");
    }
}

class Notebook extends Computer {
    public Notebook() {
        super("cpu-8-u", "8GB");
    }
}

Computer c1 = Computer.createComputer(Spec.HIGH_END);
Computer c2 = Computer.createComputer(Spec.PORTABLE);
```  

이제 `createComputer` 메소드만 사용하면 원하는 `Computer` 인스턴스를 받아 사용할 수 있다. 
이는 추후에 기능적으로, 성능적으로 개선된 객체가 나온다면 이를 리턴하도록 추가만 해주면 된다. 
그리고 특정 객체를 `Deprecated` 한다거나 할때도 제외하는 등 인스턴스 생성에 있어서 유연하게 대응할 수 있다.  


### 정적 팩토리 메소드 작성 시점에 반환할 객체의 클래스가 존재하지 않아도 된다. 
구현해야 하는 객체가 어떤 속성 설정에 따라 다른 동작을 수행해야 하는 경우라고 가정해본다. 
위의 대표적인 예시가 `JDBC` 이다. 
`JDBC` 를 보면 `DriverManager.registerDriver()` 를 통해 `JDBC` 에서 사용할 드라이버를 등록하면, 
사용자는 `DriverManager.getConnection()` 을 통해 `Driver` 에 맞는 알맞는 `DB Connection` 을 얻어 사용할 수 있다. 
이러한 구현 또한 `Static Factory Method` 이다. 

```java
public class Computer {
    private String cpu;
    private String ram;

    private static ComputerEngineer COMPUTER_MAKER;

    protected Computer(String cpu, String ram) {
        this.cpu = cpu;
        this.ram = ram;
    }

    public static void registerMaker(ComputerEngineer computerEngineer) {
        COMPUTER_MAKER = computerEngineer;
    }

    public static Computer getComputer() {
        return COMPUTER_MAKER.make();
    }
}

interface ComputerEngineer {
    Computer make();
}

class WorkStationEngineer implements ComputerEngineer {
    @Override
    public Computer make() {
        return new WorkStation();
    }
}

class NotebookEngineer implements ComputerEngineer {
    @Override
    public Computer make() {

        return new Notebook();
    }
}

class WorkStation extends Computer {
    public WorkStation() {
        super("cpu-16", "64GB");
    }
}

class Notebook extends Computer {
    public Notebook() {
        super("cpu-8-u", "8GB");
    }
}

Computer.registerMaker(new WorkStationEngineer());
Computer c1 = Computer.getComputer();
Computer.registerMaker(new NotebookEngineer());
Computer c2 = Computer.getComputer();
```  

위 예시는 `JDBC` 에서 `DB Connection` 생성에 필요한 해당 `DB` 의 `Driver` 를 설정하면, 
`Driver` 의 구현체가 알아서 `JDBC` 를 통해 `DB Connection` 을 반환하는 것과 동일하다. 
이와 같은 활용은 각 객체의 기능과 동작이 상의 할때 알맞는 인터페이스의 구현체(`Driver`와 같은)
을 등록해 원하는 객체의 인스턴스를 그때 그때 받아 사용할 수 있다.  

이는 달리 표현하면 `Static Factory Method` 를 사용해서 인스턴스의 생성을 인스턴스 생성 역할을 하는 인터페이스에게 위임(`delegation`) 하는 방식이라고 할 수 있다.  



## 단점
### Static Factory Method 상속 불가
이는 `Java` 문법에서 `Static Method` 는 상속이 불가하는 점에서 출발한다. 
`Static Factory Method` 를 사용하는 클래스의 하위 타입을 정의하고, 
`Override` 가 필요하다면 이는 다른 방식으로 구조화가 필요할 수 있다. 
굳이 상속을 사용하지 않고 대신 `Composition` 과 `불변 타입` 사용 하는 등의 방법이 있다.  

### 의미의 불명확성
`public 생성자` 는 `Java` 문법적으로 인스턴스를 생성한다 라는 정의가 돼있다. 
하지만 `Static Factory Method` 는 문법적으로 정해진 것은 아니지만, 
프로그래머가 필요와 편의에 의해 인스턴스를 제공하는 메소드이다. 

이러한 특징으로 만약 `Static Factory Method` 의 정의를 잘 모르는 프로그래머가 
본다면 생성자를 사용활 수 없는 상황에서 해당 객체의 인스턴스를 어떤 방법으로 생성해야 할지 모를 수도 있다. 
그리고 위 상황 뿐만 아니라, `Static Factory Method` 는 인스턴스 생성에 대한 이름을 부여할 수 있다고 했지만 
다른 말로 풀면 개발자 마다 다른 네이밍을 사용한다면 해당 `Static Factory Method` 가 어떤 인스턴스를 반환하는지 명확하지 않아 혼란만 더 초래 할 수 있다.  

위와 같은 문제를 해결하는 방법은 `API 문서` 를 철저하게 작성하고 관리한다거나, 
아래와 같은 보편적으로 알려진 규약을 통해 `Static Factory Method` 의 이름을 짓는 방법으로 어느정도 해결 할 수 있다.  

- `from` : 매개변수를 하나 받아서 해당 타입의 인스턴스를 반환하는 형변환 메소드
  - e.g. : `Date d = Date.from(instant)`
- `of` : 여러 매개변수를 받아 적합한 타입의 인스턴스를 반환하는 집계 메소드
  - e.g. : `Map<String, String> map = Map.of("key", "value")`
- `valueOf` : `from`, `of` 의 더 자세한 버전
  - e.g. :  `Integer i = Integer.valueOf("1")`
- `instance`, `getInstance`(매개변수를 받는 경우) : 매개변수로 명시한 인스턴스를 반환하지만, 같은 인스턴스임을 보장하진 않음
  - e.g. :  `ApplicationManager manager = ApplicationManager.getInstance(options)`
- `create`, `newInstance` : `instance`, `getInstance` 와 역할은 동일하지만, 매번 새로운 인스턴스를 반환한다. 
  - e.g. :  `Integer[] list = (Integer[])Array.newInstance(Integer.class, 2)`
- `getType` : `getInstance` 와 같지만, 생성할 클래스가 아닌 다른 클래스에 팩토리 메소드를 정의할 떄 쓴다. 여기서 `Type` 은 반환 할 객체의 타입이름을 의미한다. 
  - e.g. : `FileStore fileStore = Files.getFileStore(path)`
- `newType` : `newInstance` 와 같지만, 생성할 클래스가 아닌 다른 클래스에 맥토리 매소드를 정의할 떄 쓴다. 여기서 `Type` 은 반환 할 객체의 타입이름을 의미한다. 
  - e.g. :  `BufferedReader bufferedReader = Files.newBufferedReader(path)`
- `type` : `getType` 과 `newType` 의 간결한 버전
  - e.g. :  `ArrayList<Integer> intList = Collections.list(intVector.elements())`



---
## Reference
[Effective Java](https://book.naver.com/bookdb/book_detail.nhn?bid=14097515)  


