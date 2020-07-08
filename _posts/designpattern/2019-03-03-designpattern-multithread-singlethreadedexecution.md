--- 
layout: single
classes: wide
title: "[Multi Thread Design Pattern] Single Threaded Execution Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: 'Single Threaded Execution Pattern 이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Multi Thread
    - Single Threaded Execution
---  

# Single Threaded Execution 패턴
- 한 개의 스레드에 의한 실행 이라는 뜻이다.
- 좁은 다디를 하나 번에 한 명만 건널 수 있는 것처럼 한 번에 한 개의 쓰레드만 처리를 실행할 수 있도록 제한을 둔 패턴이다.
- Single Threaded Execution 패턴은 멀티 쓰레드 프로그래밍의 기초가 되니다.
- Critical Section(위험구역) 혹은 Critical Region 이라고 불린다.
- Single Threaded Execution 은 실행하는 쓰레드에 초첨을 맞춘 이름이고, Critical Section, Critical Region 은 실행 범주에 주목한 이름이다.

# 예제
- 한 번에 한 사람 밖에 건널 수 없는 문을 3명이 통과하는 프로그램이다.
- 사람이 문을 톡과할 때마다 그 수를 세고, 또 통과하는 사람의 이름과 출신지를 기록한다.

이름 | 해설 
---|---|
Main | 문을 만들고 3명을 문으로 통과시키는 클래스
Gate | 문을 표현하는 클래스
UserThread | 사람을 나타내는 클래스, 문을 통과한다.

## 패턴을 적용하지 않은 겅우
### Main 클래스
- 문을 만들고 사람을 통과시키는 클래스로 UserThread 클래스 3개를 만들고 start 메서드로 둥작을 수행 시킨다.

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Testing Gate, hit CTRL+C to exit.");
        Gate gate = new Gate();
        new UserThread(gate, "Alice", "Alaska").start();
        new UserThread(gate, "Bobby", "Brazil").start();
        new UserThread(gate, "Chris", "Canada").start();
    }
}

```  

### 쓰레드 세이프가 아닌 Gate 클래스
- Gate 클래스는 사람이 통과하는 문이다.
- counter 필드는 지금까지 통과한 사람수, name 과 address 필드는 마지막으로 문을 통과한 이름과 출신지 이다.
- pass() 메서드는 문을 통과하는 메서드이다.
	- 문을 통과하는 사람의 수인 counter 값을 증가시키고 인수로 주어진 이름과 출신지를 name, address 필드에 복사한다.
- toString() 메서드는 문의 현재 상태를 문자열로 반환한다.
- 사람 name, address 의 첫 문자가 일치하지 않으면 이상이 있다는 증거이다.
- 현재 Gate 클래스는 싱글 쓰레드에서는 문제가 없지만, 멀티 쓰레드에서는 올바르게 동작하지 않는다.

```java
public class Gate {
    private int counter = 0;
    private String name = "Nobody";
    private String address = "Nowhere";
    public void pass(String name, String address) {
        this.counter++;
        this.name = name;
        this.address = address;
        check();
    }
    public String toString() {
        return "No." + counter + ": " + name + ", " + address;
    }
    private void check() {
        if (name.charAt(0) != address.charAt(0)) {
            System.out.println("***** BROKEN ***** " + toString());
        }
    }
}

```  

### UserThread 클래스
- Thread 클래스의 하위 클래스로 문을 통과하는 사람을 표시한다.
