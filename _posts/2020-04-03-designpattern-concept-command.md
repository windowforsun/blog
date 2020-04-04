--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Command Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '동작수행(명령)을 클래스로 표현해 확장성을 높이고, 다수의 명령을 관리할 수 있는 Command 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Command
use_math : true
---  

## Command 패턴이란
- `Command` 는 명령이라는 의미를 가진 것처럼, 명령을 클래스로 표현해서 명령(동작)수행을 관리할 수 있도록 구성하는 패턴을 의미한다.
- 명령을 관리하는 경우는 아래와 같은 경우가 있을 수 있다.
	- 명령어를 취소한다.
	- 취소한 명령어를 다시 수행한다.
	- 명령어를 모아 재사용한다.(다른 곳에서 똑같이 수행 등)

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_command_1.png)

- `Command` : 명령의 인터페이스를 정의하는 역할로, 명령을 나타낸다.
- `ConcreteCommand` : 구체적인 명령을 나타내면서 `Command` 의 구현체이다.
- `Receiver` : 수신자의 역할로 명령을 실행할 대상, 명령을 받는 대상을 나타내는 클래스이다.
- `Client` : `ConcreteCommand` 를 생성하고, `Receiver` 에게 역할을 할당한다.
- `Invoker` : 명령을 호출하는 역할로, Event 나 특정 동작을 통해 `Command` 에 정의된 메소드를 호출해 명령을 실행한다.

### 패턴의 처리과정

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_command_2.png)

### 명령에 포함되는 정보
- 실제 명령을 나타내는 `ConcreteCommand` 에는 명령에 필요한 정보가 포함될 수 있다.
- 구현의 요구사항에 따라 필요한 정보를 포함시켜 `ConcreteCommand` 구현체를 구성할 수 있다.

### 명령 수행 이력 저장
- `Command` 패턴을 사용하면 수행된 명령을 순서대로 저장해서 이후 다양한 방식으로 활용 가능하다.

## 계산기 만들기
- 계산기에는 더하기, 빼기, 곱하기, 나누기 등의 명령이 있는데, 그중 더하기, 빼기 명령만 가능한 계산기를 만들어 본다.
- 계산기에서 계산하는 방식은 더하기, 빼기에 해당하는 메소드를 호출할 때, 피연산자를 인자값으로 넣어주면 계산을 수행해 나간다.
	- `plus(1)` = $+1$ = 1
	- `plus(2)` = $+1+2$ = 3
	- `minus(1)` = $+1+2-1$ = 2

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_command_3.png)

### Command

```java
public interface Command {
    void doCommand();
    void undoCommand();
}
```  

- `Command` 는 명령 수행에 필요한 메소드가 정의된, 명령을 나타내는 인터페이스이다.
- `Command` 패턴에서 `Command` 역할을 수행한다.
- `doCommand()` 는 명령 실행을 수행하는 메소드이다.
- `undoCommand()` 는 명령 실행을 취소시키는 메소드이다.

### MacroCommand

```java
public class MacroCommand implements Command{
    private LinkedList<Command> undoCommands;
    private LinkedList<Command> redoCommands;

    public MacroCommand() {
        this.undoCommands = new LinkedList<>();
        this.redoCommands = new LinkedList<>();
    }

    public void appendCommand(Command command) {
        if(command != this) {
            this.undoCommands.addLast(command);
        }
    }

    @Override
    public void doCommand() {
        this.undoCommands.peekLast().doCommand();
    }

    @Override
    public void undoCommand() {
        if(!this.undoCommands.isEmpty()) {
            Command command = this.undoCommands.removeLast();
            command.undoCommand();
            this.redoCommands.addFirst(command);
        }
    }

    public void redoCommand() {
        if(!this.redoCommands.isEmpty()) {
            Command command = this.redoCommands.removeFirst();
            command.doCommand();
            this.undoCommands.addLast(command);
        }
    }

    public void clear() {
        this.undoCommands.clear();
        this.redoCommands.clear();
    }
}
```  

- `MacroCommand` 는 `Command` 의 구현체이면서, 명령 집합을 구성하고 이를 컨트롤하는 클래스이다.
- `Command` 패턴에서 `ConcreteCommand` 역할을 수행한다.
- `undoCommands` 는 실행취소에 해당하는 명령어가 차례대로 추가된다.
- `redoCommands` 는 다시실행에 해당하는 명령어가 차례대로 추가된다.
- `appendCommand()` 를 통해 인자값의 명령을 `undoCommands` 의 맨 마지막에 추가하는 메소드이다.
	- `MacroCommand` 또한 `Command` 의 구현체 이기 때문에, 인자값의 인스턴스가 자신이 아닐 경우에만 추가한다.
- `doCommand()` 는 `undoCommands` 필드의 맨 마지막 명령을 실행해 가장 최근에 추가된 명령을 수행하는 메소드이다.
- `undoCommand()` 는 맨 마지막에 실행한 명령을 취소시키는 메소드이다.
    - `undoCommands` 가 비어있지 않은 상태에서만 동작을 수행한다.
    - `undoCommands` 의 맨 마지막 명령의 `undoCommand()` 를 호출하고 삭제한다.
    - `undoCommands` 의 맨 마지막 명령을 `redoCommands` 의 가장 첫 번째에 추가해준다.
- `redoCommand()` 는 가장 마지막에 실행 취소된 명령을 다시 수행하는 메소드이다.
	- `redoCommands` 가 비어있지 않은 상태에서만 동작을 수행한다.
	- `redoCommands` 의 가장 첫 번째 명령의 `doCommand()` 를 호출하고 삭제한다.
	- `redoCommands` 의 가장 첫 번째 명령을 `undoCommands` 의 가장 마지막에 추가해 준다.
- `clear()` 는 관리대상으로 있는 모든 명령을 지우는 메소드이다.

### PlusCommand, MinusCommand

```java
public class PlusCommand implements Command{
    private double operand;
    private Expression expression;

    public PlusCommand(Expression expression, double operand) {
        this.expression = expression;
        this.operand = operand;
    }

    @Override
    public void doCommand() {
        double result = this.expression.getResult() + this.operand;
        this.expression.doResult(result, this.operand, "+");
    }

    @Override
    public void undoCommand() {
        double result = this.expression.getResult() - this.operand;
        this.expression.undoResult(result);
    }
}
```  

```java
public class MinusCommand implements Command {
    private double operand;
    private Expression expression;

    public MinusCommand(Expression expression, double operand) {
        this.expression = expression;
        this.operand = operand;
    }

    @Override
    public void doCommand() {
        double result = this.expression.getResult() - this.operand;
        this.expression.doResult(result, this.operand, "-");
    }

    @Override
    public void undoCommand() {
        double result = this.expression.getResult() + this.operand;
        this.expression.undoResult(result);
    }
}
```  

- `PlusCommand`, `MinusCommand` 는 `Command` 의 구현체 이면서, 각 연산자를 나타내는 클래스이다.
- `Command` 패턴에서 `ConcreteCommand` 역할을 수행한다.
- `operand` 는 연산자에서 사용하는 피연산자를 의미하는 필드이다.
- `expression` 는 연산의 결과를 설정하는 필드이다.
- `doCommand()` 는 연산자에 해당하는 연산을 수행하고 `expression` 에 수행된 연산의 결과를 설정하는 메소드이다.
- `undoCommand()` 는 연산을 상쇄시키는 연산을 수행하고 `expression` 에 취소된 연산의 결과를 설정하는 메소드이다. 

### Expression

```java
public interface Expression {
    void doResult(double result, double operand, String operation);
    void undoResult(double result);
    String getExpressionStr();
    double getResult();
    void clear();
}
```  

- `Expression` 은 연산의 결과와 사용된 수식에 필요한 메소드가 정의된 인터페이스이다.
- `Command` 패턴에서 `Receiver` 역할을 수행한다.
- `doResult()` 는 연산의 결과를 설정하는 메소드이다.
- `undoResult()` 는 실행 취소된 연산을 설정하는 메소드이다.
- `getExpressionStr()` 는 현재까지 사용된 수식을 문자열로 리턴하는 메소드이다.
- `getResult()` 는 현재 연산의 결과를 리턴하는 메소드이다.
-  `clear()` 는 수식과 연산 결과를 초기화 시키는 메소드이다.

### SimpleExpression

```java
public class SimpleExpression implements Expression {
    private double result;
    private LinkedList<Double> operandList;
    private LinkedList<String> operationList;

    public SimpleExpression() {
        this.operandList = new LinkedList<>();
        this.operationList = new LinkedList<>();
    }

    public double getResult() {
        return this.result;
    }

    public void doResult(double result, double operand, String operation) {
        this.result = result;
        this.operandList.addLast(operand);
        this.operationList.addLast(operation);
    }

    public void undoResult(double result) {
        this.result = result;
        this.operandList.removeLast();
        this.operationList.removeLast();
    }

    public String getExpressionStr() {
        StringBuilder builder = new StringBuilder();
        Iterator<Double> operandIter = this.operandList.iterator();
        Iterator<String> operationIter = this.operationList.iterator();

        while(operandIter.hasNext() || operationIter.hasNext()) {
            builder.append(operationIter.next());
            builder.append(operandIter.next());
        }

        return builder.toString();
    }

    public void clear() {
        this.result = 0;
        this.operandList.clear();
        this.operationList.clear();
    }
}
```  

- `SimpleExpression` 은 `Expression` 의 구현체로 간단한 연산 결과와 수식을 나타내는 클래스이다.
- `Command` 패턴에서 `Receiver` 역할을 수행한다.
- 연산의 결과를 나타내는 `result` 필드, 연산에 사용된 피연사자를 저장하는 `operandList` 필드, 연산에 사용된 연산자를 저장하는 `operationList` 필드가 있다.
- `doResult()` 는 인자값으로 받은 연산결과, 피연산자, 연산자를 각 필드에 설정한다.
- `undoResult()` 는 인자값의 연산결과를 설정하고, 피연산자와 연산자를 관리하는 리스트에서 가장 마지막 요소를 지운다.
- `getExpressionStr()` 은 피연산자와 연산자 리스트를 사용해서 수식을 문자열로 만들어 리턴한다.
- `clear()` 은 연산결과와 피연산자, 연산자 리스트를 모두 초기화 시킨다.

### Calculator

```java
public class Calculator {
    private Expression expression;
    private MacroCommand macroCommand;

    public Calculator(Expression expression) {
        this.expression = expression;
        this.macroCommand = new MacroCommand();
    }

    public double calculate(Command command) {
        this.macroCommand.appendCommand(command);
        this.macroCommand.doCommand();
        return this.expression.getResult();
    }

    public double plus(double operand) {
        return this.calculate(new PlusCommand(this.expression, operand));
    }

    public double minus(double operand) {
        return this.calculate(new MinusCommand(this.expression, operand));
    }

    public double undo() {
        this.macroCommand.undoCommand();
        return this.expression.getResult();
    }

    public double redo() {
        this.macroCommand.redoCommand();
        return this.expression.getResult();
    }

    public String getExpression() {
        return this.expression.getExpressionStr();
    }

    public double getResult() {
        return this.expression.getResult();
    }

    public void clear() {
        this.macroCommand.clear();
        this.expression.clear();
    }
}
```  

- `Calculator` 는 계산기를 나타내는 클래스이다.
- `Command` 패턴에서 `Invoker` 역할과 `Client` 역할을 수행한다.
- `expression` 은 계산기에서 현재 결과와 수식을 사용하기 위해 사용되는 필드이다.
- `macroCommand` 는 계산기에서 수행되는 다수의 명령을 관리하기 위해 사용되는 필드이다.
- `calculate()` 는 인자값으로 명령을 받아 `macroCommand` 에 추가하고 명령을 실행한뒤에 `expression` 에서 결과를 가져와 리턴하는 메소드이다.
- `plus()` 는 덧셈 명령(PlusCommand) 의 인스턴스를 생성해서 `calculate()` 의 인자값으로 넘겨 준다.
- `minus()` 는 뺄셈 명령(MinusCommand) 의 인스턴스를 생성해 `calculate()` 의 인자값으로 넘겨 준다.
- `undo()` 는 `macroCommand` 를 사용해서 마지막 연산을 취소하고 `expression` 에서 결과를 받아 리턴한다.
- `redo()` 는 `macroCommand` 를 사용해서 마지막에 취소된 연산을 다시 실행하고 `expression` 에서 결과를 받아 리턴한다.

### Command 패턴의 처리과정

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_command_4.png)

### 테스트

```java
public class CommandTest {

    @Test
    public void Calculator_Plus_Once() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // when
        double actual = calculator.plus(1);

        // then
        assertThat(actual, is(1d));
        assertThat(calculator.getExpression(), is("+1.0"));
    }

    @Test
    public void Calculator_Plus_Multiple() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // when
        calculator.plus(1);
        double actual = calculator.plus(2);

        // then
        assertThat(actual, is(3d));
        assertThat(calculator.getExpression(), is("+1.0+2.0"));
    }

    @Test
    public void Calculator_Minus_Once() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // when
        double actual = calculator.minus(1);

        // then
        assertThat(actual, is(-1d));
        assertThat(calculator.getExpression(), is("-1.0"));
    }

    @Test
    public void Calculator_Minus_Multiple() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // when
        calculator.minus(1);
        double actual = calculator.minus(2);

        // then
        assertThat(actual, is(-3d));
        assertThat(calculator.getExpression(), is("-1.0-2.0"));
    }

    @Test
    public void Calculator_Plus_Minus_Multiple() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // when
        calculator.minus(1);
        calculator.plus(2);
        calculator.minus(3);
        double actual = calculator.plus(4);

        // then
        assertThat(actual, is(2d));
        assertThat(calculator.getExpression(), is("-1.0+2.0-3.0+4.0"));
    }

    @Test
    public void Calculator_UndoList_NotEmpty_Undo() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());
        calculator.plus(1);
        calculator.plus(2);
        calculator.plus(3);

        // when
        calculator.undo();
        double actual = calculator.undo();

        // then
        assertThat(actual, is(1d));
        assertThat(calculator.getExpression(), is("+1.0"));
    }

    @Test
    public void Calculator_RedoList_NotEmpty_Redo() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());
        calculator.plus(1);
        calculator.plus(2);
        calculator.plus(3);
        calculator.undo();
        calculator.undo();

        // when
        calculator.redo();
        double actual = calculator.redo();

        // then
        assertThat(actual, is(6d));
        assertThat(calculator.getExpression(), is("+1.0+2.0+3.0"));
    }

    @Test
    public void Calculator_UndoList_Empty_Undo() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());

        // then
        double actual = calculator.undo();

        // then
        assertThat(actual, is(0d));
        assertThat(calculator.getExpression(), isEmptyString());
    }

    @Test
    public void Calculator_RedoList_Empty_Redo() {
        // given
        Calculator calculator = new Calculator(new SimpleExpression());
        calculator.plus(1);
        calculator.plus(1);

        // then
        double actual = calculator.redo();

        // then
        assertThat(actual, is(2d));
        assertThat(calculator.getExpression(), is("+1.0+1.0"));
    }
}
```  


---
## Reference

	