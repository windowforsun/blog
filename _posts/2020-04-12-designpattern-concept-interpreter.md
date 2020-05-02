--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Interpreter Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '언어 문법이나 정해진 표현을 해석하는 Interpreter 패턴에 대해 알아보자'
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
- `Interpreter` 는 통역사라는 의미를 가진 것처럼, 언어 문법이나 표현을 클래스화 시켜 정의된 일련의 규칙으로 해석하는 패턴이다.
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

- `AbstractExpression` : BNF 구문 트리를 구성하는 노드의 공통적인 인터페이스를 정의한다.
- `TerminalExpression` : BNF 구문 트리 노드 중 `Terminal Expression` 의 역할을 수행하는 클래스이다.
- `NonterminalExpression` : BNF 구문 트리 노드 중 `Nonterminal Expression` 의 역할을 수행하는 클래스이다.
- `Context` : 인터프리터가 구문해석을 실행하기 위한 정보를 제공하는 역할이다.
- `Client` : 구성된 BNF 구문 트리를 조립하고 호출해서 역할을 수행하는 역할을 한다.

### Nonterminal expression
- 작성된 표현식의 파싱이 끝나는 표현을 의미하기 때문에, 주의가 필요하다.
- 파싱하는 메소드에서 어느을 읽고 있는지, 어느 부분까지 읽고 종료 해야되는지 파악이 필요하다.

### 구문 트리의 노드의 역할
- 각 구문 트리의 노드는 구성된 언어를 해석아는 역할이고, 해석에 대해서 고유한 역할이 있다.
- 구문 트리의 노드에서 수행하는 역할의 범위는 `BNF` 에 기술된 범위 안에서 처리를 수행해야 한다.

## 계산기 만들기
- 정수를 사용해서 더하기, 빼기만 가능한 간단한 계산기를 만들어 본다.
- `Interpreter` 에서 사용할 수 있는 간단한 계산기 언어도 구성해서 사용한다.

### 계산기 언어

```
<calc> ::= calc <operation expression list>
<operation epxression list> ::= <operand> <operation expression>* end
<operation expression> ::= <operation> <operand>
<operand> ::= <number>
<operation> ::= + | - 
```  

- 계산기 언어를 `BNF` 로 작성하면 위와 같다.
- `<calc>` 는 `calc` 라는 문자열로 언어가 시작하고 뒷 부분에는 수식 리스트(`<operation expression list>`) 가 오는 것을 의미한다.
- `<operation expression list>` 는 한개 피연산자(`<operand>`) 뒤에 0개 이상의 수식(`<operation expression>`) 이 오고, `end` 문자열로 끝나는 것을 의미한다.
- `<operation expression>` 은 하나의 연산자(`<operation>`) 과 하나의 피연산자(`<operand>`) 가 순서대로 있는 것을 의미한다.
- `<operand>` 은 정수인 숫자이고, `<operation>` 은 `+` 더하기, `-` 빼기를 의미하는 문자열이다.
- 계산기 언어의 간단한 예시는 아래와 같다.
	- `calc 1 end` = 1
	- `calc 1 + 1 end` = 2
	- `calc 1 + 1 + 1 end` = 3
	- `calc 1 + 1 + 1 - 2 end` = 3
	- `calc 1 + 1 + 1 - 2 end` = 2
	- `calc 1 + 1 + 1 - 2 - 1 end` = 1
	- `calc 1 + 1 + 1 - 2 - 1 + 1 end` = 2
- 계산기 언어를 구성하는 노드는 공백으로 구분한다.
- 구문 해석을 통해 계산기 언어를 `구문 트리` 형태로 구성하면 아래와 같다.
	- `calc 1 + 1 + 1 end` = 3
	
	![그림 1]({{site.baseurl}}/img/designpattern/2/concept-interpreter-1.png)

### 클래스 다이어그램

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_interpreter_2.png)

### Executor

```java
public interface Executor {
    void execute(Context context);
}
```  

- `Executor` 는 실행에 대한 인테페이스를 나타내는 인터페이스이다.
- `execute()` 메소드는 파싱을 통해 구문 분석된 데이터를 바탕으로 실제 실행하는 역할을 수행한다.

### Node

```java
public interface Node extends Executor {
    void parse(Context context) throws ParseException;
}
```  

- `Node` 는 `BNF` 의 구문 트리를 나타내는 인터페이스이다.
- `Interpreter` 패턴에서 `AbstractExpression` 역할을 수행한다.
- `parse()` 메소드를 통해 하위 구문 트리를 구성하는 노드들에게 구문 해석에 대한 구현을 위임한다.
- `Executor` 의 하위 인터페이스이기 때문에 모든 구문 트리의 노드는 각 역할에 맞춰서 실행할 메소드를 구현해야 한다.
- 모든 구문 트리의 노드는 구문 해석에 실패하면 `ParseException` 이라는 에러를 발싱해시킨다.

### CalcNode

```java
/**
 * <calc> ::= calc <operation expression list>
 */
public class CalcNode implements Node {
    private Node operationExpressionListNode;

    @Override
    public void parse(Context context) throws ParseException {
        context.skipToken("calc");
        this.operationExpressionListNode = new OperationExpressionListNode();
        this.operationExpressionListNode.parse(context);
    }

    @Override
    public void execute(Context context) {
        this.operationExpressionListNode.execute(context);
    }
}
```  

- `Calc` 는 `<calc>` 구문을 해석하고 구문 트리 노드를 나타내는 `Node` 의 구현체 클래스이다.
- `Interpreter` 패턴에서 `NonterminalExpression` 역할을 수행한다.
- `operationExpressionListNode` 는 `<operation expression list>` 구문을 의미하는 필드이다.
- 모든 구문 트리의 노드는 `context` 에서 토큰 단위로 문자열을 가져와 구문 해석을 수행한다.
- `parse()` 는 계산기 언어의 시작인 `calc` 가 있는 경우 다음 토큰으로 건너뛰고, `operationExpressionListNode` 의 인스턴스를 생성하고 `parse()` 를 호출 한다.
- `execute()` 는 `operationExpressionListNode` 의 `execute()` 를 호출해서 해석된 다음 구문을 실행 시키는 역할을 한다.

### OperationExpressionListNode

```java
/**
 * <operation epxression list> ::= <operand> <operation expression>* end
 */
public class OperationExpressionListNode implements Node {
    private Node operandNode;
    private List<Node> list;

    public OperationExpressionListNode() {
        this.list = new LinkedList<>();
    }

    @Override
    public void parse(Context context) throws ParseException {
        this.operandNode = new OperandNode();
        this.operandNode.parse(context);

        while(true) {
            if(context.getCurrentToken() == null) {
                throw new ParseException();
            } else if(context.getCurrentToken().equals("end")){
                context.skipToken("end");
                break;
            } else {
                Node operationExpressionNode = new OperationExpressionNode();
                operationExpressionNode.parse(context);
                this.list.add(operationExpressionNode);
            }
        }
    }

    @Override
    public void execute(Context context) {
        int result = 0;
        this.operandNode.execute(context);
        context.setDefaultResult();

        for(Node node : this.list) {
            node.execute(context);
        }
    }
}
```  

- `OperationExpressionListNode` 는 `<operation expression list>` 구문을 해석하는 `Node` 의 구현체 클래스이다.
- `Interpreter` 패턴에서 `NonterminalExpression` 역할을 수행한다.
- `operandNode` 는 구문 중 `<operand>` 구문을 의미하는 필드이다.
- `list` 는 `<operation expression>*` 구문을 의미하는 필드이다.
- `parse()` 에서는 먼저 `operandNode` 의 구문 해석을 수행하고, `end` 문자열이 나올 때까지 반복해서 `OperationExpressionNode` 의 구문 해석을 수행해서 `list` 필드에 추가한다.
- `execute()` 에서는 `operandNode` 의 처리를 수행하고 `Context` 의 `setDefaultResult()` 를 호출해 결과의 초기값을 설정 한후, `list` 필드의 `OperationExpressionNode` 을 차례대로 수행한다.

### OperationExpression

```java
/**
 * <operation expression> ::= <operation> <operand>
 */
public class OperationExpressionNode implements Node {
    private Node operationNode;
    private Node operandNode;

    @Override
    public void parse(Context context) throws ParseException {
        this.operationNode = new OperationNode();
        this.operationNode.parse(context);
        this.operandNode = new OperandNode();
        this.operandNode.parse(context);
    }

    @Override
    public void execute(Context context) {
        this.operandNode.execute(context);
        this.operationNode.execute(context);
    }
}
```  

- `OperationExpressionNode` 은 `<operation expression>` 구문을 해석하는 `Node` 의 구현체 클래스이다.
- `Interpreter` 패턴에서 `NonterminalExpression` 역할을 수행한다.
- `operationNode` 는 구문 중 `<operation>` 구문을 의미하는 필드이다.
- `operandNode` 는 `<operand>` 구문을 의미하는 필드이다.
- `parse()` 에서는 먼저 `operationNode` 의 구문 해석을 수행하고, `operandNode` 의 구문 해석을 수행한다.
- `execute()` 에서는 먼저 `operandNode` 의 처리를 수행하고, `operationNode` 의 처리를 수행한다.

### OperandNode

```java
/**
 * <operand> ::= <number>
 */
public class OperandNode implements Node {
    private int operand;

    @Override
    public void parse(Context context) throws ParseException {
        this.operand = context.getCurrentNumber();
        context.nextToken();
    }

    @Override
    public void execute(Context context) {
        context.setOperand(this.operand);
    }
}
```  

- `OperandNode` 는 `<operand>` 구문을 해석하는 `Node` 의 구현체 클래스이다.
- `Interpreter` 패턴에서 `TerminalExpression` 역할을 수행한다.
- `operand` 는 해석된 피연사자될 숫자 값을 나타내는 필드이다.
- `parse()` 에서는 `getCurrentNumber()` 을 통해 현재 토큰을 숫자형으로 파싱해서 성공하면 `operand` 에 설정하고, 토큰을 다음 토큰으로 설정한다.
- `execute()` 에서는 인자값 `Context` 의 `operand` 에 해석된 `operand` 를 설정한다.

### OperationNode

```java
/**
 * <operation> ::= "+" | "-"
 */
public class OperationNode implements Node {
    private String operation;

    @Override
    public void parse(Context context) throws ParseException {
        switch (context.getCurrentToken()) {
            case "+":
            case "-":
                this.operation = context.getCurrentToken();
                break;
            default:
                throw new ParseException();
        }
        context.skipToken(this.operation);
    }

    @Override
    public void execute(Context context) {
        int result = context.getResult();

        switch(this.operation) {
            case "+":
                result += context.getOperand();
                break;
            case "-":
                result -= context.getOperand();
                break;

        }

        context.setResult(result);
    }
}
```  

- `Operation` 은 `<operation>` 구문을 해석하는 `Node` 의 구현체 클래스이다.
- `Interpreter` 패턴에서 `TerminalExpression` 역할을 수행한다.
- `operation` 은 해석된 연산자를 나타내는 필드이다.
- `parse()` 는 현재 토큰이 정의된 연산자와 일치할 경우 `operation` 에 문자를 설정하고, 설정된 문자를 `skipToken()` 을 통해 건너뛴다.
- `execute()` 는 해석된 연산자에 맞춰 인자값 `Context` 의 `result` 와 `operand` 의 값으로 수행하고 다시 `result` 에 설정한다.

### Context

```java
public class Context {
    private StringTokenizer tokenizer;
    private String currentToken;
    private int result;
    private int operand;

    public Context(String exp) {
        this.tokenizer = new StringTokenizer(exp);
        this.nextToken();
    }

    public String nextToken() {
        if(this.tokenizer.hasMoreTokens()) {
            this.currentToken = this.tokenizer.nextToken();
        } else {
            this.currentToken = null;
        }

        return this.currentToken;
    }

    public void skipToken(String token) throws ParseException {
        if(!token.equals(this.currentToken)) {
            throw new ParseException();
        }
        this.nextToken();
    }

    public int getCurrentNumber() throws ParseException {
        int number = 0;

        try {
            number = Integer.parseInt(this.currentToken);
        } catch(NumberFormatException e) {
            throw new ParseException();
        }

        return number;
    }

    // getter, setter
}
```  

- `Context` 는 구문해석을 위해 필요한 메소드나, 기능을 위해 필요한 메소드가 구현된 클래스이다.
- `Interceptor` 패턴에서 `Context` 역할을 수행한다.
- `tokenizer` 는 `StringTokenizer` 클래스를 사용해서 입력된 문자열을 구문단위로 나눠 처리할 수 있도록 제공하는 필드이다.
- `currentToken` 은 `tokenizer` 를 통해 현재 처리할 토큰을 저장하는 필드이다.
- `result` 는 계산의 중간 결과나 결과를 저장하는 필드이다.
- `operand` 는 계산해야할 피연산자의 값을 저장하는 필드이다.
- `nextToken()` 는 `tokenizer` 에서 다음 처리할 토큰을 리턴하는 메소드이다.
- `skipToken()` 은 인자값으로 받은 토큰이 현재 토큰과 같으면 다음 스킵하고 다음 `nextToken()` 을 호출한다.
- `getCurrentNumber()` 은 현재 토큰이 숫자일 경우 이를 숫자형식으로 파싱해 리턴한다.
- `setDefaultNumber()` 는 계산 수식에서 첫 피연산자를 설정하는 메소드이다.


### Calculator

```java
public class Calculator {
    private Context context;
    private Node calcNode;

    public int calculate(String exp) throws ParseException{
        this.parse(exp);
        this.calcNode.execute(this.context);

        return this.context.getResult();
    }

    public void parse(String exp) throws ParseException {
        this.context = new Context(exp);
        this.calcNode = new CalcNode();
        this.calcNode.parse(this.context);
    }
}
```  

- `Calculator` 는 `Node` 와 `Context` 를 사용해서 계산기를 나타내는 클래스이다.
- `Interpreter` 패턴에서 `Client` 역할을 수행한다.
- `context` 는 구문 분석해서 사용할 정보를 나타내는 필드이다.
- `calcNode` 는 언어의 시작 노드인 `<calc>` 구문을 나타내는 필드이다.
- `calculate()` 에서 인자값으로 해석할 `exp` 를 받으면 `parse()` 를 호출해서 해석한 후, `calcNode` 의 `execute()` 를 호출해 처리 결과를 리턴한다.
- `parse()` 에서는 `context` 의 인스턴스를 해석할 `exp` 를 전달해 생성하고, `calcNode` 의 인스턴스 생성 및 `context` 를 인자 값을 넘겨 `parse()` 를 호출 한다.


### 테스트

```java
public class InterpreterTest {
    @Test
    public void calculator_SingleNumber_ReturnResult() throws Exception{
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is(1));
    }

    @Test
    public void calculator_SimplePlus_ReturnResult() throws Exception{
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 + 1 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is(2));
    }

    @Test
    public void calculator_MultiplePlus_ReturnResult() throws Exception{
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 + 1 + 2 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is(4));
    }

    @Test
    public void calculator_SimpleMinus_ReturnResult() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 2 - 1 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is(1));
    }

    @Test
    public void calculator_MultipleMinus_ReturnResult() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 2 - 1 - 2 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is( -1));
    }

    @Test
    public void calculator_CombinationOperation_ReturnResult() throws Exception{
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 + 1 + 2 - 3 + 5 - 2 end";

        // when
        int actual = calculator.calculate(exp);

        // then
        assertThat(actual, is(4));
    }

    @Test(expected = ParseException.class)
    public void calculator_NotExpStartCalc_ThrowParseException() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "1 + 1 end";

        // when
        calculator.calculate(exp);
    }

    @Test(expected = ParseException.class)
    public void calculator_NotEndExpEnd_ThrowParseException() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 + 1";

        // when
        calculator.calculate(exp);
    }

    @Test(expected = ParseException.class)
    public void calculator_NotRightOperand_ThrowParseException() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 + end";

        // when
        calculator.calculate(exp);
    }

    @Test(expected = ParseException.class)
    public void calculator_NotLeftOperand_ThrowParseException() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc + 1 end";

        // when
        calculator.calculate(exp);
    }

    @Test(expected = ParseException.class)
    public void calculator_NotOperation_ThrowParseException() throws Exception {
        // given
        Calculator calculator = new Calculator();
        String exp = "calc 1 1 end";

        // when
        calculator.calculate(exp);
    }
}
```  

---
## Reference

	