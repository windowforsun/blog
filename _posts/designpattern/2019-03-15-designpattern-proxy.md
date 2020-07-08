--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 프록시 패턴(Proxy Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '프록시 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Proxy Pattern
---  

# 프록시 패턴이란
- 어떤 객체에 대한 접근을 제어하기 위한 용도로 대리인이나 대변인에 해당하는 객체를 제공하는 패턴
- 프록시 패턴의 종류
	- 원격 프록시(Remote Proxy) : 원격 객체에 대한 접근을 제어할 수 있다.
	- 가상 프록시(Virtual Proxy) : 생성하기 힘든 자원에 대한 접근을 제어할 수 있다.
	- 보호 프록시(Protection Proxy) : 접근 권한이 필요한 자원에 대한 접근을 제어할 수 있다.
## 프록세 패턴 디자인 구조
![프록시 패턴 구조]({{site.baseurl}}/img/designpattern/designpattern-proxy-intro-classdiagram.png)
- Subject
	- Proxy 와 RealSubject 모두 Subject 인터페이스를 구현한다.
	- 어떤 클라이언트에서든 프록시를 주 객체로 하고 RealSubject 와 같은 방식으로 사용할 수 있겠다.
- RealSubject
	- 실제 작업을 대부분 처리하는 객체를 나타낸다.
	- Proxt 에서 이 객체에 대한 접근을 제어하는 객체이다.
- Proxy
	- 실제 작업을 처리하는 객체에 대한 레퍼런스가 있다.
	- 실제 객체가 필요하면 그 레퍼런스를 이용해서 요청을 전달한다.
	- Proxy 에서 RealSubject 의 인스턴스를 생성하거나, 객체의 생성 과정에 관여 할 수 있다.
	
# 원격 프록시(Remote Proxy)
뽑기 기계와 모니터링 시스템 만들기

## 적용전

- 뽑기 기계의 위치, 뽑기 알 갯수 등의 정보를 저장하는 GumballMachine 클래스 

```java
public class GumballMachine {
	String location;
	int count;
	
	// ...
	
	public GumballMachine(String location, int count) {
		this.location = location;
		this.count = count;
	}
	
	// ...
}

```  

- 뽑기 기계의 위치, 알맹이 수, 현재 상태를 출력하느 GumballMonitor 클래스

```java
public class GumballMonitor {
	GumballMachine machine;
	
	public GumballMonitor(GumballMachine machine) {
		// 생성자르 통해 뽑기 기계를 전달받고 그 객체를 인스턴스 변수에 저장한다.
		this.machine = machine;
	}
	
	public void report() {
		// 현재 뽑기 기계의 상태를 출력한다.
		System.out.println("Gumball Machine: " + machine.getLocation());
		System.out.println("Current inventory: " + machine.getCount() + " gumballs");
		System.out.println("Current state: " + machine.getState());
	}
}
```  

- 모니터링 테스트 Main 클래스

```java
public class GumballMachineTestDrive {

	public static void main(String[] args) {
		int count = 0;

        if (args.length < 2) {
            System.out.println("GumballMachine <name> <inventory>");
            System.exit(1);
        }

        try {
        	count = Integer.parseInt(args[1]);
		} catch (Exception e) {
			e.printStackTrace();
			System.exit(1);
		}
		GumballMachine gumballMachine = new GumballMachine(args[0], count);

		GumballMonitor monitor = new GumballMonitor(gumballMachine);

		// ...

		monitor.report();
	}
}
```  

## 적용하기
- 뽑기 기계는 매장에 있고, 모니터링은 회사 혹은 집에 있다고 가정해 보자.
- 가정을 통해 달라지는 점
	- GumballMonitor 에 전달하는 GumballMachine 객체는 로컬 JVM에 존재하지 않는다.
	- GumballMachine 의 객체를 받아오기 위해서는 네트워크를 통해 해당 머신의 JVM 에서 받아와야 한다.
- 이런 동작을 위해서는 원격 프록시가 필요하다.

	![원격 프록시 역할]({{site.baseurl}}/img/designpattern/designpattern-proxy-remoteproxy-intro-1.png)  
	
	- 원격 프록시는 원격 객체에 대한 로컬 대변자 역할을 한다.
		- 로컬 대변자(local representative) : 어떤 메서드를 호출하면 다른 원격 객체에게 그 메소드 호출을 전달해주는 역할
	- 다른 자바 가상 머신(JVM)의 힙에 있는 객체를 뜻한다.
	- 클라이언트 객체에서는 원격 객체의 메소드 호출을 하는 것처럼 행동한다.
	- 샐제로는 로컬 힙에 있는 **프록시** 객체의 메서드를 호출한다.
	- 네트워크 통신과 관련된 저수준 작업은 프록시에서 처리해 준다.
	- 이런 원격 객체를 주고 받는 원격 프록시 구현을 위해 **Java RMI** 를 사용한다.
- 원격 메서드의 기초
	- 로컬
		- 통신을 처리해 주는 보조 객체가 필요하다.
		- 보조 객체를 통해 클라이언트는 로컬 객체에 해대서만 메서드로 호출하게 된다.
		- 보조 객체가 실제 서비스를 제공하는 원격 객체에게 전달한다.
	- 원격
		- 통신을 처리해주는 서비스 보조 객체가 필요하다.
		- 서비스 보조 객체는 보조 객체로부터 요청을 받고, 서비스 객체의 메소드를 호출한다.
		- 서비스 객체는 메서드 호출이 원격 요청이 아닌 로컬 요청처럼 인식한다.
		- 서비스 보조 객체는 서비스 객체의 결과를 보조 객체에게 전달한다.
	
	![원격 메서드 기초]({{site.baseurl}}/img/designpattern/designpattern-proxy-remoteproxy-remotemethod.png)

	- 
		
### Java RMI 개요
Java RIM 부분은 스킵해도 좋습니다.  

![RIM 구조]({{site.baseurl}}/img/designpattern/designpattern-proxy-remoteproxy-rmi-archi.png)

- 원격 메서드의 구조에서 **LocalSupportObject -> RMIStub**, **ServiceSupportObject -> RMISkeleton** 으로 대체된 것을 확인 할 수 있다.
- 로컬에서 원격 객체를 찾아 그 원격 객체에 접근하기 위해 룩업(lookup) 서비스와 같은 것도 RMI 에서 제공한다.
- 원격 서비스 만들기
	1. 원격 인터페이스 만들기  	
	
		원격 인터페이스에서는 로컬에서 원격으로 호출 할 수 있는 메소드를 정의한다. 
		로컬에서 이 인터페이스를 서비스 클래스 형식으로 사용한다. 
		Stub 와 실제 서비스에서는 모두 이 인터페이스를 구현해야 한다.
	
		1. java.rmi.Remote 를 확장한다.  
		
			```java
			public interface MyRemote extends Remote {}
			```  
		
		1. 모든 메서드를 RemoteException 을 던지는 메서드로 선언한다.  
		
			로컬에서는 원격 인터페이스를 구현하는 Stub 의 메서드를 호출한다. 
			Stub 에는 네트워킹 및 각종 입출력 작업을 처리하기 때문에 예외가 발생할 수 있다. 
			로컬에서는 원격 예외를 처리하거나 선언해서 대비가 필요하다.
			
			```java
			import java.rmi.*;
			
			public interface MyRemote extends Remote {
				public String say() throws RemoteException;
			}
			```  
		
		1. 인자와 리턴값은 반드시 원시(Primitive) 형식 또는 Serializable 형식으로 선언한다.  
		
			원격 메소드의 인자는 모두 네트워크를 통해 전달되어야 하기 때문에 직렬화가 필요하다. 리턴값도 인자값과 동일하다. 
			사용자 정의 클래스(Custom Class) 를 사용하는 경우라면 클래스를 만들 때 Serializable 인터페이스를 구현해야 한다.
			
			```java
			public class MyObject implements Serializable {}
			```  
	
	1. 서비스 구현 클래스 만들기
		
		원격 인터페이스에서 정의한 원격 메서드를 실제로 구현한 클래스로 실제 작업을 처리한다. 
		로컬에서 이객체에 있는 메서드를 호출하게 된다. 
		
		1. 원격 인터페이스를 구현한다.
		
			서비스 클래스에서는 반드시 원격 인터페이스를 구현해야 한다. 
			로컬에서 그 인터페이스에 있는 메소드를 호출하기 때문이다.
			
			```java
			public class MyRemoteImpl implements MyRemote {
				public String say() {
					return "Hello";
				}
			}
			```  
			
			
		

### Gumball 에 적용하기


# 가상 프록시(Virtual Proxy)
CD 커버 뷰어 만들기

## 설계
![]()

# 보호 프록시(Protection Proxy)

---
 
## Reference
[aaa](aaa)
