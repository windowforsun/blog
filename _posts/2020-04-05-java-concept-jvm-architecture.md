--- 
layout: single
classes: wide
title: "[Java 개념] JVM 과 JAVA 환경"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java 환경의 구성과 구동환경인 JVM 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - JVM
    - JDK
    - JRE
toc: true
use_math: true
---  

## JAVA 코드의 실행과정

![그림 1]({{site.baseurl}}/img/java/concept_jvm_1.png)

- 사용자가 작성한 소스코드 파일을 `Java Compiler` 를 통해 컴파일 한다.
- `Java Compiler` 는 `.java` 확장자의 파일들을 컴파일을 통해 `.class` 확장자인 `Java byte code` 를 만들어 낸다.
- `Java byte code` 는 기계어가 아니기 때문에 바로 `OS` 환경에서 실행될 수 없다.
- 각 다른 `OS` 에 맞춰 실행될 수 있도록 제공하는 역할을 `JVM` 이 수행한다.

## JAVA 환경

![그림 1]({{site.baseurl}}/img/java/concept-jvm-1.png)

- `JVM`, `JDK`, `JRE` 모두 `JAVA` 를 실행하기 위해 필요한 기반 환경이다.

### JRE

![그림 1]({{site.baseurl}}/img/java/concept_jvm_2.png)

- `JAVA` 애플리케이션을 실행하기 위한 최소 배포 단위를 의미한다.
- `JVM` 과 실행에 필요한 `Library` 들이 포함돼있다.
- `JAVA` 애플리케이션 개발 목적이 아닌, 실행에 목적이 있기 때문에 개발도구는 포함되지 않는다.

### JDK

![그림 1]({{site.baseurl}}/img/java/concept_jvm_3.png)

- `JAVA` 애플리케이션을 실행과 개발을 가능하도록하는 기반환경을 의미한다.
- `JRE` 와 `Compiler`, `Development Tools`, `Debugger` 등을 포함한다.


## JVM 이란

![그림 1]({{site.baseurl}}/img/java/concept_jvm_4.png)

- `JVM` 은 `JAVA Virtual Machine` 의 줄임말로 불리는 용어이다.
- 이름에서 알 수 있듯이 추상적인 컴퓨팅 머신이다.
- 다양한 환경(Windows, Linux)에서 의존성 없이(독립적으로) 사용자의 애플리케이션이(`Java byte code`) 구동가능하도록 `JAVA` 와 `OS` 사이에서 중재자 역할을 수행한다.
- `JVM` 은 중재자 역할을 수행하기 위해서 내부적으로 여러 역할과 모듈로 구성돼, 안정적이면서 지속적으로 애플리케이션이 실행될 수 있도록 제공한다.

### Class Loader

[그림]

- `JAVA` 에서 `Dynamic class loading` 을 담당하는 모듈이다.
- `JVM` 으로 클래스파일(.class) 을 로드하고, 링크를 통해 배치하는 작업을 수행한다.
- `.class` 파일의 참조(클래스 참조)는 컴파일 타임에 수행되는 것이 아니라, 런타임에 수행된다.
- `JAVA` 는 클래스를 처음 참조할 때, 해당 클래스에 대해서 `Class Loader` 가 작업을 수행한다.

#### Loading
- 모든 클래스들은 `Loading` 을 통해 로드된다.
- `BootStrap ClassLoader`, `Extension ClassLoader`, `Application ClassLoader` 세가지 과정을 거치게 된다.
- `BootStrap ClassLoader` : 가장 첫번째로 실행되는 로더로, Bootstrap Classpath 에서 `rt.jar` 이외의 클래스를 로드한다.
- `Extension ClassLoader` : `ext` 디렉토리에 있는 클래스를 로드한다.(jre/lib)
- `Application ClassLoader` : 애플리케이션 레벨에서 사용되는 Classpath, 환경 변수 등을 로드한다.

#### Linking
- `Verity` : `bytecode`(.class) 을 적절한지 검증하고, 그렇지 않으면 에러를 발생시킨다.
- `Prepare` : 정적 변수를 메모리에 할당하고 기본값을 설정한다.
- `Resolve` : 심볼릭 메모리의 참조가 `Method Area` 의 원래 참조로 교체된다.

#### Initialization
- `Class Loader` 의 마지막 단계로, 모든 정적 변수에 값이 설정되고, `static block` 이 실행된다.


### Runtime Data Area

[그림]

- 애플리케이션 구동을 위해 `OS` 로 부터 할당 받은 메모리 공간을 의미한다.
- `Runtime Data Area` 는 5개의 컴포넌트로 구성돼있다.

#### Method Area
- 정적 변수를 포함해서, 모든 클래스 레벨 데이터가 저장되는 공간이다.
- 클래스 정보를 처음 메모리로 로드할 때, 초기화되는 대상을 저장하기 위한 공간이라고 할 수 있다.
- 애플리케이션의 시작인 `main` 메소드를 시작으로, 사실상 컴파일된 대부분은 `bytecode` 가 저장된다고 볼수있다.
- 각 클래스의 구조인 상수(`constant pool`), 필드, 메소드 데이터, 메소드 코드 등이 저장된다.
- `JVM` 에서 `Method Area` 는 단 하나만 존재하고, 멀티 스레드환경에서 공유되기 때문에 `thread-safe` 하지 않다.

#### Heap
- 모든 객체에 해당하는 인스턴스 변수, 배열이 저장되는 공간이다.
- `JVM` 에서 `Heap` 또한 단 하나만 존재하고, 멀티 스레드환경에서 공유되기 때문에 `thread-safe` 하지 않다.
- `Heap` 공간은 이후 `Garbage Collection` 에 의해 관리된다.

#### JVM Language Stacks(Stack Area)
- 각 스레드마다 생성되는 공간으로, 서로 분리된 런타임 스택을 갖는다.
- 모든 메소드 호출시마다 하나의 스택 프레임이 `Stack Area` 에 생성된다.
- 모든 로컬 변수, 리턴 값, 매개변수는 `Stack Area` 에 해당하는 스택 프레임에 생성된다.
- `Stack Area` 에 저장된 값들은 멀티 쓰레드환경에서 공유되지 않기 때문에, `thread-safe` 하다.
- 메소드 실행시에 할당되었다가 메소드가 종료되면서 소멸되는 공간이라고 할 수 있다.
- 스택 프레임(Stack Frame)은 아래 세가지로 구분된다.
	- `Local Variable Array` : 메소드 내에서 사용되는 지역변수의 공간이다.
	- `Operand stack` : 메소드 구동시에 발생되는 피연산자의 공간이다.
	- `Frame data` : 메소드 실행과 종료까지 관련 정보의 공간으로, 예외가 발생할 경우 그 정보가 저장되는 공간이다.


#### PC Registers
- 각 쓰레드에서 실행중인 명령의 주소를 유지하는 공간이고, 쓰레드가 동작으로 수행하며 다음 명령 주소로 갱신된다.
- 쓰레드가 시작될 때 생성되고, 종료되면 소멸되는 공간이다.

#### Native Method Stacks
- 컴파일된 `.class` 파일이 아닌 실제 실행가능한 원시어로(C) 작성된 부분을 실행하는 영역이다.
- `JAVA` 가 아닌 다른 언어(C) 를 실행시키기 위한 공간으로, 쓰레드마다 할당 될 수 있다.
			
### Execution Engine

[그림]

- 컴파일된 `bytecode` 를 `Class Loader` 가 `Runtime Data Area` 에 로드 시키면, `Execution Engine` 에서는 이를 실행시키는 역할을 수행한다.
- `Execution Engine` 은 `bytecode` 를 읽어 하나씩 실행한다.

#### Interpreter
- `bytecode` 를 실행가능한 원시어로 변역해서 실제 명령을 수행한다.
- 번역시 한 줄씩 수행하기 때문에 번역을 빠르지만, 명령 수행은 느릴 수 있다.
- 위와 같은 특징으로 같은 메소드를 여러 곳에서 동시에 호출해 실행해야 될 경우 `Interpreter` 는 호출한 수만큼 반복해서 실행 해야 한다.

#### JIT Compiler
- `Just-In-Time` 의 약자로 `Interpreter` 의 단점을 보완하기 위해 도입된 컴파일러이다.
- `Execution Engine` 은 `Interpreter` 방식으로 명령을 수행하다가, 반복되는 코드가 발견될 경우 이를 `JIT Compiler` 를 통해 전체 `bytecode` 를 원시어로 변환해 명령을 수해 성능을 높이는 방식이다.
- `JIT Compiler` 는 아래 4가지 과정을 거친다.
	- `Intermediate Code Generator` : 중간 코드 생성
	- `Code Optimizer` : 중간 코드 최적화
	- `Target Code Generator` : 기계어나 원시어 생성
	- `Profiler` : `hotspot` 찾기(반복 되는 코드 영역 찾기)

#### Garbage Collector
- 사용되지 않는 객체를 수집하거나, 지우는 역할을 수행한다.
- `System.gc()` 메소드를 통해 사용자 임의로 호출해 동작을 수행할 수 있도록 할 수 있지만, 실제 동작에 대해서는 보장하지 않는다.

### Native Method Interface(Java Native Interface, JNI)
- `Native Method Library` 와 상호작용을 통해 `Execution Engine` 에서 필요한 `Native Library` 를 제공한다.

### Native Method Library
- `Execution Engine` 에서 필요한 `Native Library` 가 위치하는 곳이다.































































































---
## Reference
[Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html)  
[Java Platform Standard Edition 8 Documentation](https://docs.oracle.com/javase/8/docs/)  
[Java Virtual Machine (JVM), Difference JDK, JRE & JVM – Core Java](https://beginnersbook.com/2013/05/jvm/)  
[Java Virtual Machine (JVM) & its Architecture](https://www.guru99.com/java-virtual-machine-jvm.html)  
[The JVM Architecture Explained](https://dzone.com/articles/jvm-architecture-explained)  
[자바 가상 머신](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%B0%94_%EA%B0%80%EC%83%81_%EB%A8%B8%EC%8B%A0)  
