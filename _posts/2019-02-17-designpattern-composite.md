--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 컴퍼지트 패턴(Composite Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
subtitle: '컴퍼지트 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Composite Pattern
---  

'컴퍼지트 패턴이 무엇이고, 어떠한 특징을 가지는지'

### 컴퍼지트 패턴이란
  - 여러 개의 객체들로 구성된 복합 객체와 단일 객체를 클라이언트에서 구별 없이 다루게 해주는 해턴
    - 전체 부분의 관계(ex. Dictionary-File)를 갖는 객체들 사이의 관계를 정의할 때 유용하다.
    - 또한 클라이언트는 전체와 부분을 구분하지 않고 동일한 인터페이스를 사용할 수 있다.
    - [구조 패턴 중 하나]({{site.baseurl}}{% link _posts/2019-02-08-designpattern-intro.md %})
  - ![컴퍼지트 패턴 예시1]({{site.baseurl}}/img/designpattern-composite-ex-1-classdiagram.png)
  - 역할이 수행하는 작업
    - Component
      - 구체적인 부분
      - Leaf 클래스와 전체에 해당하는 Composite 클래스에 공통 인터페이스를 정의
    - Leaf
      - 구체적인 부분 클래스
      - Composite 객체의 부붐으로 설정
    - Composite
      - 전체 클래스
      - 복수 개의 Component를 갖도록 정의
      - 복수 개의 Leaf, 심지어 복수 개의 Composite 객체를 부분으로 가질 수 있음
      
### 예시
#### 컴퓨터에 추가 장치 지원하기
- ![컴퍼지트 패턴 컴퓨터1]({{site.baseurl}}/img/designpattern-composite-computer-1-classdiagram.png)
  - 컴퓨터(Computer 클래스) 모델링
    - 키보드(Keyboard 클래스) : 데이터를 입력받는다.
    - 본체(Body 클래스) : 데이터를 처리한다.
    - 모니터(Monotir 클래스) : 처리 결과를 출력한다.
    - Computer - 합성관계 - 구성장치들

```java
public class Keyboard {
	private int price;
	private int power;
	
	public Keyboard(int power, int price) {
		this.power = power;
		this.price = price;
	}
	
	public int getPrice() {
		return this.price;
	}
	
	public int getPower() {
		return this.power;
	}
}


public class Body {
	private int price;
	private int power;
	
	public Body(int power, int price) {
		this.power = power;
		this.price = price;
	}
	
	public int getPrice() {
		return this.price;
	}
	
	public int getPower() {
		return this.power;
	}
}

public class Monitor {
	private int price;
	private int power;
	
	public Monitor(int power, int price) {
		this.power = power;
		this.price = price;
	}
	
	public int getPrice() {
		return this.price;
	}
	
	public int getPower() {
		return this.power;
	}
}
```  

```java
public class Computer {
	private Keyboard keyboard;
	private Body body;
	private Monitor monitor;
	
	public void addKeyboard(Keyboard keyboard) {
		this.keyboard = keyboard;
	}
	
	public void addBody(Body body) {
		this.body = body;
	}
	
	public void addMonitor(Monitor monitor) {
		this.monitor = monitor;
	}
	
	public int getPrice() {
		int keyboardPrice = this.keyboard.getPrice();
		int bodyPrice = this.body.getPrice();
		int monitorPrice = this.monitor.getPrice();
		
		return keyboardPrice + bodyPrice + monitorPrice;
	}
	
	public int getPower() {
		int keyboardPower = this.keyboard.getPower();
		int bodyPower = this.body.getPower();
		int monitorPower = this.monitor.getPower();
		
		return keyboardPower + bodyPower + monitorPower;
	}
}
```  

```java
public class Client {
	public static void main(String[] args){
		// 컴퓨터의 부품으로 Keyboard, Body, Monitor 객체를 생성
		Keyboard keyboard = new Keyboard(5, 2);
		Body body = new Body(100, 70);
		Monitor monitor = new Monitor(20, 30);
		
		// Computer 객체를 생성하고 부품 객체들을 설정
		Computer computer = new Computer();
		computer.addKeyboard(keyboard);
		computer.addBody(body);
		computer.addMonitor(monitor);
		
		// 컴퓨터의 가격과 전력 소비량을 구함
		int computerPrice = computer.getPrice();
		int computerPower = computer.getPower();
		
		System.out.println("price : " + computerPrice);
		System.out.println("power : " + computerPower);
	}
}
```  

#### 문제점
- 다른 부품이 추가되는 경우
	- Computer 클래스의 부품으로 Speaker 클래스 또는 Mouse 클래스를 추가하려면 ?
	- ![컴포지트 패턴 컴퓨터 문제1]({{site.baseurl}}/img/designpattern-composite-computer-probolem-1-claadiagram.png)
	
```java
public class Speaker {
	private int price;
	private int power;
	
	public Speaker(int power, int price) {
		this.power = power;
		this.price = price;
	}
	
	public int getPrice() {
		return this.price;
	}
	
	public int getPower() {
		return this.power;
	}
}

public class Mouse {
	private int price;
	private int power;
	
	public Mouse(int power, int price) {
		this.power = power;
		this.price = price;
	}
	
	public int getPrice() {
		return this.price;
	}
	
	public int getPower() {
		return this.power;
	}
}
```  

```java
public class Computer {
	// ...
	// 추가
	private Speaker speaker;
	private Mouse mouse;
	
	// ...
	// 추가
	public void addSpeaker(Speaker speaker) {
		this.speaker = speaker;
	}	
	public void addMouse(Mousue mouse){
		this.mouse = mouse;
	}
	
	public int getPrice() {
		// ...
		// 추가
		int speakerPrice = this.speaker.getPrice();
		int mousePrice = this.mouse.getPrice();
		
		return keyboardPrice + bodyPrice + monitorPrice + speakerPrice + mousePrice;
	}
	
	public int getPower() {
		// ...
		// 추가
		int speakerPower = this.speaker.getPower();
		int mousePower = this.mouses.getPower();
		
		return keyboardPower + bodyPower + monitorPower + speakerPower + mousePower;
	}
}
```
	
    