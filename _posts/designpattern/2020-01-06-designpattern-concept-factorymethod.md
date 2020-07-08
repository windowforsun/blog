--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Factory Method Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: 'Template Method 패턴을 활용해서 인스턴스를 생성하는 Factory Method 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Factory Method
---  

## Factory Method 패턴이란
- [Template Method Pattern]({{site.baseurl}}{% link _posts/designpattern/2020-01-05-designpattern-concept-templatemethod.md %})은 
상위 클래스에서 로직의 템플릿을 구성하고, 하위 클래스에서 템플릿에 맞춰 세부 구현을 구성하는 패턴이다.
- `Factory Method` 패턴은 `Faoctry`(공장) 이라는 의미처럼 인스턴스를 생성하는 공장을 `Template Method` 패턴으로 구성한 것이다.
- `Facotry Method` 패턴은 인스턴스를 만드는 방법을 상위 클래스에서 결정하지만, 구체적인 생성은 하위클래스에서 수행한다.
- 인스턴스 생성을 위한 골격과 실제 인스턴스를 생성하는 클래스를 분리한 패턴이다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_factorymethod_1.png)

- 패턴의 구성요소
	- `framework` 패키지 : 인스턴스 생성에 대한 방식, 골격, 추상적인 내용의 클래스들이 있는 패키지이다.
	- `concrete` 패키지 : 구체적인 인스턴스 생성에 대한 내용과 실제 구현의 클래스들이 있는 패키지 이다.
	- `framework.Product` : 생성되는 인스턴스의 추상적인 내용을 포함하는 추상 클래스이다. 보다 구체적인 인스턴스의 내용은 하위 클래스인 `ConcreteProduct` 에서 결정하게 된다.
	- `framework.Creator` : 인스턴스(`Product`)를 생성하는 골격의 역할을 수행한다. `create()` 에 인스턴스 생성에 대한 기본 틀이 구현되어 있고, `factoryMethod()` 는 하위 클래스에서 구현하는 인스턴스 생성에 대한 세부적인 내용이다.
	- `conrete.ConcreteProduct` : 구체적인 인스턴스의 내용을 정의하는 클래스이다.
	- `concrete.ConcreteCreator` : 구체적인 인스턴스 생성에 대한 내용을 결정하는 클래스이다.

- 하위 클래스에서 구현하는 `factoryMethod()` 중에서 인스턴스 생성 메서드(`createProduct()`)는 아래와 같은 방식으로 구현 가능하다.
	- 추상 메서드의 사용
	
		```java
		abstract class Factory {
			public abstract Product createProduct(String name);
		}
		```  
		
		- `createProduct` 메서드를 상위클래스에서 추상 메서드로 선언하면 하위클래스에서는 필수 적으로 이를 구현해야한다.
		- 본 글에서는 위 방법을 사용해서 패턴을 구성한다.
	- 디폴트 구현의 사용
	
		```java
		class Factory {
			public Product createProduct(String name) {
				return new Product(name);
			}
		}
		```  
		
		- 디폴트 구현을 사용해서 하위 클래스에서 생성의 부분이 다를 경우만, `Override` 를 사용해서 구현 하도록 한다.
		- `Product` 클래스의 인스턴스를 생성해야 하기 때문에 추상 클래스로 둘수 없다.
	- 에러를 이용
	
		```java
		class Factory {
			public Product createProduct(String name) {
				throw new ProductCreateRuntimeException();
			}
		}
		```  
		
		- 디폴트 구현을 다르게 접근한 방식으로, `Override` 하지 않을 경우 에러를 발생 시켜 하위 클래스에서 필수적으로 세부 구현을 하도록 강제한다.
- `Facotry Method` 패턴은 인스턴스 생성에 있어서 생성 방법과 실제 생성을 분리하는데, `framework` 패키지의 클래스들은 `concnrete` 패키지의 클래스에 의존하지 않아 보다 확장성 있는 구조가 가능하다.

## 계산기 공장 만들기
- 어느 기업에서 계산기르 만드는데, 계산기에는 여러 브랜드와 여러 제품 라인이 있다고 가정한다.
- 브랜드에는 `Good`, `Excellent` 가있다.
- 각 브랜드에서 제품라인은 아래와 같다.
	- `Good` : 더하기 계산을 할 수 있는 `Plus`, 곱하기 계산을 할 수 있는 `Multiply`
	- `Excellent` : 더하기 계산을 할 수 있는 `Plus`, 빼기 계산을 할 수 있는 `Minus`

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_factorymethod_2.png)

### framework.Calculator

```java
public abstract class Calculator {
    protected String id;
    protected String brand;
    protected String type;

    public Calculator(String brand, String id, String type) {
        this.brand = brand;
        this.id = id;
        this.type = type;
    }

    public abstract int calculate(int a, int b);

    public String getUid() {
        return this.brand + ":" + this.type + ":" + this.id;
    }
}
```  

- `Calculator` 는 생성하는 인스턴스에 대한 추상적인 내용이 정의된 추상 클래스이다.
- `Factory Method` 패턴에서 `Product` 역할을 수행한다.
- 계산기는 브랜드, 계산기 타입, 아이디로 구성된다.
- `calculate()` 메서드는 하위 클래스의 특성에 따라 구현된다.
- `getUid()` 메서드에서 위 필드를 사용해서 고유한 아이디를 생성한다.
	
### package.CalculatorFactory

```java
public abstract class CalculatorFactory {
    public final Calculator create(String id, String type) {
        Calculator calculator = this.createCalculator(id, type);
        registerCalculator(calculator);

        return calculator;
    }

    protected abstract Calculator createCalculator(String id, String type);
    protected abstract void registerCalculator(Calculator calculator);
}
```  

- `CalculatorFacotry` 는 인스턴스를 생성하는 방법, 골격이 정의된 추상 클래스이다.
- `Factory Method` 패턴에서 `Creator` 역할을 수행한다.
- `createCalculator()` 메서드에서는 하위 클래스의 구현에 따라 인스턴스를 생성한다.
- `registerCalculator()` 메서드에서는 하위 클래스의 구현에 따라 인스턴스를 등록한다.
- `create()` 메서드에는 인스턴스를 생성하는 방법에 대한 내용이 있다.
- `CalculatorFacotry` 는 인스턴스를 생성 할때 `Template Method` 패턴을 사용한다.

### 구체적인 Calculator
- `goodcalculator.PlusCalculator`

	```java
	public class PlusGoodCalculator extends Calculator {
        public PlusGoodCalculator(String id, String type) {
            super("Good", id, type);
        }
    
        @Override
        public int calculate(int a, int b) {
            return a + b;
        }
    }
	```  
	
- `goodcalculator.MultiplyCalculator`

	```java
	public class MultiplyGoodCalculator extends Calculator {
        public MultiplyGoodCalculator(String id, String type) {
            super("Good", id, type);
        }
    
        @Override
        public int calculate(int a, int b) {
            return a * b;
        }
    }
	```  
	
- `excellentcalculator.PlusCalculator`

	```java
	public class PlusExcellentCalculator extends Calculator {	
		public PlusExcellentCalculator(String id, String type) {
			super("Excellent", id, type);
		}
	
		@Override
		public int calculate(int a, int b) {
			return a + b;
		}
	}
	```  
	
- `excellentcalculator.MinusCalculator`

	```java
	public class MinusExcellentCalculator extends Calculator {
		public MinusExcellentCalculator(String id, String type) {
			super("Excellent", id, type);
		}
	
		@Override
		public int calculate(int a, int b) {
			return a - b;
		}
	}
	```  
	
- 구체적인 `Calculator` 는 생성하려는 각 인스턴스에 대한 구체적인 내용이 정의된 클래스이다.
- `Factory Method` 패턴에서 `ConcreteProduct` 역할을 수행한다.
- 브랜드는 각 클래스가 정의된 패키지에 따라 결정된다.
- `calculator()` 메서드 또한 각 클래스의 타입값에 따라 구현되었다.

### 구체적인 Factory
- `goodcalculator.GoodCalculatorFactory`

	```java
	public class GoodCalculatorCalculatorFactory extends CalculatorFactory {
		private List<String> calculatorUids = new ArrayList<>();
	
		@Override
		protected Calculator createCalculator(String id, String type) {
			Calculator calculator = null;
	
			switch(type) {
				case "Plus":
					calculator = new PlusGoodCalculator(id, type);
					break;
				case "Multiply":
					calculator = new MultiplyGoodCalculator(id, type);
					break;
			}
	
			return calculator;
		}
	
		@Override
		protected void registerCalculator(Calculator calculator) {
			this.calculatorUids.add(calculator.getUid());
		}
	
		public List<String> getCalculatorUids() {
			return calculatorUids;
		}
	}
	```  
	
- `excellentcalculator.ExcellentCalculatorFactory`

	```java
	public class ExcellentCalculatorCalculatorFactory extends CalculatorFactory {
		private Map<String, Calculator> calculatorMap = new HashMap<>();
	
		@Override
		protected Calculator createCalculator(String id, String type) {
			Calculator calculator = null;
	
			switch(type) {
				case "Plus":
					calculator = new PlusExcellentCalculator(id, type);
					break;
				case "Minus":
					calculator = new MinusExcellentCalculator(id, type);
					break;
			}
	
			if(calculator != null && this.calculatorMap.containsKey(calculator.getUid())) {
				calculator = this.calculatorMap.get(calculator.getUid());
			}
	
			return calculator;
		}
	
		@Override
		protected void registerCalculator(Calculator calculator) {
			this.calculatorMap.put(calculator.getUid(), calculator);
		}
	
		public Map<String, Calculator> getCalculatorMap() {
			return calculatorMap;
		}
	}
	```  
	
- 구체적인 `Facotry` 는 각 브랜드에 따라 실제 인스턴스를 생성하는 구체적인 구현 내용이 있는 클래스이다.
- `Factory Method` 패턴에서 `ConcreteCreator` 역할을 수행한다.
- `createCalculator()` 메서드에서는 인자 값으로 받은 타입값에 따라 브랜드에 해당하는 인스턴스를 생성하는 구현내용 이다.
- `registerCalculator()` 메서드에서는 생성된 인스턴스를 각 브랜드에 따라 등록에 대한 구현 내용이다.

### 테스트
- `goodcalculator` 테스트

	```java
	public class GoodCalculatorFactoryTest {
		@Test
		public void create() {
			// given
			GoodCalculatorCalculatorFactory goodCalculatorFactory = new GoodCalculatorCalculatorFactory();
			String calculator1Id = "1";
			String calculator2Id = "2";
	
			// when
			Calculator calculator1 = goodCalculatorFactory.create(calculator1Id, "Plus");
			Calculator calculator2 = goodCalculatorFactory.create(calculator2Id, "Multiply");
	
			// then
			List<String> actual = goodCalculatorFactory.getCalculatorUids();
			assertThat(actual, hasSize(2));
			assertThat(actual, hasItems("Good:Plus:1", "Good:Multiply:2"));
			assertThat(calculator1.calculate(2, 1), is(3));
			assertThat(calculator2.calculate(2, 1), is(2));
		}
	}
	```  
	
- `excellentcalculator` 테스트

	```java
	public class ExcellentCalculatorFactoryTest {
        @Test
        public void create() {
            // given
            ExcellentCalculatorCalculatorFactory excellentCalculatorFactory = new ExcellentCalculatorCalculatorFactory();
            String calculator1Id = "1";
            String calculator2Id = "2";
    
            // when
            Calculator calculator1 = excellentCalculatorFactory.create(calculator1Id, "Plus");
            Calculator calculator2 = excellentCalculatorFactory.create(calculator2Id, "Minus");
    
            // then
            Map<String, Calculator> actual = excellentCalculatorFactory.getCalculatorMap();
            assertThat(actual.size(), is(2));
            assertThat(actual, hasKey("Excellent:Plus:1"));
            assertThat(actual, hasKey("Excellent:Minus:2"));
            assertThat(calculator1.calculate(2, 1), is(3));
            assertThat(calculator2.calculate(2, 1), is(1));
        }
    }
	```  
	
	
---
## Reference

	