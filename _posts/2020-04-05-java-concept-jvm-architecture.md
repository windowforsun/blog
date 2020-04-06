--- 
layout: single
classes: wide
title: "[Java 개념] JVM 과 JAVA 환경"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
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
[사진]
`Java source code(.java) -> Java compiler(javac) -> Java byte code(.class) -> JVM -> Interpreter for (Mac, Windows, Linux)`

- 사용자가 작성한 소스코드 파일을 `Java Compiler` 를 통해 컴파일 한다.
- `Java Compiler` 는 `.java` 확장자의 파일들을 컴파일을 통해 `.class` 확장자인 `Java byte code` 를 만들어 낸다.
- `Java byte code` 는 기계어가 아니기 때문에 바로 `OS` 환경에서 실행될 수 없다.
- 각 다른 `OS` 에 맞춰 실행될 수 있도록 제공하는 역할을 `JVM` 이 수행한다.

## JAVA 환경
- `JVM`, `JDK`, `JRE` 모두 `JAVA` 를 실행하기 위해 필요한 기반 환경이다.

### JRE
[사진]
- `JAVA` 애플리케이션을 실행하기 위한 최소 배포 단위를 의미한다.
- `JVM` 과 실행에 필요한 `Library` 들이 포함돼있다.
- `JAVA` 애플리케이션 개발 목적이 아닌, 실행에 목적이 있기 때문에 개발도구는 포함되지 않는다.

### JDK
[사진]
- `JAVA` 애플리케이션을 실행과 개발을 가능하도록하는 기반환경을 의미한다.
- `JRE` 와 `Compiler`, `Development Tools`, `Debugger` 등을 포함한다.



## JVM 이란
[사진]
- `JVM` 은 `JAVA Virtual Machine` 의 줄임말로 불리는 용어이다.
- 이름에서 알 수 있듯이 추상적인 컴퓨팅 머신이다.
- 다양한 환경(Windows, Linux)에서 의존성 없이(독립적으로) 사용자의 애플리케이션이(`Java byte code`) 구동가능하도록 `JAVA` 와 `OS` 사이에서 중재자 역할을 수행한다.
- `JVM` 은 중재자 역할을 수행하기 위해서 내부적으로 여러 기능을 제공하면서 안정적이면서 지속적으로 애플리케이션이 실행될 수 있도록 제공한다.

### Class Loader

[그림]

- `JAVA` 에서 `Dynamic class loading` 을 담당하는 모듈이다.
- `JAVA` 의 `.class` 파일의 참조(클래스 참조)는 컴파일 타임에 수행되는 것이 아니라, 런타임에 수행된다.

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

- `Runtime Data Area` 는 5개의 컴포넌트로 구성돼있다.

#### Method Area
- 정적 변수를 포함해서, 모든 클래스 레벨 데이터가 저장되는 공간이다.
- `JVM` 에서 `Method Area` 는 단 하나만 존재고, 멀티 스레드환경에서 공유되기 때문에 `thread-safe` 하지 않다.

#### Heap
- 모든 객체에 해당하는 인스턴스 변수, 배열이 저장되는 공간이다.
- `JVM` 에서 `Heap` 또한 단 하나만 존재하고, 멀티 스레드환경에서 공유되기 때문에 `thread-safe` 하지 않다.

#### JVM Language Stacks(Stack Area)
- 각 스레드마다 생성되는 공간으로, 서로 분리된 런타임 스택을 갖는다.
- 모든 메소드 호출시마다 하나의 스택 프레임이 `Stack Area` 에 생성된다.
- 모든 로컬 변수는 `Stack Area` 에 해당하는 스택 프레임에 생성된다.
- `Stack Area` 에 저장된 값들은 멀티 쓰레드환경에서 공유되지 않기 때문에, `thread-safe` 하다.
- 스택 프레임(Stack Frame)은 아래 세가지로 구분된다.


#### PC Registers
#### Native Method Stacks
			
### Execution Engine
[그림]
#### Interpreter
#### JIT Compiler
#### Garbage Collector

### Native Method Interface(Java Native Interface, JNI)


### Native Method Library
































































































---
## Reference
[Java Virtual Machine (JVM), Difference JDK, JRE & JVM – Core Java](https://beginnersbook.com/2013/05/jvm/)  
[Java Virtual Machine (JVM) & its Architecture](https://www.guru99.com/java-virtual-machine-jvm.html)  
[The JVM Architecture Explained](https://dzone.com/articles/jvm-architecture-explained)  
[자바 가상 머신](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%B0%94_%EA%B0%80%EC%83%81_%EB%A8%B8%EC%8B%A0)  