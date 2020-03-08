--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Facade Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '복잡한 서브 시스템의 단순한 인터페이스 제공으로 창구 역할을 수행하는 Facade 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Facade
use_math : true
---  

## Facade 패턴이란
- `Facade` 는 프랑스어로 건물의 정면이라는 의미를 가진것과 같이, 복잡하게 구성돼있는 서브 시스템을 높은 레벨의 인터페이스로 랩핑해 외부에 단순한 인터페이스만 제공하는 것을 의미한다.
- 서브 시스템을 사용하기 위해 필요한 인스턴스 구성 및 메소드 호출등을 외부에서 사용하기 쉽게 랩핑해 간단한 인터페이스만 제공하는 것을 의미한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_facade_1.png)

- 패턴의 구성요소
	- `Facade` : 외부로 제공되는 단순한 인터페이스를 제공하는 창구의 역할로 서브 시스템 사용에 대한 부분이 미리 구성돼 있다.
	- `subsystem` : 서브 시스템을 구성하는 패키지로 시스템 구성에 필요한 여러 클래스로 구성돼있다.
	- `Client` : `Facade` 에서 제공하는 단순한 인터페이스를 사용해서 서브 시스템을 사용한다.
- `Facade` 의 역할
	- 복잡한것을 미리 구성해 간단하게 외부에 제공한다.
	- 주의가 필요한 인스턴스 생성, 메소드 호출의 순서에 대해 미리 구성해 둘 수 있다.
	- 외부에 공개되는 클래스, 메소드를 최소화해 사용에 직관성을 높인다.
	- 단순한 인터페이스의 제공으로 외부와의 결합도를 낮춘다. (약한 결합, 유연한 결합)
	- `public` 접근제어자 등 공개되지 말아야할 부분에 대해 보완가능하다.
- `Facade` 패턴의 재귀성
	- `Facade` 패턴으로 구성된 여러 클래스 조합해, `Facade` 패턴으로 구성할 수 있다.
	- 큰 시스템에서 요소요소에 `Facade` 패턴을 적용하며 시스템을 정리할 수 있다.
- 서브 시스템의 설명서 역할을 하는 `Facade` 패턴
	- 서브 시스템을 직접 구성한 개발자라면 시스템의 내용에 대해 숙지하고 있기 때문에 `Facade` 패턴이 필요하지 않을 수 있다.
	- 팀원 구성원들과의 협업을 위해서 `Facade` 패턴으로 단순한 인터페이스를 제공을 하면 보다 효율적인 개발이 가능하다.
		
## 계산기
- `2+3-2*3/3+2-2*2` 와 같은 수식을 입력 받으면 사칙연산에 맞춰 계산한 결과를 반환하는 계산기를 만들어본다.
- 예제는 어떻게 보면 재귀적인 `Facade` 패턴으로 구성돼있다고 할 수 있다.
	- 계산기의 연산자(`Operation`)를 묶어 계산을 수행할수있도록 구성한 `OperationHandler`
	- `OperationHandler` 의 인스턴스를 알맞게 생성해 조합하고 이를 사용할 수 있도록 구성한 `Calculator` 가 해당한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_facade_2.png)

### Parser

```java
public class Parser {
    public static List<String> parsing(String expression, String operation) {
        List<String> list = new LinkedList<>();
        expression = expression.trim();
        StringTokenizer tokenizer = new StringTokenizer(expression, operation);

        while(tokenizer.hasMoreTokens()) {
            ((LinkedList<String>) list).addLast(tokenizer.nextToken());
        }

        if(list.isEmpty()) {
            throw new ExpressionException();
        }

        return list;
    }
}
```  

- `Parser` 는 수식을 알맞게 파싱하는 클래스이다.
- `Facade` 패턴에서 `subsystem` 패키지를 구성하는 클래스이다.
- `parsing()` 메소드느 수식을 표현하는 문자열과 연산자를 의미하는 문자열을 인자값으로 받아 수식을 연산자단위로 짤라진 리스트르 반환한다.

### Operation

```java
public abstract class Operation {
    private String operatorStr;

    public Operation(String operatorStr) {
        this.operatorStr = operatorStr;
    }

    public String getOperatorStr() {
        return this.operatorStr;
    }

    public abstract double operate(double ...nums);
}
```  

- `Operation` 은 계산기에서 사용할 수 있는 연산자를 정의하는 추상 클래스이다.
- `Facade` 패턴에서 `subsystem` 패키지를 구성하는 클래스이다.
- `operationStr` 필드는 연산자를 의미하는 문자열이고, 생성자를 통해 설정된다.
- `operate(nums)` 은 피연산자 여러개를 인자값으로 받아 결과를 리턴하는 추상 메소드이다.


### PlusOperation, MinusOperation, MultiplyOperation, DivisionOperation

```java
public class PlusOperation extends Operation {
    public PlusOperation() {
        super("+");
    }

    @Override
    public double operate(double... nums) {
        double result = 0;
        int size = nums.length;

        for(int i = 0; i < size; i++) {
            result += nums[i];
        }

        return result;
    }
}
```  

```java
public class MinusOperation extends Operation {
    public MinusOperation() {
        super("-");
    }

    @Override
    public double operate(double... nums) {
        double result = nums[0];
        int size = nums.length;

        for(int i = 1; i < size; i++) {
            result -= nums[i];
        }

        return result;
    }
}
```  

```java
public class MultiplyOperation extends Operation {
    public MultiplyOperation() {
        super("*");
    }

    @Override
    public double operate(double... nums) {
        double result = 1;
        int size = nums.length;

        for(int i = 0; i < size; i++) {
            result *= nums[i];
        }

        return result;
    }
}
```  

```java
public class DivisionOperation extends Operation {
    public DivisionOperation() {
        super("/");
    }

    @Override
    public double operate(double... nums) {
        double result = nums[0];
        int size = nums.length;

        for(int i = 1; i < size; i++) {
            result /= nums[i];
        }

        return result;
    }
}
```  

- `PlusOperation`, `MinusOperation`, `MultiplyOperation`, `DivisionOperation` 은 `Operation` 의 구현체로 각 연산자의 구현을 수행하는 클래스이다.
- `Facade` 패턴에서 `subsyste` 패키지를 구성하는 클래스이다.
- 생성자에서 각 하위 클래스가 의미하는 연산자의 문자열을 설정한다.
- `operate(nums)` 또한 각 연산자에 맞춰 구현한다.

### OperationHandler

```java
public class OperationHandler {
    private Operation operation;
    private LinkedList<String> operandList;
    private OperationHandler next;

    public OperationHandler(Operation operation) {
        this.operation = operation;
        this.operandList = new LinkedList<>();
    }

    public OperationHandler setNext(OperationHandler next) {
        this.next = next;
        return next;
    }

    private double calculate() {
        double[] operandArray = new double[this.operandList.size()];
        int index = 0;

        while(!this.operandList.isEmpty()) {
            try {
                operandArray[index++] = Double.parseDouble(this.operandList.removeFirst());
            } catch (NumberFormatException e) {
                throw new ExpressionException();
            }
        }

        return this.operation.operate(operandArray);
    }

    public double expressionProcess(String expression) {
        this.operandList.addAll(Parser.parsing(expression, this.operation.getOperatorStr()));
        double doubleOperand;
        String operand;
        int size = this.operandList.size();

        for(int i = 0; i < size; i++) {
            operand = this.operandList.removeLast();

            try {
                doubleOperand = Double.parseDouble(operand);
            } catch(Exception e) {
                if(this.next == null) {
                    throw new ExpressionException();
                }

                doubleOperand = this.next.expressionProcess(operand);
            }

            this.operandList.addFirst(doubleOperand + "");
        }

        return this.calculate();
    }
}
```  

- `OperationHandler` 는 `Operation` 와 `Parser` 클래스를 사용해 하나의 연산자가 수식을 계산하는 `Facade` 패턴이 적용된 클래스이다.
- `Facade` 패턴에서 `subsystem` 패키지의 구성요소이면서 `Facade` 역할을 수행한다.
- `operation` 필드로 수식을 어떤 연산자로 계산할지 생성자를 통해 설정한다.
- `operandList` 에는 연산자로 계산할 피연자에 대한 여러개의 문자열이 들어간다.
	- `operandList` 는 계산을 수행 할 수 있는 숫자와 파싱이 필요한 수식을 포함할 수 있다.
- `calculate()` 메소드는 `operandList` 에 있는 피연사자를 `Operation` 의 `operate()` 메소드를 통해 계산한다.
- `expressionProcess(expression)` 은 인자값으로 받은 수식을 담당하는 연산자 문자열과 `Parser` 를 통해 파싱해 `operandList` 에 추기하고, 이를 계산할 수 있는 피연산자로 만들어 `calculate()` 를 호출한다.
	- `OperationHandler` 는 [Chain Of Responsibility 패턴]()
	으로 구성돼 있어, 피연산자가 수식일 경우(담당하는 연산자가 아닐 경우) 계속해서 다음 `OperationHandler` 의 `expressionProcess(expression)` 메소드로 수식의 처리를 위임한다.


### Calculator

```java
public class Calculator {
    private static Calculator instance = null;
    private OperationHandler handler;

    private Calculator() {
        OperationHandler plusHandler = new OperationHandler(new PlusOperation());
        OperationHandler minusHandler = new OperationHandler(new MinusOperation());
        OperationHandler multiplyHandler = new OperationHandler(new MultiplyOperation());
        OperationHandler divisionHandler = new OperationHandler(new DivisionOperation());

        this.handler = plusHandler;
        this.handler.setNext(minusHandler).setNext(multiplyHandler).setNext(divisionHandler);

    }

    public static Calculator getInstance() {
        if(instance == null) {
            instance = new Calculator();
        }

        return instance;
    }

    public double calculate(String expression) {
        if(expression.charAt(0) == '-') {
            expression = "0" + expression;
        }

        return this.handler.expressionProcess(expression);
    }
}
```  

- `Calculator` 는 `operation` 패키지를 사용해 수식을 계산하는 계산기를 나타내는 클래스이다.
- `Facade` 패턴에서 `Facade` 역할을 수행한다.
- `operation` 패키지의 연산자관련 인스턴스를 미리 생성하고, 수식 계산에 필요한 관련 처리를 해서 단순한 계산 인터페이스만 외부로 제공한다.
- `Calculator` 는 [Singleton 패턴]()
으로 구성됐다.
- 생성자에서 미리 연산자 우선순위에 따라 `Chain Of Responsibility 패턴` 에 맞게 `OperationHandler` 인스턴스를 구성한다.
	- 사칙연산은 곱하기, 나누기를 먼저 수행하고 더하기 빼기를 수행하는 것이 원칙
	- `더하기 -> 빼기 -> 곱하기 -> 나누기` 순으로 파싱을 수행해서 실제 연산 순서는 `나누기 -> 곱하기 -> 빼기 -> 더히기` 순으로 진행
- `calculate(expression)` 메소드에 수식을 인자값으로 받으면 `OperationHandler` 의 `expressionProcess(expression)` 을 호출해서 구성된 순서에 맞춰 연산을 수행한다.


### 테스트

```java
public class FacadeTest {
    @Test
    public void Calculator_Simple_Plus() {
        // given
        String expression = "1+2";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(3d));
    }

    @Test
    public void Calculator_Simple_Minus() {
        // given
        String expression = "1-2";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(-1d));
    }

    @Test
    public void Calculator_Simple_Multiply() {
        // given
        String expression = "4*4";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(16d));
    }

    @Test
    public void Calculator_Simple_Division() {
        // given
        String expression = "15/3";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(5d));
    }

    @Test
    public void Calculator_Multiple_Plus() {
        // given
        String expression = "1+2+3+4+5+6+7+8+9+10";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(55d));
    }

    @Test
    public void Calculator_Multiple_Minus() {
        // given
        String expression = "10-1-2-3-4-5";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(-5d));
    }

    @Test
    public void Calculator_Multiple_Multiply() {
        // given
        String expression = "1*2*3*4";

        // then
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(24d));
    }

    @Test
    public void Calculator_Multiple_Division() {
        // given
        String expression = "12/2/3";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(2d));
    }

    @Test
    public void Calculator_Combination() {
        // given
        String expression = "2+4*3-4/2+8/2*3";

        // when
        double actual = Calculator.getInstance().calculate(expression);

        // then
        assertThat(actual, is(24d));
    }
}
```  

---
## Reference

	