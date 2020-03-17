--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Observer Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '오브젝트의 상태변화를 다른 오브젝트에 알리는 Observer 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Observer
use_math : true
---  

## Observer 패턴이란
- `Observer` 는 관찰자라는 의미를 가진 것과 같이, 객체의 상태가 변할때 이를 관찰자에게 알려줘서 상태에 변화에 따른 처리를 구성할 수 있는 패턴이다.

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_observer_1.png)

- `Subject` : 관찰이 되는 대상자를 의미하고, 관찰차인 `Observer` 를 등록 및 삭제하는 메소드와 현재상태를 취득하는 메소드를 정의하는 추상 클래스이다.
- `ConcreteSubject` : `Subject` 의 구현체로 관찰의 대상을 구체적으로 구현한 클래스이다.
- `Observer` : 관찰자로 `Subject` 의 상태 변화를 인지하면 하위 구현체에 따라 구현하는 메소드를 선언하는 인터페이스이다.
- `ConcreteObserver`: `Observer` 의 구현체로 `Subject` 의 상태 변화에 따라 상태를 받아 실제 처리를 수행하는 클래스이다.

### 상위 타입을 사용해서 높은 확장성 제공
- 확장성은 아래와 같은 특징을 통해 제공 될 수 있다.
	- 추상 클래스나 인터페이스를 사용해서 구현(하위) 클래스로부터 추상 메소드를 분리한다.
	- 인스로 인스턴스를 전달 하거나, 필드에서 인스턴스를 저장할 때 하위 클래스의 형태를 사용하지않고 상위 타입(추상 클래스, 인터페이스)의 형태를 사용한다.
- 위와 같은 특징들을 사용해서 `Subject` 와 `Observer` 가 지속적으로 상호작용을 할때, 구현에 따라 추가되는 하위 클래스는 큰 영향 없이 패턴에 적용 시킬 수 있다.

### `Observer` 호출 순서의 독립성
- `Subject` 에서 상태의 변화 가 있을 때, 한개 이상의 `Observer` 에게 상태변화를 알리게 된다.
- 상태 변화를 알릴때 `Observer` 의 호출 순서와는 무관하게 `ConcreteObserver` 의 처리가 가능하도록 설계해야 한다.

### `Observer` 의 동작이 `Subject` 에 영향을 주는 경우
- `Observer` 의 구현 동작이 `Subject` 의 상태를 변경하는 경우를 의미한다.
- `Subject 상태 변화` -> `Observer 전달` -> `Observer 가 Subject 상태 변경` -> `Observer 전달` -> .. 반복 ..
- 이러한 반복을 방지하기 위해서 `Observer` 에서 `Subject` 에게 상태를 받아 처리 중인지를 나타내는 플래그를 사용해서 관리할 수 있다.

### `Subject` 의 상태를 `Observer` 에게 전달하는 다양한 방법
- `Subject` 는 문자열(`str`)에 대한 상태값을 가지고 있다고 가정한다.
- `void update(Subject subject);`
	- `Subject` 의 인스턴스를 `Observer` 에게 전달하고, `Observer` 는 `Subject` 인스턴스 사용해서 동작을 수행한다.
- `void update(Subject subject, String str);`
	- `Subject` 의 인스턴스와 상태값인 `str` 도 같이 전달해서, 항상 얻어야 하는 상태값을 바로 전달하는 방식이다.
	- 이러한 방식은 `Subject` 가 `Observer` 의 처리에 대해 인지하고 있는 경우이다.
- `void update(String str);`
	- `Subject` 의 상태 값만 `Observer` 에게 전달하는 방식으로, `Observer` 에서 필요라한 정보가 상태값 뿐이면 동작은 가능하지만 추후 확장성에 대해서는 고민이 필요하다.
- `Subject` 에서 `Observer` 에게 정보를 전달할 때, 얼마나 어떤 정보를 전달할지는 구현하는 내용에 따라 적절한 방식을 선정해야 한다.

### `Observer` 는 정보를 전달 받는 역할
- `Observer` 는 의미와는 달리 정보를 전달 받는 다는 수동적인 역할을 내포하고 있다.
- `Observer` 패턴을 다른 용어로는 `Publish-Subscribe` 패턴이라고도 한다.
- `Publish` 는 발생이라는 의미로 `Subject` 와 같은 역할을 수행하고, `Subscribe` 는 구독이라는 의미로 `Observer` 와 같은 역할을 수행한다.

### `java.util.Observer`
- Java 라이브러리 중에서는 `Observer` 패턴을 제공해주는 `java.util.Observer` 인터페이스와 `java.util.Observable` 클래스가 있다.
- `Observer` 인터페이스는 `void update(Observable obj, Object arg)` 라는 메소드를 가지고 있다.
- Java 라이브러리에서 제공하는 `Observer` 를 사용하기 위해서는 `Subject` 역할을 수행하는 클래스가 `Observable` 클래스의 하위 클래스일 경우에 가능하다는 것을 인지할 필요가 있다.
	- Java 는 단일 상속인 점을 고려하면 사용하기 까다로운 구조이다.
		
### 기타
- MVC 패턴에서 Model 과 View 는 `Subject` 와 `Observer` 의 관계이다.

## 로그 수집기
- 애플리케이션에서는 테스트용이던, 유저 정보수집을 위한 목적이던 수많은 로그가 쌓이게 된다.
- 애플리케이션은 다양한 환경(테스트, 상용 ..)에서 실행되고, 로그의 종류(debug, info, error ..) 도 다양하다.
- 로그를 발생시키는 `Subject` 와 이를 각 목적에 맞게 수집하는 `Observer` 로 구성해서 구현해 본다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_observer_2.png)

### Log

```java
public class Log {
    private String profile;
    private String level;
    private String contents;

    public Log(String profile, String level, String contents) {
        this.profile = profile;
        this.level = level;
        this.contents = contents;
    }
    
    // getter, setter
}
```  

- `Log` 는 로그를 나타내는 클래스이다.
- `profile` 은 로그가 남겨진 시스템의 프로필 정보 필드이다.(dev, prod)
- `level`  남겨진 로그의 레벨을 의미한다.(debug, info)
- `contents` 는 남겨진 로그의 내용을 의미한다.

### Observer

```java
public interface Observer {
    void update(LogGenerator generator);
}
```  

- `Observer` 는 관찰자가 수행해야하는 메소드르 정의한 인터페이스이다.
- `Observer` 패턴에서 `Observer` 역할을 수행한다.
- 하위 클래스에서는 `update()` 메소드를 통해 `Subject` 로 부터 전달받은 상태의 변화를 구현체에 따라 처리한다.

### LogGenerator

```java
public abstract class LogGenerator {
    private List<Observer> observerList;
    protected Log log;

    public LogGenerator() {
        this.observerList = new LinkedList<>();
    }

    public void addObserver(Observer observer) {
        this.observerList.add(observer);
    }

    public void deleteObserver(Observer observer) {
        this.observerList.remove(observer);
    }

    public void notifyObservers() {
        for(Observer observer : this.observerList) {
            observer.update(this);
        }
    }

    public Log getLog() {
        return this.log;
    }

    public abstract void generate(String level, String profile, String contents);
}
```  

- `LogGenerator` 는 로그를 생성하는 추상 클래스이다.
- `Observer` 패턴에서 `Subject` 의 역할을 수행한다.
- `observerList` 필드에서 여러개의 `Observer` 를 담아 관리한다.
- `log` 필드는 현재 상태를 의미하는 필드이다.
- `addObserver()` 메소드는 관찰자를 추가하고, `deleteObserver()` 는 관찰자를 삭제한다.
- `notifyObservers()` 메소드를 통해 등록된 관찰자들에게 `Subject` 의 상태 변화(로그 생성)을 알린다.
- `generate()` 로그를 생성하는 추상 메소드로 하위 클래스에서 각 로그 생성기에 맞춰 구현한다.

### PlainLogGenerator

```java
public class PlainLogGenerator extends LogGenerator{
    @Override
    public void generate(String level, String profile, String contents) {
        this.log = new Log(profile, level, contents);
        this.notifyObservers();
    }
}
```  

- `PlainLogGenerator` 는 `LogGenerator` 의 구현체로 별다른 구현없이 로그를 남기는 로그 생성기를 나타내는 클래스이다.
- `Observer` 패턴에서 `ConcreteSubject` 역할을 수행한다.
- `generate()` 메소드에서는 인자 값으로 받은 값들을 바탕으로 `Log` 인스턴스를 만들어 필드에 설정하고 `notifyObservers()` 메소드를 통해 알린다.

### LogObserver

```java
public abstract class LogObserver implements Observer{
    protected List<Log> logList;

    public LogObserver() {
        this.logList = new LinkedList<>();
    }
    
    // getter, setter
}
```  

- `LogObserver` 는 `Observer` 의 하위 구현체로 로그를 관찰자(수집기)를 나타내는 추상 클래스이다.
- `Observer` 패턴에서 `ConcreteObserver` 의 역할을 수행한다.
- `logList` 필드를 통해 각 로그 관찰자가 수신한 로그를 관리한다.

### LogAllObserver, LogLevelObserver, LogProfileObserver

```java
public class LogAllObserver extends LogObserver {
    @Override
    public void update(LogGenerator generator) {
        this.logList.add(generator.getLog());
    }
}
```  

```java
public class LogLevelObserver extends LogObserver {
    private String logLevel;

    public LogLevelObserver(String logLevel) {
        this.logLevel = logLevel;
    }

    @Override
    public void update(LogGenerator generator) {
        if(this.logLevel.equals(generator.getLog().getLevel())) {
            this.logList.add(generator.getLog());
        }
    }
    
    // getter, setter
}
```  

```java
public class LogProfileObserver extends LogObserver {
    private String profile;

    public LogProfileObserver(String profile) {
        this.profile = profile;
    }

    @Override
    public void update(LogGenerator generator) {
        if(this.profile.equals(generator.getLog().getProfile())) {
            this.logList.add(generator.getLog());
        }
    }
    
    // getter, setter
}
```  

- `LogAllObserver`, `LogLevelObserver`, `LogProfileObserver` 는 모두 `LogGenerator` 의 구현체로 각기 다른 로그 수집기를 나타내는 클래스이다.
- `Observer` 패턴에서 `ConcreteObserver` 역할을 수행한다.
- `LogAllObserver` 는 `LogGenerator` 의 모든 로그를 수집한다.
- `LogLevelObserver` 는 `LogGenerator` 에서 생성된 로그 중 `level` 필드 값과 일치한 로그만 수집한다.
- `LogProfileObserver` 는 `LogGenerator` 에서 생성된 로그 중 `profile` 필드 값과 일치한 로그만 수집한다.


### 테스트

```java
public class ObserverTest {
    @Test
    public void LogGenerator() {
        // given
        LogGenerator logGenerator = new PlainLogGenerator();
        LogObserver debugLevelObserver = new LogLevelObserver("debug");
        LogObserver infoLevelObserver = new LogLevelObserver("info");
        LogObserver devProfileObserver = new LogProfileObserver("dev");
        LogObserver localProfileObserver = new LogProfileObserver("local");
        LogObserver allObserver = new LogAllObserver();
        logGenerator.addObserver(debugLevelObserver);
        logGenerator.addObserver(infoLevelObserver);
        logGenerator.addObserver(devProfileObserver);
        logGenerator.addObserver(localProfileObserver);
        logGenerator.addObserver(allObserver);

        // when
        logGenerator.generate("debug", "local", "debug local log");
        logGenerator.generate("debug", "dev", "debug dev log");
        logGenerator.generate("info", "local", "info local log");
        logGenerator.generate("info", "dev", "info dev log");
        logGenerator.generate("info", "local", "error local log");

        // then
        assertThat(debugLevelObserver.getLogList(), hasSize(2));
        assertThat(debugLevelObserver.getLogList(), everyItem(hasProperty("level", is("debug"))));
        assertThat(infoLevelObserver.getLogList(), hasSize(3));
        assertThat(infoLevelObserver.getLogList(), everyItem(hasProperty("level", is("info"))));
        assertThat(devProfileObserver.getLogList(), hasSize(2));
        assertThat(devProfileObserver.getLogList(), everyItem(hasProperty("profile", is("dev"))));
        assertThat(localProfileObserver.getLogList(), hasSize(3));
        assertThat(localProfileObserver.getLogList(), everyItem(hasProperty("profile", is("local"))));
        assertThat(allObserver.getLogList(), hasSize(5));
    }
}
```  
	
---
## Reference

	