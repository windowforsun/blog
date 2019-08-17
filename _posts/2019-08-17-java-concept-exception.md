--- 
layout: single
classes: wide
title: "[Java 개념] Java 예외 클래스와 예외 처리"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Exception 의 개념과 종류 및 차이에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Exception
    - Error
    - RuntimeException
---  

## Java 예외 클래스의 구조

![]()

- Java 의 모든 예외클래스는 Object 클래스의 자식인 Throwable 클래스를 상속받고 있다.
- Throwable 클래스의 자식클래스로는 Error 와 Exception 이 있다.

## Error
- Error 는 시스템 레벨의 심각한 수준의 에러를 의미한다.
- 주로 시스템에 변화를 주어 문제를 처리해야 하는 경우가 일반적이다.
- Catch 블록으로 Error 잡더라도 해결할 수 있는 방법이 없다.

## Exception
- Exception 은 개발자의 로직에 대한 예외 상황을 뜻한다.
- Exception 은 크게 Checked Exception 과 Unchecked Exception 으로 나뉜다.
- Unchecked Exception 은 RuntimeException 및 하위 클래스들을 뜻하고, 그 외 RuntimeException 을 상속받지 않는 Exception 클래스는 모두 Checked Exception 이다.

### Checked Exception
- 반드시 예외 처리를 해야한다.
- `try/catch` 문을 통해 예외를 잡아내거나, `throw` 를 통해 메서드 밖으로 예외를 던져야 한다.
- 컴파일 단계에서 명확하게 Exception 체크가 가능하다.
- 트랜잭션 처리에서 Checked Exception 은 roll-back 을 하지 않는다.

### Unchecked Exception
- 명시적인 예외 처리를 하지 않아도 무방하다.
- 실행단계에서 예외 확인이 가능하기 때문에 로직에 개발자 로직에 대한 부분, 예상하지 못한 논리에 의해 발견된다.
- 트랜잭션 처리에서 Unchecked Exception 은 roll-back 처리를 수행한다.


---
## Reference
[Exception, RuntimeException의 개념과 사용 용도](http://blog.naver.com/PostView.nhn?blogId=serverwizard&logNo=220789097495)  
[Java 예외(Exception) 처리에 대한 작은 생각](http://www.nextree.co.kr/p3239/)  
