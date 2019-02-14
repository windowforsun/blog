--- 
layout: post
title: "디자인 패턴(Design Pattern) 싱글턴 패턴(Singleton Pattern)"
subtitle: '싱글턴 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
tags:
    - 디자인 패턴
    - Design Pattern
    - Singleton
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 싱글턴 패턴이란
- 전역 변수를 사용하지 않고 객체를 하나만 생성 하도록 하며, 생성된 객체를 어디에서든지 참조할 수 있도록 하는 패턴
- ![싱글턴 클래스 다이어그램 예시](/img/designpattern-singleton-ex-classdiagram.png)
- 하나의 인스턴스만을 생성하는 책임이 있으며 getInstance() 메서드를 통해 모든 클라이언트에게 동일한 인스턴스를 반환하는 작업을 수행한다.

### 예시
#### 프린터 관리자 만들기
- 동일한 하나의 프린트를 10명이 공유해서 사용한다고 가정하자.
- 아래는 일반적인 프린트 클래스다.
```java
public class Printer {
    public Printer() {}     // @1
    
    public void print(String str){      // @2
        System.out.println(str);
    }
}
```  

- 위의 상황에서 하나의 프린트를 여러명이 공유하기 때문에 여러명의 클라이언트가 하나의 객채를 사용하는 상황이다.
- 이러한 프린트 클래스를 싱클턴으로 작성하면 아래와 같다.

```java
public class PrinterSingleton {
    private static PrinterSingleton instance = null;    // @3
    
    private PrinterSingleton(){}        // @1
    
    public static PrinterSingleton getInstance(){   // @4
        if(instance == null){
            instance = new PrinterSingleton();
        }
        
        return instance;
    }
    
    public void print(String str) {     // @2
        System.out.println(str);
    }    
}
```  

#### 비교 및 분석
- Printer 클래스
    - 생성자, 출력 함수로 간단하게 구성되어 있다.
    - new Printer()(생성자)를 통해 객체를 생성하고, print() 함수를 호출해서 사용해야한다.
    - 즉 여러명이 사용 할 경우 모두 생성자를 통해 프린트 객체를 생성하고 출력함수를 호출하게 된다.
    - 프린트는 정작 1개지만 여러개의 여러개의 프린트를 사용 중인 상황이 되버린다.
- PrintSingleton 클래스(Lazy Initialization)
    - Printer 클래스와 비교해서 @3, @4가 추가 되어, private 클래스 변수, private 생성자, static 메서드, 출력 함수 로 구성되어 있다.
        1. @3 private 클래스 변수는 자신의 인스턴스를 가진다.
        1. @4 private 생성자는 외부로 부터 생성자 호출을 통한 인스턴스 생성를 못하도록 한다.
        1. @4 static 메서드는 외부로 부터 자신의 인스턴스를 리턴 한다.
        1. @2 출력 함수는 인스턴스로(private 클래스 변수) 부터 호출되어 프린트 동작을 수행한다.
    - 사용자들이 프린트를 사용하기 위해서는 getInstance를 호출하여 인스턴스를 리턴 받아 사용하게 된다.
    - getInstance() 메서드에서는 초기에만(== null) 인스턴스를 생성하고 이후에는 계속해서 재사용 하기때문에, 하나라는 조건과 동일하는 조건이 만족된다.
    
#### 사용
```java
public class Main {
    public static void main(String[] args){
        final int MAX = 10;
        
        for(int i = 0; i < MAX; i++){
            User user = new User("name-" + i);
            
            user.printName();
        }
    }
}

class User {
    private String name;
    
    public User(String name) {
        this.name = name;
    }
    
    public void printName(){
        Printer p = new Printer();
        // PrinterSingleton p = PrinterSingleton.getInstance();
        
        System.out.println(p.hashCode());
        p.print(this.name);
    }
}
```

#### 현재 싱글턴 방식의 문제점
멀티 스레드 환경에서 PrinterSingleton 클래스를 이용 할때 인스턴스가 1개 이상 생성되는 경우가 발생할 수 있다.  

- 경합 조건(Race Condition)을 발생시키는 경우
    1. PrinterSingleton 인스턴스가 아직 생성되지 않았을 때 스레드-1이 getPrinter 메서드의 if문을 실행해 이미 인스턴스가 생성되었는지 확인해야 한다. 현재 instance 클래스 변수는 null인 상태
    1. 만약 스레드-1이 생성자를 호출해 인스턴스를 만들기 전 스레드-2가 if문을 싱행해 instance 클래스 변수가 null인지 확인한다. 현재 instance 클래스 변수는 null이므로 인스턴스를 생성하는 생성자를 호출하는 코드를 실행하게 된다.
    1. 스레드-1, 스레드-2 모두 인스턴스를 생성하는 코드를 실행하게 되면 결과적으로 PrintSingleton 클래스의 인스턴스가 2개가 생성된다.
    
#### 해결 방법
##### Thread safe Lazy initialization
```java
public class PrinterSingleton {
    private static PrinterSingleton instance;
    
    private PrinterSingleton (){}
    
    public static synchronized PrinterSingleton getInstance() {     // @4
        if(instance == null){
            instance = new PrinterSingleton();
        }
        
        return instance;
    }
    
    // ...
}
```  

- 실질 적으로 변경된 부분은 @4에 해당되는 getInstance() 메서드에 synchronized modifier가 붙은 것이다.
- 즉 Java의 synchronized modifier를 사용해 thread-safe 하게 끔 수정 한 것이다.
- 하지만 synchronzied 특성상 비교적 성능저하가 발생하므로 권장하지 않는다.

##### Thread safe Lazy initialization + Double-checked locking
```java
public class PrinterSingleton {
    private volatile static PrinterSingleton instance;
    
    private PrinterSingleton(){}
    
    public static PrinterSingleton getInstance() {      // @4
        if(instance == null){ 
            synchronized (PrinterSingleton.class) {
                if(instance == null){
                    instance = new PrinterSingleton();
                }
            }
        }
        
        return instance;
    }
}
```  

- Thread safe Lazy initialization 방법의 성능 저하를 완화 시킨 방법이다.
- getInstance() 함수를 호출시 마다 synchronized를 사용하는 것이 아니라, 첫번째 if문으로 instance 클래스 변수의 null 여부를 판별 한 후 null일 경우에 두번째 if문을 사용할때 동기화를 시키기 때문에 thread-safe 하면서도 위의 방법보다 성능 저하가 완화 되었다.
- 하지만 이 또한 더 개선할 부분이 남아있다.

##### Initialization on demand holder idiom(Holder에 의한 초기화)
```java
public class PrinterSingleton {
    private PrinterSingleton(){}
    
    private static class LazyHolder {
        public static final PrinterSingleton INSTANCE = new PrinterSingleton();
    }
    
    public static PrinterSingleton getInstance(){
        return LazyHolder.INSTANCE;
    }
}
```  

- 이 방법은 중첩 클래스를 이용한 Holder를 사용하는 방법이다. getInstance() 메서드가 호출되기 전까지는 PrinterSingleton 인스턴스는 생성되지 않는다.
- getInstance() 메서드가 호출되는 시점에 LazyHolder 가 참조되고 그 때 PrinterSingleton 인스턴스가 생성된다.
- Lazy initialization 기법으로 메모리 점유율 면에서 유리하다.
- synchronzied 사용에 따른 성능 저하도 없다.
- 이 방법은 JVM의 클래스 초기화 과정에서 보장되는 원자적 특성을 이용하여 싱클턴의 초기화 문제에 대한 책임을 JVM에 떠넘긴 방법이다.
- Holder 안에 선언된 INSTANCE는 클래스 변수이기 때문에 로딩 시점에 한번만 호출 될 것이며, final 키워드를 사용해 다시 값이 할당 되지 않도록 한 방법이다.

##### Enum
```java
public enum PrinterSingleton {
    INSTANCE;
    
    public static PrinterSingleton getInstance() {
        return INSTANCE;
    }
}
```  

- Thread-safe와 Serialization이 보장된다.
- Reflection을 통한 공격에도 안전하다.
    
---
 
## Reference
- [싱글턴 패턴이란](https://gmlwjd9405.github.io/2018/07/06/singleton-pattern.html)
- [싱글톤 패턴(Singleton pattern)을 쓰는 이유와 문제점](https://jeong-pro.tistory.com/86)
- [[DP] 1. 싱글턴 패턴(Singleton pattern)](https://asfirstalways.tistory.com/335)