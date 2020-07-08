--- 
layout: single
classes: wide
title: "[Java 개념] Lambda Expressions (람다식)"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Lambda Expression 이란 무엇이고, Java Lambda 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Java
    - Lambda
    - Java8
---  

## Lambda Expressions (람다식) 이란
- 람다식은 수학자 아론조 처치(Alonzo Church) 가 발표한 람다 계산법에서 사용된 식으로, 이를 제자인 존 매카시(John McCarthy) 가 프로그래밍 언어에 도입했다.
- 람다식은 익명함수(anonymous function)를 생성하기 위한 식으로 객체 지향 언어보다는 함수 지향언어에 가깝다.
- Java 는 함수적 프로그래밍을 위해 Java8 부터 람다식(Lambda Exprdessions)을 지원하며 코드 패턴이 많이 달라졌다.

### 람다식의 장점
- 병렬 처리와 이벤트 지향 프로그래밍에 적합하다.
- 객체 지향 프로그래밍과 함수적 프로그래밍을 혼합함으로써 더욱 효율적인 프로그래밍을 할 수 있다.
- 기존 코드가 매우 간결해 진다.
	- 컬렉션의 요소를 필터링, 매핑해서 원하는 결과를 쉽게 도출 해낼 수 있다.
	
### 람다식의 구조
- 람다식의 형태는 매개 변수를 가진 코드 블록이지만, 런타임 시에는 익명 구현 객체를 생성한다.

```
람다식 -> 매개 변수를 가진 코드 블록 -> 익명 구현 객체
```  

- Runnable 인터페이스의 익명 구현 객체를 생성하는 기존의 코드는 아래와 같다.

```java
Runnable runnable = new Runnable() {
	@Override
	public void run() {
		// ...
	}
};
```  

- 이를 람다식으로 작성하면 아래와 같다.

```java
Runnable runnable = () -> {
	// ...
}
```  

- 람다식은 함수 정의와 비슷한 구조를 가지고 있지만, 런타임시 인터페이스의 익명 구현 객체로 생성된다.
- 어떤 인터페이스를 구현할 것인가는 대입되는 인터페이스가 어떤 인터페이스나에 달려 있다.
- 위의 예는 Runnable 변수에 대입되기 때문에 람다식은 Runnable 의 익명 구현 객체를 생성하게 된다.

## 람다식의 기본 문법
- 함수적 스타일의 람다식을 작성하는 방법은 아래와 같다.

```
(타입 매개변수, ...) -> {
	실행문;
	// ...
}
```  

- `(타입 매개변수, ...)` 은 `-> {}` 의 중괄호 블록을 실행하기 위해 필요한 값을 제공하는 역할을 한다.
- int 매개변수 a 의 값을 출력하는 람다식은 아래와 같다.

```java
(int a) -> {
	System.out.println(a);
}
```  

- 매개 변수의 타입은 런타임 시에 대입되는 값에 따라 자동으로 인식 가능하기 때문에 아래와 같이 작성 가능하다.

```java
(a) -> {
	System.out.println(a);	
}
```  

- 위 처럼 하나의 매개 변수만 있다면 `(a)` 괄호를 생락 할 수 있고, 하나의 실행문 만 있다면 `{ }` 중괄호도 생략 가능하다.

```java
a -> System.out.println(a)
```  

- 매개 변수가 없는 경우에는 매개 변수의 자리에 대한 명시가 있어야 하므로 빈 괄로를 써주어야 한다.

```java
( ) -> {
	System.out.println("hellow");
	// 실행문 ...
}
```  

- 람다식을 실행하고 결과 값을 리턴해야 한다면 아래와 같이 작성 가능하다.

```java
(x, y) -> {
	return x + y;
}
```  

- 위와 같이 실행문에 리턴문만 있을 경우, 람다식에서는 return 문을 사용하기 않고 아래와 같이 작성 가능하다.

```java
(x, y) -> x + y
```  

## 타겟 타입과 함수적 인터페이스
- Java 의 람다식은 메소드의 선언과 비슷하다.
	- Java 에서 메소드는 단독으로 선언할 수 없고 항상 클래스의 구성 멤버로 선언된다.
- 람다식 또한 단순히 메소드를 선언하는 것이 아니라, 람다식으로 표현한 메소드를 가지고 있는 객체를 생성한다.

```
인터페이스 변수 = 람다식;
```  

- 작성한 람다식은 위와 같이 인터페이스 변수에 대입된다.
	- 다시 말하면 람다식은 인터페이스의 익명 구현 객체를 생성한다.
- 인터페이스는 직접 객체화가 불가능 하기 때문에 구현 클래스가 필요로 한데, 이를 람다식을 통해 익명 구현 클래스를 생성하고 객체화 한다.
- 람다식이 대입될 인터페이스를 타겟 타입(target type)이라고 한다.

### 함수적 인터페이스(@FunctionalInterface)
- @FunctionalInterface Annotation 은 함수적 인터페이스로 람다식으로 가능한 인터페이스를 체킹해준다.
	- 이 Annotation 붙은 인터페이스에 2개 이상의 추상 메서드가 선언되면 컴파일 오류를 발생시킨다.
- 위와 같은 Annotation 이 있는 이유는 모든 인터페이스를 람다식의 타겟 타입으로 사용할 수 없기 때문이다.
	- 람다식은 인터페이스 중에서도 추상 메소드가 하나인 인터페이스만 타겟 타입이 될 수 있다.
	- 람다식이 하나의 메소드를 정의하기 때문에 두 개 이상의 추상 메소드가 선언된 인터페이스는 람다식을 이용해 구현 객체를 생성할 수 없다.

```java
@FunctionalInterface
public interface MyInerface {
	public void doIt();
	public void doSome();  // 컴파일 오류
}
```  

### 매개 변수와 리턴값이 없는 람다식

```java
@FunctionalInterface
public interface MyInterface {
	public void method();
}
```  

- 위 인터페이스를 타겟 타입으로 갖는 람다식은 아래와 같다.

```java
MyInterface mi = () -> {
	// ...
}
```  

- 람다식에 매개변수가 없는 이유는 인터페이스의 추상 메소드에 매개변수가 없기 때문이다.
- 람다식이 대입된 인터페이스의 참조 변수는 아래와 같이 실행 할 수 있다.

```java
mi.method();
```  

- 예제 코드

```java
public class Main {

    public static void main(String[] args) {
        MyInterface mi;

        mi = () -> {
            String name = "windowforsun";
            System.out.println("hi " + name);
        };

        mi.method();

        mi = () -> {
            System.out.println("Hello world");
        };

        mi.method();
    }

    @FunctionalInterface
    interface MyInterface {
        public void method();
    }
}
```  

```
hi windowforsun
Hello world
```  

### 매개 변수가 있는 람다식

```java
@FunctionalInterface
public interface MyInterface {
	public void method(int x);
}
```  

- 위와 같은 인터페스를 타겟 타입으로 갖는 람다식은 아래와 같이 작성한다.

```java
MyInterface mi = (x) -> {
	// ...
}

// 혹은

MyInterface mi = x -> {
	// ...
}
```  

- 람다식이 대입된 인터페이스의 참조 변수는 아래와 같이 메소드를 호출 할 수 있다.

```java
mi.method(5);
```  

- 예제 코드

```java
public class Main {

    public static void main(String[] args) {
        MyInterface mi;

        mi = (x) -> {
            int result = x * 10;
            System.out.println(result);
        };

        mi.method(5);

        mi = (x) -> {
            System.out.println(x * 10);
        };

        mi.method(5);

        mi = x -> {
            int a = 10;
            System.out.println(x * a);
        };

        mi.method(5);
    }

    @FunctionalInterface
    interface MyInterface {
        public void method(int x);
    }
}
```  

```
50
50
50
``` 

### 리턴값이 있는 람다식

```java
@FunctionalInterface
public interface MyInterface {
	public int method(int x, int y);
}
```  

- 위와 같은 인터페이스를 타겟 타입으로 갖는 람다식은 아래와 같다.

```java
MyInterface mi = (x, y) -> {
	// ...
	return 값;
}
```  

- 실행문에 리턴문 만 있다면 아래와 같이 작성 가능하다.

```java
MyInterface mi = (x, y) -> return x + y;
```  

```java
MyInterface mi = (x, y) -> {
	return sum(x, y);
}

MyInterface mi = (x, y) -> sum(x, y);
```  

- 인터페이스의 참조 변수는 아래와 같이 메소드를 호출 하여 사용할 수 있다.

```java
int result = mi.method(2, 5);
```  

- 예제 코드

```java
public class Main {
    public static void main(String[] args) {
        MyInterface mi;

        mi = (x, y) -> {
            int result = x + y;
            return result;
        };

        System.out.println(mi.method(2, 5));

        mi = (x, y) -> {
            return x + y;
        };

        System.out.println(mi.method(2, 5));

        mi = (x, y) -> x + y;

        System.out.println(mi.method(2, 5));

        mi = (x, y) -> sum(x, y);

        System.out.println(mi.method(2, 5));
    }

    public static int sum(int x, int y) {
        return x + y;
    }

    @FunctionalInterface
    interface MyInterface {
        public int method(int x, int y);
    }
}
```  

```
7
7
7
7
```  

## 클래스 멤버와 로컬 변수 사용
- 람다식의 실행 블록에서는 클래스의 멤버(필드, 메소드) 및 로컬 변수를 사용할 수 았다.
	- 클래스 맴버는 제약 없지만, 로컬 변수는 몇가지 제약이 있다.

### 클래스의 맴버 사용
- 위의 설명 처럼 람다식에서 클래스의 멤버는 제약 없이 사용 가능하다. 하지만 `this` 키워드에는 몇가지 주의 사항이 있다.
- 람다식에서 `this` 는 내부적으로 생성되는 익명 객체의 참조가 아니라 람다식을 실행한 객체의 참조이다.
- 예제 코드

```java
public class Main {
    public int outterValue = 10;

    class InnerClass {
        public int innerValue = 20;

        void method() {
            MyInterface mi = () -> {
                System.out.println("outerValue : " + outterValue);
                System.out.println("outerValue : " + Main.this.outterValue);
                System.out.println("innerValue : " + innerValue);
                System.out.println("innerValue : " + this.innerValue);
            };

            mi.method();
        }
    }

    public static void main(String[] args) {
        Main main = new Main();
        InnerClass innerClass = main.new InnerClass();
        innerClass.method();
    }

    @FunctionalInterface
    interface MyInterface {
        public void method();
    }
}
```  

```
outerValue : 10
outerValue : 10
innerValue : 20
innerValue : 20
```  

- 위 예제는 람다식에서 바깥 객체와 중첩 객체의 참조를 얻어 값을 출력하는 방법을 보여주고 있다.
- 중첩 객체 InnerClass 에서 람다식을 실행했기 때문에 람다식 내부에서의 `this`는 중첩 객체 InnerClass 이다.

### 로컬 변수 사용
- 메소드의 매개변수 또는 로컬 변수를 람다식에서 사용하기 위해서는 `final` 특성을 가져야 한다.
- 매개변수, 로컬 변수를 람다식에서 읽는 것은 가능하지만, 람다식 내부, 외부에서 변경은 불가능 하다.
- 예제 코드

```java
public class Main {
    public void method(int paramValue) {
           int localValue = 40;

           MyInterface mi = () -> {
               System.out.println("paramVallue : " + paramValue);
               System.out.println("localValluee : " + localValue);
           };

           mi.method();
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.method(20);
    }

    @FunctionalInterface
    interface MyInterface {
        public void method();
    }
}
```  

```
paramValue : 20
localValue : 40
```  

## 표준 API 의 함수적 인터페이스
- Java 에서 제공되는 표준 API 중 한 개의 추상 메소드를 가지는 인터페이스는 모두 람다식으로 이용 가능하다.
- 예제 코드

```java
public class Main {
    public static void main(String[] args) {
        Runnable runnable = () -> {
            for(int i = 1; i <= 10; i++) {
                System.out.println(i);
            }
        };

        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```  

```java
public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            for(int i = 1; i <= 10; i++) {
                System.out.println(i);
            }
        });

        thread.start();
    }
}
```  

```
1
2
3
4
5
6
7
8
9
10
```  

- Java8 부터 함수적 인터페이스(functional interface) 를 java.util.function 패키지 표준 API 로 지원한다.
- 위 패키지는 메소드, 생성자의 매개 타입으로 사용되어 람다식을 대입할 수 있도록 하기 위해서이다.
- java.util.function 패키지의 함수적 인터페이스는 크게 Consumer, Supplier, Function, Operator, Predicate 로 구분된다.
- 구분 되는 기준은 인터페이스의 선언된 추상 메소드의 매개값과 리턴값의 유무이다.

종류|추상 메소드 특징|
---|---|
Consumer|매개값은 있고, 리턴값은 없음
Supplier|매개값은 없고, 리턴값은 있음
Function|매개값도 있고, 리턴값도 있음, 주로 매개값을 리턴값으로 매핑(타입변환)
Operator|매개값도 있고, 리턴값도 있음, 주로 매개값을 연산하고 결과를 리턴
Predicate|매개값은 있고, 리턴 타입은 boolean, 매가값을 조사해서 boolean 값 리턴

### Consumer 함수적 인터페이스
- Consumer 의 특징은 리턴값이 없는 accept() 메소드를 가지고 있다.
- accept() 메소드는 매개값을 소비하고 값을 리턴하지 않는다.
- 매개변수 타입에 따른 Consumer 의 종류는 아래와 같다.

인터페이스명|추상메소드|설명
---|---|---
Consumer<T>|void accept(T t)|객체 T를 받아 소비
BiConsumer<T, U>|void accept(T t, U u)|객체 T와 U를 받아 소비
DoubleConsumer|void accept(double value)|double 값을 받아 소비
IntConsumer|void accept(int value)|int 값을 받아 소비
LongConsumer|void accept(long value)|long 값을 받아 소비
ObjDoubleConsumer<T>|void accept(T t, double value)|객체 T와 double 값을 받아 소비
ObjIntConsumer<T>|void accept(T t, int value)|객체 T와 int 값을 받아 소비
ObjLongConsumer<T>|void accept(T t, long value)|객체 T와 long 값을 받아 소비

- 예제코드

```java
public class Main {
    public static void main(String[] args) {
        Consumer<String> consumer = t -> System.out.println(t + " hi");
        consumer.accept("java");

        BiConsumer<String, String> biConsumer = (t, u) -> System.out.println(t + u);
        biConsumer.accept("java", "hi");

        DoubleConsumer doubleConsumer = d -> System.out.println("hi " + d);
        doubleConsumer.accept(1.1);

        ObjIntConsumer<String> objIntConsumer = (t, i) -> System.out.println(t + i);
        objIntConsumer.accept("java", 11);
    }
}
```  

```
java hi
javahi
hi 1.1
java11
```  



























---
## Reference
[이것이 자바다](https://book.naver.com/bookdb/book_detail.nhn?bid=8589375)   

