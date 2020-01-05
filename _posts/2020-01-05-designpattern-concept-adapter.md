--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Adapter Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '서로 다른 두 기능을 연결해서 사용할 수 있도록 해주는 Adapter 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - DesignPattern
tags:
  - DesignPattern
  - Adapter
---  

## Adapter 패턴이란
- 220v 콘센트에 110v 를 사용하기 위해서는 어댑터가 필요하듯이, `Adapter` 패턴은 이미 구성되어 있는 것이 다른 무언가와 맞지 않을 때 중간에 `Adapter` 라는 것을 사용해서 사용가능 하도록 만들어주는 패턴을 뜻한다.
- `Adapter` 패턴은 `Wrapper` 패턴 라고도 불리는데, `Wrapper` 의 의미처럼 무언가를 감싸 다른 용도로 사용 될 수 있도록 한다.
- `Adapter` 패턴은 크게 2가지 방식으로 나뉜다.
	- 클래스에 의한 Adapter(`extneds`) : 기존 것을 상속해서 `Adapter` 역할을 할 수 있도록 구현한다.
	
		![그림 1]({{site.baseurl}}/img/designpattern/2/concept_adapter_1.png)
		
	- 인스턴스에 의한 Adapter(`delegate`) : 기존 것의 동작을 위임해 `Adapter` 역할을 할 수 있도록 구현한다.
	
		![그림 1]({{site.baseurl}}/img/designpattern/2/concept_adapter_2.png)
		
- 패턴의 구성요소
	- `Target` : 필요한 다른 기능(메서드)이 선언된 인터페이스 혹은 추상 클래스이다.
	- `Adaptee` : 기존에 존재하고 있는 기능으로, `Adapter` 의 기반이 되는 클래스이다.
	- `Adapter` : 다른 기능인 `Target` 을 제공할 수 있도록, `Adaptee` 의 기능을 사용해서 구현하는 클래스이다.
	
	
## 제품 가격에 세일 적용하기
- 아래와 같이 스낵과 커피의 가격과 관련된 클래스가 있다.

	```java
	public class Price {
		private int snackPrice;
		private int coffeePrice;
	
		public Price(int snackPrice, int coffeePrice) {
			this.snackPrice = snackPrice;
			this.coffeePrice = coffeePrice;
		}
	
		public int getSnackPrice() {
			// 스낵에 붙는 세금을 더해준다.
			return this.snackPrice + 10;
		}
	
		public int getCoffeePrice() {
			// 커피에 붙는 세금을 더해준다.
			return this.coffeePrice + 5;
		}
	}
	```  
	
	- `Adapter` 패턴에서 `Adaptee` 역할을 한다.
	- `getSnackPrice()` 메서드에서는 기존 스낵 가격에 세금인 10을 더해서 리턴한다.
	- `getCoffeePrice()` 머세드에서는 기존 커피 가격에 세금인 5를 더해서 리턴한다.
	
- 스낵과 커피 가격에 각기 다른 세일이 적용되었다고 가정했을 때 위 클래스를 아래와 같이 변경 해서 기능을 구현할 수 있다.

	```java
	public class Price {
		private int snackPrice;
		private int coffeePrice;
	
		public Price(int snackPrice, int coffeePrice) {
			this.snackPrice = snackPrice;
			this.coffeePrice = coffeePrice;
		}
	
		public int getSnackPrice() {
			// 스낵에 붙는 세금을 더해준다.
			return this.snackPrice + 10;
		}
	
		public int getCoffeePrice() {
			// 커피에 붙는 세금을 더해준다.
			return this.coffeePrice + 5;
		}
	
		public int getSnackSalePrice() {
			return (int)(this.getSnackPrice() * 0.9);
		}
	
		public int getCoffeeSalePrice() {
			return (int)(this.getCoffeePrice() * 0.8);
		}
	}
	```  
	
	- 스낵에는 10% 할인이 들어가고, 커피는 20% 할인이 들어가는 것을 `Price` 클래스에 메서드로 추가해서 구현했다.
	
- 초기에 제시된 `Price` 클래스는 현재 프로그램을 개발하는 개발자가 구현한 코드가 아닌 라이브러리단에 있는 클래스라고 가정한다.
	- 라이브러리 코드이기 때문에 개발자 임의로 클래스 수정이 불가하다.
	
## 클래스에 의한 Adapter(`extneds`) 로 세일 가격 적용하기

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_adapter_3.png)

### Calculate

```java
public interface Calculator {
    int calculateSnack();
    int calculateCoffee();
}
```  

- `Caculate` 는 새로 필요한 기능을 정의한 인터페이스이다.
- `Adapter` 패턴에서 `Traget` 역할을 한다.
- `calculateSnack()` 메서드에서는 할인이 적용된 스낵가격을 계산한다.
- `calculateCoffee()` 메서드에서는 할인이 적용된 커피가격을 계산한다.

### CalculateSalePrice

```java
public class CalculatorSalePrice extends Price implements Calculator {
    public CalculatorSalePrice(int snackPrice, int coffeePrice) {
        super(snackPrice, coffeePrice);
    }

    @Override
    public int calculateSnack() {
        // 스낵 세일 퍼센트를 적용한다.
        return (int)(this.getSnackPrice() * 0.9);
    }

    @Override
    public int calculateCoffee() {
        // 커피 세일 퍼센트를 적용한다.
        return (int)(this.getCoffeePrice() * 0.8);
    }
}
```  

- `CalculateSalePrice` 는 기존 기능 `Price` 을 상속하고 와 다른 기능 `Calculator` 을 구현해서 최종적으로  다른 기능이 완성될 수 있도록 구현하는 클래스이다.
- `Adapter` 패턴에서 `Adapter` 역할을 한다.
- `Calculator` 인터페이스의 `calculateSnack()` 메서드를 `Price` 클래스의 `getSnackPrice()` 메서드를 사용해서 구현했다.
- `Calculator` 인터페이스의 `calculateCoffee()` 메서드를 `Price` 클래스의 `getCoffeePrice()` 메서드를 사용해서 구현했다.

### 테스트

```java
public class ExtendAdapterTest {
    @Test
    public void Adapter_Snack() {
        // given
        Calculator calculator = new CalculatorSalePrice(100, 200);

        // when
        int actual = calculator.calculateSnack();

        // then
        assertThat(actual, is((int)((100 + 10) * 0.9)));
    }

    @Test
    public void Adapter_Coffee() {
        // given
        Calculator calculator = new CalculatorSalePrice(100, 200);

        // when
        int actual = calculator.calculateCoffee();

        // then
        assertThat(actual, is((int)((200 + 5) * 0.8)));
    }
}
```  

## 인스턴스에 의한 Adapter(`delegate`) 로 세일 가격 적용하기

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_adapter_4.png)

- 위임(delegate) 란 직접 어떠한 것을 처리하지 않고, 다른 무언가에게 맡기는 것을 의미한다.

### Calculator

```java
public abstract class Calculator {
    public abstract int calculateSnack();
    public abstract int calculateCoffee();
}
```  

- `Calculator` 는 다른 기능을 정의한 추상 클래스이다.
- `Adapter` 패턴에서 `Target` 역할을 한다.
- `calculateSnack()` 메서드에서는 할인이 적용된 스낵가격을 계산한다.
- `calculateCoffee()` 메서드에서는 할인이 적용된 커피가격을 계산한다.

### CalculatorSalePrice

```java
public class CalculatorSalePrice extends Calculator {
    private Price price;

    public CalculatorSalePrice(int snackPrice, int coffeePrice) {
        this.price = new Price(snackPrice, coffeePrice);
    }

    @Override
    public int calculateSnack() {
        return (int)(this.price.getSnackPrice() * 0.9);
    }

    @Override
    public int calculateCoffee() {
        return (int)(this.price.getCoffeePrice() * 0.8);
    }
}
```  

- `CalculatorSalePrice` 는 `Calculator` 인터페이스를 통해 다른 기능을 구현하고, 다른 기능은 기존 기능인 `Price` 인터턴스에게 위임해 구현해서 어탭터 기능을 완성하는 클래스이다.
- `Adapter` 패턴에서 `Adapter` 역할을 수행한다.
- `Calculator` 인터페이스의 `calculateSnack()` 메서드를 `Price` 인스턴스의 `getSnackPrice()` 메서드를 사용해서 구현했다.
- `Calculator` 인터페이스의 `calculateCoffee()` 메서드를 `Price` 인스턴스의 `getCoffeePrice()` 메서드를 사용해서 구현했다.

### 테스트

```java
public class DelegateAdapterTest {
    @Test
    public void Adapter_Snack() {
        // given
        Calculator calculator = new CalculatorSalePrice(100, 200);

        // when
        int actual = calculator.calculateSnack();

        // then
        assertThat(actual, is((int)((100 + 10) * 0.9)));
    }

    @Test
    public void Adapter_Coffee() {
        // given
        Calculator calculator = new CalculatorSalePrice(100, 200);

        // when
        int actual = calculator.calculateCoffee();

        // then
        assertThat(actual, is((int)((200 + 5) * 0.8)));
    }
}
```



	