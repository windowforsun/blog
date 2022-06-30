--- 
layout: single
classes: wide
title: "[Java 개념] Java 9 Release"
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
  - Java 9
  - Java Release
  - Release
toc: true 
use_math: true
---  

## Java 9 Release 주요 특징 정리

### Modular System - Jigsaw Project
`Jigsaw`(직소) 프로젝트를 기반으로 하는 `Java 9 Module` 시스템은 안정적인 구성과 강력하고 
유연한 캡슐화를 체공한다. 
이를 통해 확장 가능한 플랫폼을 만들고 플랫폼 무결성을 높이며 성능을 향상 시킬 수 있다.  

`Jigsaw` 의 특징은 아래와 같다. 

  - Modular JDK
  - Modular Java Source Code
  - Modular Run-time Images
  - Encapsulate Java Internal APIs
  - Java Platform Module System

여기서 `Module` 이란 아래 2가지로 크게 구분 지을 수 있다. 
- `code` : `type` 을 포함하는 `pagekes`(e.g. `Class`, `Interface`)
- `data` : `resources` 와 다른 종류의 정적 정보

여기서 `Class` 는 `field` 와 `method` 를 포함하고, 
`Package` 는 `Class`, `Enum`, `Interface` 와 설정 파일을 포함한다. 
그리고 `Module` 은 `Package` 와 다른 데이터 자원을 포함하는 것을 의미 한다.  

`Moudle` 을 사용해서 하나의 `Module` 이 다른 `Module` 을 읽고 다른 `Module` 에서 접근 할 수 있는 
방법을 제어하는 가독성과 접근성을 제공한다. `Moudle` 의 종류로는 아래 3가지 종류가 있다. 

- `named module`
- `unamed module`
- `automatic module`

간단한 예제를 진행해 본다. 
예제 진행을 위해 계산기 구현을 위한 아래와 같은 서로 다른 3개의 모듈을 구성한다. 

- `com.calculator.Main`
- `com.operation.plus.PlusOperation`
- `com.operation.minus.MinusOperation`

`com.calculator.Main` 모듈이 아래 2개 모듈을 사용해서 계산기가 최종적으로 구현된다. 
컴파일된 `mymodule` 디렉토리와 소스코드가 존재하는 `src` 의 파일 구성은 아래와 같다.  

```
.
├── mymodule
│   ├── com.calculator
│   │   ├── com
│   │   │   └── calculator
│   │   │       └── Main.class
│   │   └── module-info.class
│   ├── com.operation.minus
│   │   ├── com
│   │   │   └── operation
│   │   │       └── minus
│   │   │           └── MinusOperation.class
│   │   └── module-info.class
│   └── com.operation.plus
│       ├── com
│       │   └── operation
│       │       └── plus
│       │           └── PlusOperation.class
│       └── module-info.class
└── src
    ├── com.calculator
    │   ├── com
    │   │   └── calculator
    │   │       └── Main.java
    │   └── module-info.java
    ├── com.operation.minus
    │   ├── com
    │   │   └── operation
    │   │       └── minus
    │   │           └── MinusOperation.java
    │   └── module-info.java
    └── com.operation.plus
              ├── com
              │   └── operation
              │       └── plus
              │           └── PlusOperation.java
              └── module-info.java
```  

```java
// src/com.operation.plus/module-info.java
module com.operation.plus {
	exports com.operation.plus;
}
```  

```java
// src/com.operation.plus/com/operation/plus/PlusOperation.java
package com.operation.plus;

public class PlusOperation{
	public static int plus(int a, int b) {
		return a + b;
	}
}
```  

```
// src/com.operation.minus/module-info.java
module com.operation.minus {
    exports com.operation.minus;
}
```

```java
// src/com.operation.minus/com/operation/minus/MinusOperation.java
package com.operation.minus;

public class MinusOperation{
	public static int minus(int a, int b) {
		return a - b;
	}
}
```  

```java
// src/com.calculator/module-info.java
module com.calculator {
    requires com.operation.plus;
    requires com.operation.minus;
}
```  

```java
// src/com.calculator/com/calculator/Main.java
package com.calculator;

import com.operation.plus.PlusOperation;
import com.operation.minus.MinusOperation;

public class Main{
	public static void main(String[] args) {
		int a = 5;
		int b = 2;

		System.out.println(String.format("%d + %d = %d", a, b, PlusOperation.plus(a, b)));
		System.out.println(String.format("%d - %d = %d", a, b, MinusOperation.minus(a, b)));
	}
}
```  

아래 명령으로 우선 `plus`, `minus` 모듈을 컴파일 해준다. 
먼저 컴파일 결과 저장을 위해 `mymodule` 디렉토리를 생성한 뒤 컴파일을 수행한다. 

```bash
$ mkdir -p mymodule/com.operation.plus
$ javac -d mymodule/com.operation.plus src/com.operation.plus/module-info.java src/com.operation.plus/com/operation/plus/PlusOperation.java

$ mkdir -p mymodule/com.operation.minus
$ javac -d mymodule/com.operation.minus src/com.operation.minus/module-info.java src/com.operation.minus/com/operation/minus/MinusOperation.java
```  

마지막으로 `plus`, `minus` 모듈을 사용해서 애플리케이션을 구현하는 `calculator` 모듈을 컴파일 한다.  

```bash
$ mkdir -p mymodule/com.calculator
$ javac --module-path mymodule -d mymodule/com.calculator src/com.calculator/module-info.java src/com.calculator/com/calculator/Main.java
```  

아래 명령으로 모듈 경로를 명시해주고 실행하면 결과를 확인 할 수 있다.  

```bash
$ java --module-path mymodule -m com.calculator/com.calculator.Main
5 + 2 = 7
5 - 2 = 3
```  


### New HTTP Client ????

### Process API
example

### Try-With-Resources
example

### Diamond Operation Extension
example

### Interface Private Method
example

### JShell Command Line Tool
PS C:\jdk\jdk-9> cd bin
PS C:\jdk\jdk-9\bin> jshell.exe
|  Welcome to JShell -- Version 11
|  For an introduction type: /help intro

jshell> "abcdefg".substring(2, 3);
$1 ==> "c"

jshell> "abcdefg".substring(2, 4);
$2 ==> "cd"


### JCMD Sub-Commands

### Multi-Resolution Image API

### Variable Handles
skip

### Publisher-Subscribe Framework(Reactive Stream API)

### Unified JVM Logging

### Immutable Collection
example

### Optional to Stream
example

### Improvement CompletableFuture API
example

### Improvement Stream API
example 

### Multiple-release JARs




---
## Reference
[JDK 9](https://openjdk.org/projects/jdk9/)  
[New Features in Java 9](https://www.baeldung.com/new-java-9)  


