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

![그림 1]({{site.baseurl}}/img/java/concept-exception-01.png)

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

### Checked Exception (Exception)
- 반드시 예외 처리를 해야한다.
- `try/catch` 문을 통해 예외를 잡아내거나, `throw` 를 통해 메서드 밖으로 예외를 던져야 한다.
- 컴파일 단계에서 명확하게 Exception 체크가 가능하다.
- 트랜잭션 처리에서 Checked Exception 은 roll-back 을 하지 않는다.
- Exception 의 하위 클래스로는 아래와 같은 것들이 있다.
	- IOException
	- SQLException
	- ClassNotFoundException
	- InstantiationException
	- CloneNotSupportException
	
### Unchecked Exception (RuntimeException)
- 명시적인 예외 처리를 하지 않아도 무방하다.
- 실행단계에서 예외 확인이 가능하기 때문에 로직에 개발자 로직에 대한 부분, 예상하지 못한 논리에 의해 발견된다.
- 트랜잭션 처리에서 Unchecked Exception 은 roll-back 처리를 수행한다.
- RuntimeException 의 하위 클래스로는 아래와 같은 것들이 있다.
	- NullPointException
	- IllegalArgumentException
	- IndexOutOfBoundException
	- SystemException
	

## 예외 처리

![그림 1]({{site.baseurl}}/img/java/concept-exception-02.jpg)

### 예외 복구

```java
int maxRetryCount = MAX_RETRY_COUNT;

while(maxRetryCount-- > 0) {
	try {
		// 예외가 발생할 수 있는 로직
		
		// 예외가 발생하지 않았으면 리턴
		return;
	} catch(SomeException e) {
		// 로그 및 대기 등 예외상황에 대한 처리
	} finally {
		// 리소스 반납 및 정리 작업
	}
}

// 몇번의 시도에도 실패 한경우 예외 발생
throw new RetryFailException();
```  

- 예외가 발생하더라도 애플리케이션은 정상적인 흐름으로 진행된다.
- 네트워크와 같이 환경이 좋지 않을때 예외가 발생할 수 있는 부분에 적용 가능한 처리이다.
- 재시도 횟수를 정해두고, 재시도 횟수에도 실패한 경우 실패로 간주하고 예외를 던진다.
- 예외가 발생할 경우 예외 상황에 대한 처리 코드와 같은 다름 흐름을 타게해서 예외 처리도 가능하다.

### 예외처리 회피

```java
public void someMethod() throws SomeExeption {
	// ...
}
```  

- 해당 블럭(메서드) 에서 예외를 처리하지 않고 `throws` 를 통해 발생할 수 있는 예외를 호출 한 쪽으로 던진다.
- 간단하지만 해딩 메서드에서 예외를 던지는 것이 최선의 방법이라는 확신이 있을 경우에만 사용해야 한다.

### 예외 전환

```java
try {
	// ..
} catch(SomeExeption e) {
	throw OtherException();
}
```  

- 예외가 발생하면 `try/catch` 문을 통해 예외를 잡아내고 이를 다른 예외로 던지는 방법이다.
- 호출한 쪽에서 예외를 받아 보다 명확하게 인지하게 하기 위함이다.
- Checked Exception 중 복구 불가능한 예외가 발생했을 때, 이를 Unchecked Exception 으로 전환해서 예외 선언이 필요 없도록 할 수도 있다.

---
## Reference
[Exception, RuntimeException의 개념과 사용 용도](http://blog.naver.com/PostView.nhn?blogId=serverwizard&logNo=220789097495)  
[Java 예외(Exception) 처리에 대한 작은 생각](http://www.nextree.co.kr/p3239/)  
