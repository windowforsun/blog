--- 
layout: single
classes: wide
title: "[Spring 실습] POJO 와 IoC 컨테이너 리소스"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'POJO 에게 IoC 컨테이터의 리소스를 알려주자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Aware
    - IoC
    - spring-core
---  

# 목표
- 컴포넌트가 IoC 컨테이너와 직접적인 의존 관계를 가지도록 설계하는 방법은 바람직하지 않지만, 때로는 빈에서 컨테이너 리소스를 인지해야 하는 경우가 존재한다.

# 방법
- 빈이 IoC 컨테이너 리소스를 인지하게 하려면 Aware 인터페이스를 구현해야 한다.
- 스프링은 이 인터페이스를 구현한 빈은 감지해 대상 리소스를 setter 메서드로 주입한다.

Aware 인터페이스 | 대상 리소스 타입
---|---
BeanNameAware | IoC 컨테이너에 구성한 인스턴스의 빈 이름
BeanFactoryAware | 현재 빈 팩토리 컨테이너, 서비스를 호출한다.
ApplicationContextAware | 현재 애플리케이션, 컨텍스트 컨테이너 서비스를 호출한다.
MessageSourceAware | 메시지 소스, 텍스트 메시지를 해석한다.
ApplicationEventPublisherAware | 애플리케이션 이벤트 발생기, 애플리케이션 이벤트를 발생한다.
ResourceLoaderAware |리소스 로더, 외부 리소스를 로드한다.
EnvironmentAware | ApplicationContext 인터페이스에 묶인 org.springframework.core.env.Environment 인스턴스

- ApplicationContext 는 MessageSource, ApplicationEventPublisher, ResourceLoader 를 모두 상속한 인터페이스라서 애플리케이션 컨텍스트만 인지하면 나머지 서비스도 액세스 할 수 있다.
	- 요건을 충족하는 최소범위 내에서 Aware 인터페이스를 선택하는 게 바람직하다.
- Aware 인터페이스의 setter 메서드는 스프링이 빈 프로퍼티를 설정한 이후, 초기화 콜백 메서드를 호출하기 이전에 호출한다.
	1. 생성자나 팩토리 메서드를 호출해 빈 인스턴스를 생성한다.
	1. 빈 프로퍼티에 값, 빈 레퍼런스를 설정한다.
	1. Aware 인터페이스에 정의한 setter 메서드를 호출한다.
	1. 빈 인스턴스를 각 빈 후처리기(BeanPostProcess)에 있는 postProcessBeforeInitialization() 메서드로 넘겨 초기화 콜백 메서드를 호출한다.
	1. 빈 인스턴스를 각 빈 후처리기 postProcessAfterInitialization() 메서드로 넘긴다. 이제 빈을 사용할 준비가 끝났다.
	1. 컨테이너가 종료되면 폐기 콜백 메서드를 호출한다.
- Aware 인터페이스를 구현한 클래스는 스프링과 엮이게 되므로 IoC 컨테이너 외부에서는 동작하지 않을 수 있다.
	- 스프링에 종족된 인터페이스를 구현이 필요한지는 잘 고려해야 한다.
- 스프링 최신 버전에서는 Aware 인터페이스를 구현할 필요가 없다.
	- @Autowired 만 붙여도 얼마든지 ApplicationContext 를 가져 올 수 있기 때문이다.
	- 프레임워크나 라이브러리를 개발할 때에는 Aware 인터페이스를 구현하는게 더 좋을 수 있다.

# 예제
- Cashier 클래스의 POJO 인스턴스가 자신의 빈 이름을 인지하려면 BeanNameAware 인터페이스를 구현하도록 해야한다.
- BeanNameAware 인터페이스를 구현하기만 해도 스프링은 이 빈 이름을 POJO 인스턴스에 자동으로 주입하며 이렇게 가져온 빈 이름을 setter 메서드에서 처리하면 된다.

```java
public class Cashier implements BeanNameAware {

    private String fileName;
    
    // ...

    @Override
    public void setBeanName(String name) {
        this.fileName = name;
    }
}
```  

- 빈 이름이 주입되면 그 값을 이용해 빈 이름이 필수인, 다른 연관된 작업을 할 수 있다.
- 예를 들어 Cashier 클래스에 체크아웃 데이터를 기록할 파일명에 해당하는 fileName 프로퍼티는 앞서 빈 이름으로 설정했기 때문에 더 이상 setFileName() 메서드를 호출할 필요가 없다.

```java
@Bean(initMethod = "openFile", destroyMethod = "closeFile")
public Cashier cashier() {
	final String path = System.getProperty("java.io.tmpdir") + "cashier";
	Cashier c1 = new Cashier();
	c1.setPath(path);
	return c1;
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
