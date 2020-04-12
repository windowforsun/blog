--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Interpreter Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Interpreter
use_math : true
---  

## Interpreter 패턴이란
- `Interpreter` 는 통역사라는 의미를 가진 것처럼, 언어 문법이나 표현을 클래스화 구조로 이련의 규칙으로 정의된 언어를 패석하는 패턴이다.
- 여기서 언어 문법이나 표현은 `SQL`, `Shell`, 통신 프로토콜, 정규표현식, 배치처리 등으로 정해전 문법 규칙으로 명령을 내리거나 표현할 수 있는 것을 의미한다.

### BNF
- `BNF` 는 연어 문법이나 표현을 구성할 떄 사용하는 표기법으로 메타언어에 속한다.
- `Backus-Naur Form`, `Backus-Normal Form` 의 약자로 언어 문법 등의 정규화 표현에 많이 사용한다.
- `Interpreter` 의 예제 또한 `BNF` 를 통해 구문을 구성하기 때문에 간단하게 설명하도록 한다.
- 표현식의 정의는 아래와 같이 할 수 있다.

	```
	<이름> ::= <표현식>
	<digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
    <hex_letter> ::= A | B | C | D | E | F
    <hex> ::= <digit> | <hex_letter>
	```  
	
- 연산자는 `|`(or), `-`(표현식 제거), `*`(0개 이상), `+`(1개 이상), `?`(Optional) 가 있다.

	```
	<digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
	<even-digit> :: = 0 | 2 | 4 | 6 | 8 
	<odd-digit> ::= <digit> - <event-digit>
	<string> ::= <character>*
	<integer> ::= <digit>+
	<number> ::= <number><integer>
	<oct-digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7
    <oct> ::= 0?<oct-digit>
	```   
	
- `BNF` 구문 트리는 `terminal expression` 과 `nonterminal expression` 으로 구분될 수 있다.
	- `terminal expression` 은 구문의 종착점으로 `<digit>` 이 해당된다. (숫자이외 이후 전개가 없기 때문)
	- `nontermial expression` 은 계속해서 전개 될수 있는 구문을 의미하고 `<number>`, `<string>` 이 해당된다.

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_interpreter_1.png)

- `AbstractExpression` : BNF 구문 트리를 구성하는 노의의 공통적인 인터페이스를 정의한다.
- `TerminalExpression` : BNF 구문 트리 노드 중 `Terminal Expression` 의 역할을 수행하는 클래스이다.
- `NonterminalExpression` : BNF 구문 트리 노드 중 `Nonterminal Expression` 의 역할을 수행하는 클래스이다.
- `Context` : 인터프리터가 구문해석을 실행하기 위한 정보를 제공하는 역할이다.
- `Client` : 구성된 BNF 구문 트리를 조립하고 호출해서 역할을 수행하는 역할을 한다.

### 

---
## Reference

	