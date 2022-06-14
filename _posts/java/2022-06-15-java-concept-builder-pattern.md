--- 
layout: single
classes: wide
title: "[Java 개념] 빌더 패턴(Builder Pattern)"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: '매개변수가 많은 객체를 생성할 때 가독성과 효율성을 올릴 수 있는 빌더 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Effective Java
  - Builder Pattern
toc: true 
use_math: true
---  

## 빌더 패턴(Builder Pattern)
앞서 살펴본 [정적 팩토리 메소드]({{site.baseurl}}{% link _posts/java/2022-06-11-java-concept-static-fectory-method.md %})
도 한가지 제약 사항이 있는데, 바로 생성자의 매개변수가 많을 수록 대응이 힘들어 진다는 점이다. 
이는 일반적인 생성자를 사용해서 객체를 생성한다면 직면할 수 있는 일반적인 이슈이다.  

`Compueter` 라는 객체를 생성한다고 가정해보자. 
대표적인 속성 값으로는 `cpu`, `memory` 등이 있겠지만, 
좀더 세분화 한다면 필요한 속성은 수십 가지에 이를 것이다. 
하지만 그 수집개의 속성들이 필수적으로 필요한 속성 보다는 선택적으로 필요한 경우가 많다.  

위와 같은 상황처럼 객체를 생성하기 위해 많은 속성 값이 있는데, 
필수와 선택적인 속성으로 나뉜 상황에서 효율적이면서 간편하게 객체를 생성할 수 있는 방법이 바로 `Builder Pattern` 이다.  


### 점증적 생성자 패턴
전증적 생성자 패턴(`telescoping constructor pattern`) 은 
일반적으로 사용하는 생성자 오버로딩이다. 
필수 속성 값을 받는 생성자 하나 그리고 선택 속성값을 받는 n개의 생성자를 두는 방식으로 예시는 아래와 같다.  

```java
public class Computer {
	private String cpu;			// 필수
	private String memory;		// 필수
	private String mainBoard;	// 필수
	private String gpu;			// 선택
	private String cooler;		// 선택

	public Computer(String cpu, String memory, String mainBoard) {
		this(cpu, memory, mainBoard, "", "");
	}

	public Computer(String cpu, String memory, String mainBoard, String gpu) {
		this(cpu, memory, mainBoard, gpu, "");
	}

	public Computer(String cpu, String memory, String mainBoard, String gpu, String cooler) {
		this.cpu = cpu;
		this.memory = memory;
		this.mainBoard = mainBoard;
		this.gpu = gpu;
		this.cooler = cooler;
	}
}

Computer computer1 = new Computer("cpu", "memory", "mainBoard");
Computer computer2 = new Computer("cpu", "memory", "mainBoard", "gpu");
Computer computer3 = new Computer("cpu", "memory", "mainBoard", "gpu", "cooler");
```  

위 예시는 객체 생성에 필요한 대부분 경우의 생성자를 만들어 둔 상태이다. 
그러므로 원하는 생성자를 호출해 객체를 생성할 수 있다.  

하지만 이런 `점증적 생성자 패턴` 은 매개변수가 많아지면 코드 작성과 가독성이 좋지 않다는 점이 있다. 
속성 개수가 20개라고 한다면 필요한 조합에 따라 모든 경우에 생성자를 다시 다 만들어 줘야 하고, 
사용자는 수 많은 생성자 중에서 자신이 사용해야 하는 생성자를 찾아야 할 것이다. 
그리고 매개변수 순서도 맞춰줘야 하는 등의 버그로 이어질 수 있는 요소들이 존재한다.  


### 자바빈즈 패턴
자바빈즈 패턴(`JavaBeans Pattern`) 은 생성자는 기본 생성자만 두고, 
객체 속성 설정은 `setter` 메소드로 수행하는 방법이다. 

```java
public class Computer {
	private String cpu;			// 필수
	private String memory;		// 필수
	private String mainBoard;	// 필수
	private String gpu;			// 선택
	private String cooler;		// 선택

	public Computer() {

	}

	public void setCpu(String cpu) {this.cpu = cpu;}

	public void setMemory(String memory) {this.memory = memory;}

	public void setMainBoard(String mainBoard) {this.mainBoard = mainBoard;}

	public void setGpu(String gpu) {his.gpu = gpu;}

	public void setCooler(String cooler) {this.cooler = cooler;}
}

Computer computer1 = new Computer();
computer1.setCpu("cpu")
computer1.setMemory("memory");

Computer compuet2 = new Computer();
computer2.setGpu("gpu");
computer2.setCooler("cooler");
```  

`자바빈즈 패턴` 에서는 `점증적 생성자 패턴` 의 단점들이 보이지 않는다. 
속성 설정을 위한 `setter` 메소드만 정의해주면, 
사용자는 원하는 `setter` 메소드를 사용해서 객체의 속성값을 설정해 주면된다.  

하지만 `자바빈즈 패턴` 은 또다른 문제점이 존재한다. 
객체 하나를 생성하기 위해 메소드를 여러번 호출해야 하고, 
모든 `setter` 메소드가 호출이 완료되기 전까지는 객체의 일관성이 보장되지 않는다는 점이다.  

`점증적 생성자 패턴` 에서는 그래도 객체 생성을 생성자에서 수행했기 때문에 
생성자만 호출해주면 객체 생성의 일관성이 보장 됐지만, 
`자바빈즈 패턴` 그렇지 못하다. 
그리고 필수 속성 값에 대한 지정도 `setter` 메소드 만으로는 제약이 있고, 
클래스 불편으로 만들 수 없다는 점이 있다.  


### 빌더 패턴
마지막 대안이자 지금까지 발견된 이슈를 해결 하는 방법이 바로 빌더 패턴(`Builder pattern`) 이다. 
`빌더 패턴` 은 필수 속성 값만 생성자를 통해 설정해 빌더 객체를 생성하고, 그 객체에 대해서 `setter` 메소드를 사용해
선택 속성 값들을 설정한 후 `build` 메소드를 통해 실제 객체를 생성하는 방식이다.  

```java
public class Computer {
	private final String cpu;		// 필수
	private final String memory;	// 필수
	private final String mainBoard;	// 필수
	private final String gpu;		// 선택
	private final String cooler;	// 선택

	public Computer(Builder builder) {
		this.cpu = builder.cpu;
		this.memory = builder.memory;
		this.mainBoard = builder.mainBoard;
		this.gpu = builder.gpu;
		this.cooler = builder.cooler;
	}

	public static class Builder {
		// 필수
		private final String cpu;
		private final String memory;
		private final String mainBoard;

		// 선택
		private String gpu = "";
		private String cooler = "";

		public Builder(String cpu, String memory, String mainBoard) {
			this.cpu = cpu;
			this.memory = memory;
			this.mainBoard = mainBoard;
		}

		public Builder gpu(String gpu) {
			this.gpu = gpu;
			return this;
		}

		public Builder cooler(String cooler) {
			this.cooler = cooler;
			return this;
		}

		public Computer build() {
			return new Computer(this);
		}
	}
}

Computer computer1 = new Computer.Builder("cpu", "memory", "mainBoard")
	.build();
Computer computer2 = new Computer.Builder("cpu", "memory", "mainBoard")
	.gpu("gpu")
	.cooler("cooler")
	.build();
```  

`빌더 패턴` 의 `Computer` 클래스는 불편임을 알 수 있다. 
그리고 필수값은 생성자 방식으로, 선택값은 `setter` 방식을 사용해 설정하고 있다. 
그리고 이러한 과정이 메소드 연쇄(`method chaining`) 방식으로 끊김 없이 모든 과정이 한번에 이루어 지는 것을 확인 할 수 있다.  


### 상속 관계에서 빌더 패턴
추가적인 예시로 상속 관계에 있을때 `빌더 패턴` 을 활용하는 방법에 대해 알아본다. 
`빌더 패턴` 은 상속 관계에서도 큰 어려움 없이 기존의 모든 장점들을 발휘 할 수 있다.  

```java
public abstract class Computer {
	private final String cpu;		// 필수
	private final String memory;	// 필수
	private final String mainBoard;	// 필수
	private final String gpu;		// 선택
	private final String cooler;	// 선택

	abstract static class Builder<T extends Builder<T>> {
		// 필수
		private final String cpu;
		private final String memory;
		private final String mainBoard;

		// 선택
		private String gpu;
		private String cooler;

		public Builder(String cpu, String memory, String mainBoard) {
			this.cpu = cpu;
			this.memory = memory;
			this.mainBoard = mainBoard;
		}

		public T gpu(String gpu) {
			this.gpu = gpu;
			return self();
		}

		public T cooler(String cooler) {
			this.cooler = cooler;
			return self();
		}

		public abstract Computer build();

		protected abstract T self();
	}

	public Computer(Builder<?> builder) {
		this.cpu = builder.cpu;
		this.memory = builder.memory;
		this.mainBoard = builder.mainBoard;
		this.gpu = builder.gpu;
		this.cooler = builder.cooler;
	}
}
```  

상속 관계에서 최상위에 위치하는 `Compueter` 는 `빌더 패턴` 과 큰 차이는 없다. 
실제로 객체를 생성하는 `Computer.Builder` 클래스를 유심히 봐야 한다. 
우선 `Computer.Builder` 는 제네릭 타입으로 추상 메소드인 `self` 를 통해 형 변환 없이 기존 메소드 연쇄를 제공 할수 있도록 했다. 

`Computer` 는 하위 클래스로 `Pc` 와 `Notebook` 이 있다. 

```java
class Pc extends Computer {
	private final String keyboard;
	private final String mouse;

	public static class Builder extends Computer.Builder<Builder> {
		private String keyboard;
		private String mouse;

		public Builder(String cpu, String memory, String mainBoard) {
			super(cpu, memory, mainBoard);
		}

		public Builder keyboard(String keyboard) {
			this.keyboard = keyboard;
			return this;
		}

		public Builder mouse(String mouse) {
			this.mouse = mouse;
			return this;
		}

		@Override
		public Pc build() {
			return new Pc(this);
		}

		@Override
		protected Builder self() {
			return this;
		}
	}

	private Pc(Builder builder) {
		super(builder);
		this.keyboard = builder.keyboard;
		this.mouse = builder.mouse;
	}
}


Pc pc = new Pc.Builder("cpu", "memory", "mainBoard")
	.cooler("cooler")
	.keyboard("keyboard")
	.mouse("mouse")
	.build();
```  

```java
class Notebook extends Computer {
	private final String monitor;
	private final String charger;

	public static class Builder extends Computer.Builder<Builder> {
		private final String monitor;
		private String charger;

		public Builder(String cpu, String memory, String mainBoard, String monitor) {
			super(cpu, memory, mainBoard);
			this.monitor = monitor;
		}

		public Builder charger(String charger) {
			this.charger = charger;
			return this;
		}

		@Override
		public Notebook build() {
			return new Notebook(this);
		}

		@Override
		protected Builder self() {
			return this;
		}
	}

	private Notebook(Builder builder) {
		super(builder);
		this.monitor = builder.monitor;
		this.charger = builder.charger;
	}
}

Notebook notebook = new Notebook.Builder("cpu", "memory", "mainBoard", "monitor")
	.cooler("cooler")
	.charger("charger")
	.build();
```  

`Pc.Builder`, `Notebook.Builder` 에서 각 `build` 메소드의 리턴 값만 자신이 생성하는 객체로 변경해 주면된다. 
그렇게 되면 `Pc.Builder` 는 `Pc` 를 반환하고, `Notebook.Builder` 는 `Notebook` 을 반환하게 된다.  

이를 `계층적 빌더` 라고 하고 우리는 이를 통해 생성자와 `setter` 메소드만 사용할 때는 
누릴 수 없었던 다양한 조합의 속성값 설정을 행할 수 있게 됐다. 
이렇게 유연한 `빌더 패턴` 으로 여러 객체를 순회하면서 만들 수 있고, 
빌더에 넘기는 매개변수에 따라 다른 객체도 만들어 낼 수 있다. 
그리고 객체의 타입별로 특정 속성 값을 알아서 설정하는 등의 동작으로도 활용 할 수 있을 것이다.  

물론 `빌더 패턴` 이 장점만 존재하는 것은 아니다. 
지금까지 `빌더 패턴` 의 예제를 보면 기존 `점증적 생성자 패턴`, `자바빈즈 패턴` 에 비해 코드가 월등히 긴 것을 확인 할 수 있다. 
즉 초기 생산비용이 추가로 들어간다느 점은 명시해야 한다. 
하지만 객체 하나를 초반에 설계하고 계속해서 유지보수 하며 사용하게 되면 속성은 계속해서 늘어난다는 점을 기억해야 한다.  



---
## Reference
[Effective Java](https://book.naver.com/bookdb/book_detail.nhn?bid=14097515)  


