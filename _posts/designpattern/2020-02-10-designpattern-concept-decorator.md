--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Decorator Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '내용물과 장식물을 동일시하면서, 내용물(객체) 를 장식(확장)해나가는 Decorator 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Decorator
use_math : true
---  

## Decorator 패턴이란
- Decorator 패턴은 장식이라는 의미를 가진 것과 같이, 객체를 기능적 확장이나 속성의 값을 추가하는 방식으로 장식해 목적에 맞는 객체로 만드는 패턴을 뜻한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_decorator_1.png)

- 패턴의 구성요소
	- `Component` : 객체를 장식할 때 핵심이 되는 추상 클래스 혹은 인터페이스로 장식할 객체가 가질 메소드를 선언한다.
	- `ConcreteComponent` : `Component` 를 구현한 클래스로, 실제로 장식이 되는 객체를 나타내는 클래스이다.
	- `Decorator` : `Component` 를 상속해서 동일한 메소드를 가지면서 장식의 대상인 `Component` 를 필드로 가지는 추상 클래스이다.
	- `ConcreteDecorator` : `Decorator` 를 상속해서 구체적인 장식을 `Component` 의 메소드를 구현해서 나타내는 클래스이다.
- 클래스 구조에서 알 수 있듯이 `Decorator` 패턴은 내용물 `Component` 와 장식물 `Decorator` 를 동일시 해서 객체를 확장해 나간다.
- `Decorator` 패턴의 재귀적 요소
	- [Composite 패턴]({{site.baseurl}}{% link _posts/designpattern/2020-02-08-designpattern-concept-composite.md %})
	와 같이 `Decorator` 패턴을 보면 재귀적인 구조를 가지고 있다.
	- 내용물을 장식하는 장식또한 동일한 내용물인 셈이다. (`ConcreteComponent`, `Decorator` 모두 `Component` 의 하위 클래스)
	- 이는 `Composite` 패턴과 동일해 보일 수 있지만, `Decorator` 패턴의 재귀성은 객체의 기능이나 속성을 확장 및 수정해 목적에 맞게 만들어 간다는 점에 있다.
- `Decorator` 패턴에서 객체의 수정
	- 내용물(`Component`)과 장식물(`Decorator`) 모두 동일한 메소드를 가지고 있지만, 작성된 클래스를 변경하지 않고 계속 장식을 하면서 목적에 맞는 객체로 수정이 가능하다.
	- 장식물은 동작에 대한 부분을 내용물에 위임해서 수행하기 때문에, 지속적인 객체의 확장이 가능하다.
- `Decorator` 패턴은 필요한 장식물을 다양하게 만들어 놓으면 상황에 맞춰 필요한 객체를 구성해서 사용가능 하다.
	- 이는 다른 관점으로 봤을 때, 유사하고 작은 클래스의 수가 많아질 수 있다는 점이 있다.
- `java.io` 패키지는 `Decorator` 패턴으로 잘 구성된 패키지이다.
	- 파일을 읽기 위한 인스턴스는 아래와 같이 만들수 있다.
	
		```java
		Reader reader = new FileReader("file.txt");
		```  
		
	- 파일에서 데이터를 읽을 때 버퍼링을 사용하도록 하는 인스턴스는 아래와 같이 만드 수 있다.
	
		```java
		Reader reader = new BufferedReader(
				new FileReader("file.txt")
		);
		```  
		
	- 파일에서 데이터를 읽을 떄 버퍼링을 사용하고, 줄 번호를 관리할 수 있는 인스턴스는 아래와 같이 만들 수 있다.
	
		```java
        Reader reader = new LineNumberReader(
                new BufferedReader(
                        new FileReader("file.txt")
                )
        );
		```  
		
	- 파일에서 데이터를 읽을 때 줄번호만 관리하는 인스턴스는 아래와 같이 만들 수 있다.
	
		```java
		
        Reader reader = new LineNumberReader(
                new FileReader("file.txt")
        );
		```  
		
	- 파일이 아닌 소켓에서 데이터를 버터링을 사용하고, 줄 번호를 관리할 수 있는 인스턴스는 아래와 같이 만들 수 있다.
	
		```java
		Socket socket = new Socket(host, port);

		
        Reader reader = new LineNumberReader(
                new BufferedReader(
                        new InputStreamReader(
                        	socket.getInputStream()
                        )
                )
        );
		```  
		
	- 위의 예시들과 같이 `java.io` 패키지에서 다양한 방식으로 데이터를 읽는 인스턴스를 만들 수 있다.
	- 이러한 구조 가능 한 것은 모두 `Reader` 클래스의 하위 클래스로 인스턴스를 전달이 가능하기 때문이다.

## 계산기 장식하기
- 연산은 존재하지 않고, 숫자만 입력할 수 있는 계산기가 있다고 가정한다.
- 숫자만 입력할 수 있는 계산기에 연산이라는 장식물로 장식해 계산 결과를 도출해보도록 한다.
- 이 계산기는 우리가 알고 있는 사칙연산과는 다르게 동작한다는 점에 유의해야 한다.
	- 연산은 가장 오른쪽 연산자 부터 왼쪽으로 하나씩 연산해 간다.
	- $5\times2+3=30$ 처럼 $2+3=6$ 을 먼저하고, $5\times6=30$ 연산한다.


![그림 1]({{site.baseurl}}/img/designpattern/2/concept_decorator_2.png)
	
### Calculate

```java
public abstract class Calculator {
    public abstract double calculate();
}
```  

- `Calculate` 는 계산기라는 객체에서 수행해야 하는 메소드를 정의하는 추상클래스이다.
- `Decoratoe` 패턴에서 `Component` 역할을 수행한다.
- `calculate()` 추상 메소드는 계산기의 계산 결과를 리턴한다.

### SingleCalculator

```java
public class SingleCalculator extends Calculator{
    protected double num;

    public SimpleCalculator(double num) {
        this.num = num;
    }

    @Override
    public double calculate() {
        return this.num;
    }
}
```  

- `SingleCalculator` 는 `Calculator` 의 구현체로 하나의 숫자를 입력하는 계산기를 나타내는 클래스이다.
- `Decorator` 패턴에서 `ConcreteComponent` 역할을 수행한다.
- 생성자에서 입력할 숫자를 인자값으로 받아 필드에 설정한다.
- `Calculate` 의 추상 메소드인 `calculate()` 는 입력 받은 숫자를 그대로 리턴하는 것으로 구현했다.

### Operation

```java
public abstract class Operation extends Calculator{
    protected Calculator calculator;
    protected double operand;

    public Operation(double operand, Calculator calculator) {
        this.operand = operand;
        this.calculator = calculator;
    }
}
```  

- `Operation` 는 계산기의 연산자를 나타내는 추상 클래스이다.
- `Decorator` 패턴에서 `Decorator` 역할을 수행한다.
- `Operation` 은 연산자에서 사용할 피연산자와 장식할 내용물을 생정자의 인자값으로 받는다.
- `Operation` 은 `Calculator` 의 하위 클래스이면서, `Calculator` 를 필드로 가지고 있다. 
	- 이는 내용물(`Calculator`) 와 장식물(`Operation`) 을 동일시 하면서, 내용물을 다시 장식(확장)할 수 있다는 의미가 된다.

### PlusOperation, MinusOperation, MultiplyOperation

```java
public class PlusOperation extends Operation {
    public PlusOperation(double operand, Calculator calculator) {
        super(operand, calculator);
    }

    @Override
    public double calculate() {
        return this.operand + this.calculator.calculate();
    }
}
```  

```java
public class MinusOperation extends Operation{
    public MinusOperation(double operand, Calculator calculator) {
        super(operand, calculator);
    }

    @Override
    public double calculate() {
        return this.operand - this.calculator.calculate();
    }
}
```  

```java
public class MultiplyOperation extends Operation {
    public MultiplyOperation(double operand, Calculator calculator) {
        super(operand, calculator);
    }

    @Override
    public double calculate() {
        return this.operand * this.calculator.calculate();
    }
}
```  

- 각 연산자를 나타내는 `PlusOperation`, `MinusOperation`, `MultiplyOperation` 은 `Operation` 의 하위 클래스이면서 `Calculator` 의 추상 메소드를 구현하는 클래스이다.
- `Decorator` 패턴에서 `ConcreteDecorator` 역할을 수행한다.
- `calculate()` 메소드를 의미하는 연산자(`<operate>`)에 맞춰 `operand <operate> calculator.calculate()` 의 방식으로 구현했다.

### 테스트

```java
public class DecoratorTest {
    @Test
    public void calculate_3_plus_5_minus_2_result_6() {
        // given
        SingleCalculator singleCalculator = new SingleCalculator(2);

        // when
        Calculator actual = new PlusOperation(
                3, new MinusOperation(
                        5, singleCalculator
                )
        );

        // then
        assertThat(actual.calculate(), is(6d));
    }

    @Test
    public void calculate_3_minus_5_plus_2_result_minus_4() {
        // given
        SingleCalculator singleCalculator = new SingleCalculator(2);

        // when
        Calculator actual = new MinusOperation(
                3, new PlusOperation(
                        5, singleCalculator
                )
        );

        // then
        assertThat(actual.calculate(), is(-4d));
    }

    @Test
    public void calculate_3_multiply_2_plus_5_minus_4_result_9() {
        // given
        SingleCalculator singleCalculator = new SingleCalculator(4);

        // when
        Calculator actual = new MultiplyOperation(
                3, new PlusOperation(
                        2, new MinusOperation(
                                5, singleCalculator
                        )
                )
        );

        // then
        assertThat(actual.calculate(), is(9d));
    }
}
```  

---
## Reference

	