--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Command Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: ''
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

### 명령에 포함되는 정보
- 실제 명령을 나타내는 `ConcreteCommand` 에는 명령에 필요한 정보가 포함될 수 있다.
- 구현의 요구사항에 따라 필요한 정보를 포함시켜 `ConcreteCommand` 구현체를 구성할 수 있다.

### 명령 수행 이력 저장
- `Command` 패턴을 사용하면 수행된 명령을 순서대로 저장해서 이후 다양한 방식으로 활용 가능하다.



---
## Reference

	