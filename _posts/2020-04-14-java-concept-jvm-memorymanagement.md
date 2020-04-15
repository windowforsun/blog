--- 
layout: single
classes: wide
title: "[Java 개념] JVM 메모리 관리"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'JVM 의 메모리 구조와 메모리 관리에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
toc: true
use_math: true
---  

## JVM 메모리
- [JVM](https://windowforsun.github.io/blog/java/java-concept-jvm-architecture/)
을 통해 JVM 에 대해서 알아보았다면, JVM 에서 사용되는 메모리에 대해 보다 자세히 알아보도록 한다.
- JVM 은 `main` 메소드가 실행되면 `OS` 에게 설정된 메모리를 할당 받고, 이는 프로그램이 중지될때까지는 변경할 수 없다.
- JVM 에서 생성되는 모든 객체는 JVM 메모리에 저장되고, 이는 주소값 참조를 통해 사용된다.
- JAVA 애플리케이션은 JVM 메모리를 벗어난 공간은 사용할 수 없다.

## 메모리 관리
- JVM 은 아래 3가지 종류로 메모리를 관리한다.

### Heap 
- 애플리케이션에서 고유한 공간으로 동적인 데이터의 공간이다.
- 런타임에 사용되는 모든 클래스의 인스턴스와 배열(Array)의 값들이 저장되는 공간이다.

### Stack
- 애플리케이션에서 Thread 단위로 생성되는 공간이다.
- 실행 중인 메모드 블럭 단위로 로컬 변수와 `Primitive` 타입 값이 저장된다.

### Non-Heap
- 애플리케이션에서 영구적인 데이터가 저장되는 공간이다.
- Class 메타정보, Method 메타정보, 상수 풀, 문자열 인스턴스 등이 저장된다.

## Stack, Heap 의 관계
- 메모리의 영역으로 변수의 값과 관련된 데이터가 저장되는 `Stack` 과 `Heap` 의 관계를 간단하게 그림으로 표현하면 아래와 같다.

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_1.png)

## Stack 영역의 동작
- [JVM-Stack](https://windowforsun.github.io/blog/java/java-concept-jvm-architecture/#jvm-language-stacksstack-area)
을 보면 `Stack` 에는 로컬 변수, 리턴 값, 매개변수 등이 저장된다고 하는데, 더욱 자세히 알아본다.
- `Stack` 은 `Thread` 단위로 할당되는 메모리 공간이다.
- `Stack` 에는 `Heap` 에 생성된 객체의 인스턴스 참조를 위한 값과, `Primitive` 타입의 변수 값이 저장된다.
- 사용되는 모든 변수는 고유한 주소를 가지면서 값을 저장하는데, `Primitive` 타입을 경우 타입에 맞는 값 자체를 저장하고 `Reference` 타입의 경우 인스턴스의 주소를 저장한다.
- `Stack` 에는 로컬 변수가 저장되는데, 이런 로컬 변수는 `visibility` 라는 특성을 가진다.
	- `scope` 와 관련된 개념으로, 구문에서 `{}` 단위로 변수가 사용될 수 있는 범위를 특정 짓는다.
- `Stack` 은 우리가 알고있는 Stack 자료구조를 통해 데이터를 관리하는데, Stack 의 단위는 `Method` 이고 이러한 데이터를 `Stack frame` 이라고 한다.

### 예제 1
- `Stack` 영역에서 `Method` 단위로 `Stack frame` 이 생성되고, `Stack frame` 마다 로컬 변수가 저장되는 과정을 알아보는 예제이다.

```java
public class Main {
	public static void main(String[] args) {
		int mainNum = 10;
		mainNum = methodA(mainNum);
	}
	
	public static int methodA(int param) {
		int methodNum = param + 1;
		
		if(methodNum % 2 == 1) {
			int odd = 1;
		} else {
			int even = 1;
		}
		
		int result = methodNum * 2;
		
		return result;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_2.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_3.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_4.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_5.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_6.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_7.png)


### 예제 2
- 다른 메소드에서 인자값으로 전달 된 `Primitive` 타입의 변수의 값을 수정했을 때 값의 변화를 알아보는 예제이다.

```java
public class Main {
	public static void main(String[] args) {
		int mainNum = 10;
		methodA(mainNum);
	}
	
	public static void methodA(int mainNum) {
		mainNum *= 2;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_8.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_9.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_10.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_11.png)

## Heap 영역의 동작
- `Heap` 영역은 `Stack` 영역과 비교하면 비교적 긴 생명주기를 가진 데이터들이 저장된다.
- 애플리케이션 동작을 위해 사용되는 모든 클래스의 인스턴스값이 저장되는 공간이다.
- 쓰레드 단위로 생성되는 `Stack` 영역과는 달리 애플리케이션에서 고유하게 존재하는 공간이다.
- 이후 예제에서는 아래와 같은 클래스의 인스턴스를 만들어 사용한다.

```java
public class Exam {
	public int num;
	public String str;
	
	public Exam(int num, String str) {
		this.num = num;
		this.str = str;
	}
}
```

### 예제 1
- 기본적으로 `Stack` 에 선언된 변수가 `Heap` 영역에 있는 인스턴스 참조에 대해 알아보는 예제이다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam = new Exam(1, "str1");
		List<Exam> list = new ArrayList<>();
		list.add(new Exam(2, "str2"));
		list.add(new Exam(3, "str3"));
	}
}
```  


![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_12.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_13.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_14.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_15.png)

### 예제 2
- `Stack` 에서 `Heap` 에 생성된 인스턴스를 여러 변수에 설정했을 때 참조에 대해 알아보는 예제이다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam = new Exam(1, "str1");
		Exam exam2 = exam;
		
		methodA(exam);
	}
	
	public static methodA(Exam param) {
		
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_16.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_17.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_18.png)

### 예제 3
- 다른 메소드에서 인자값으로 전달된 인스턴스를 멤버를 수정했을 때의 상황을 알아보는 예제이다.
- `main` 에서 생성한 인스턴스의를 `methodA` 의 인자값으로 전달한 다음 필드를 수정하고, 다시 `main` 으로 되돌아와도 값 변경은 유효하다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam = new Exam(1, "str1");
		methodA(exam);
	}
	
	public static methodA(Exam param) {
		param.num *= 2;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_19.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_20.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_21.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_22.png)

### 예제 4
- 다른 메소드에서 인자값으로 전달된 인스턴스 대신 다른 인스턴스를 설정 했을 때의 예제이다.
- `main` 에서 생성한 인스턴스를 `methodA` 의 인자값으로 전달하고 인스턴스를 다시 생성하더라도, `main` 으로 되돌아오면 인스턴스는 변함없다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam = new Exam(1, "str1");
		methodA(exam);
	}
	
	public static void methodA(Exam param) {
		param = new Exam(2, "str2");
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_23.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_24.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_25.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_26.png)

### 예제 5
- 다른 메소드에서 생성한 인스턴스를 리턴했을 때의 예제이다.
- `main` 에서 인스턴스를 생성한 변수에 `methodA` 에서 생성한 인스턴스를 리턴해서 설정하면, `main` 으로 되돌아 오더라도 변수의 인스턴스는 유효하다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam = new Exam(1, "str1");
		exam = methodA();
	}
	
	public static Exam methodA() {
		Exam tmp = new Exam(2, "str2");
		return tmp;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_27.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_28.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_29.png)

### 예제 6
- 다른 메소드에서 인자값의 두 인스턴스를 `Swap` 했을 때의 예제이다.
- `main` 에서 생성한 두 인스턴스를 `methodA` 의 인자값으로 전달한 다음 인스턴스를 `Swap` 하더라도, `main` 으로 되돌아오면 적용되지 않는다.

```java
public class Main {
	public static void main(String[] args) {
		Exam exam1 = new Exam(1, "str1");
		Exam exam2 = new Exam(2, "str2");
		methodA(exam1, exam2);
	}
	
	public static void methodA(Exam a, Exam b) {
		Exam tmp = a;
		a = b;
		b = tmp;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_30.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_31.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_32.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_33.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_34.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_35.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_36.png)


## Non-Heap
- `Non-Heap` 은 동적인 데이터 보다는 정적인 데이터가 저장되는 여역이다.
- 상수 풀, 문자열 풀, Class 메타 정보, Method 메타 정보 등, 애플리케이션에서 정적인 데이터들이 저장된다.

### 예제 1
- 서로다른 변수에 같은 문자열을 할당 했을 때의 예제이다.
- 각 문자열은 `Immutable Object` 한 특정을 가지면서, 문자열 풀에 저장된다.

```java
public class Main {
	public static void main(String[] args) {
		String a = "hello";
		String b = "hello";
		methodA();
	}
	
	public static void methodA(){
		String methodAStr = "hello";
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_37.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_38.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_39.png)

### 예제 2
- 변수에 설정된 문자열을 수정 할 때의 예제이다.
- 같은 변수에 할당된 문자열을 수정할 때 마다, 새로운 문자열의 인스턴스가 생성되고 참조 또한 변경된다.

```java
public class Main {
	public static void main(String[] args) {
		String a = "hello";
		a += " world";
		a = methodA(a);
	}
	
	public static String methodA(String str) {
		str += "!!";
		return str;
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_40.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_41.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_42.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_43.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_44.png)

### 예제 3
- 다른 메소드에서 인자값으로 받은 문자열을 수정 할때의 예제이다.

```java
public class Main {
	public static void main(String[] args) {
		String a = "hello";
		methodA(a);
	}
	
	public static void methodA(String str) {
		str += " world!!";
	}
}
```

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_45.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_46.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_47.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_48.png)

### 예제 4
- 문자열 타입의 변수에 값을 할당할 때 `Immutable Object` 를 사용한 방법과 `String` 클래스의 인스턴스를 사용한 방법의 예제이다.
- `String` 클래스의 인스턴스를 생성해서 문자열을 할당하면 문자열 풀에 새로운 인스턴스로 생성된다. 

```java
public class Main {
	public static void main(String[] args) {
		String a = "hello";
		String b = new String("hello");
	}
}
```  

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_49.png)

![그림 1]({{site.baseurl}}/img/java/concept_memorymanagement_50.png)


---
## Reference
[Heap and stack space in java](https://www.slideshare.net/btocakci1/heap-and-stack-space-in-java)  
[Java Memory Management for Java Virtual Machine (JVM)](https://betsol.com/java-memory-management-for-java-virtual-machine-jvm/)  