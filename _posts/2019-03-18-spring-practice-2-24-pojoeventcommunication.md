--- 
layout: single
classes: wide
title: "[Spring 실습] POJO 사이에서 Application Event 주고 받기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'POJO 간의 통신을 해서 Application Event 를 주고 받아 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-core
    - ApplicationEvent
    - A-EventListener    
---  

# 목표
- POJO 들이 서로 통신할 때 Sender, Receiver 를 찾아 그 메서드를 호출한다.
- Sender 가 Receiver 를 인지ㅣ하는 구조의 경우 단순하고 직접적인 통신이 가능하지만, 통신하는 POJO 간의 결합도가 강해질 수 있다.
- IoC 컨테이너에서는 POJO 가 구혀녀체가 아닌, 인터페이스를 사용해 통신하므로 이러한 결합도를 낮출 수 있다.
- 1:1 통신의 경우 효율적이지만, 1:N 통신의 경우 각 Receiver 를 호출해야 한다.

# 방법
- Spring Application Context 는 Bean 간의 Event 기반 통신을 지원한다.
	- Event 기반 통신 모델에서는 실제로 Receiver 가 여러개 존재할 수 있기 때문에 Sender 는 누가 수신할지 모랜츠 Event 를 발행한다.
	- Receiver 역시 누가 Event 를 발행 했는지 알 필요 없고 여러 Sender 가 발행한 Event 를 리스닝 할 수도 있닫.
	- 위와 같은 방법으로 Sender 와 Receiver 를 느슨하게 엮어 결합도를 낮추게 된다.
- Event 를 리스닝 하기위해서 ApplicationListener 를 사용하는 방법이 있다.
	- ApplicationListener 인터페이스를 구현하고 알림받고 싶은 Event 형을 (ApplicationListener<이벤트 타입>) 타입 매개변수로 지정하여 빈을 구현한다.
	- 이러한 리스너는 ApplicationListener 인터페이스에 타입 시그니처로 명시된 ApplicationEvent 를 상속한 Event 만 리스닝 할 수 있다.
	- Event 를 발행하기 위해 빈에서 ApplicationEventPublisher 를 가져오고, Event 전송을 위해 Event 에서 publishEvent() 메서드를 호출해야 한다.
	- ApplicationPublisher 에 액새스 하기 위해서는 해당 클래스가 ApplicationEventPublisherAware 인터페이스를 구현 또는 ApplicationEventPublisher 형 필드에 @Autowired 를 붙여야 한다.
	
# 예제
- 커스텀 ApplicationEvent 를 만들고, Event 를 받고 특정 동작을 하는 컴포넌트를 구현한다.

## ApplicationEvent 로 Event 정의하기
- Event 기반 통신을 위해서는 Event 자체를 정의해야 한다.
- ShoppoingCart 를 체크아웃하면 Casher 빈이 체크아웃 시간이 기록된 CheckoutEvent 를 발행한다고 하자.

```java
public class CheckoutEvent extends ApplicationEvent {

    private final ShoppingCart cart;
    private final Date time;
    
    public CheckoutEvent(ShoppingCart cart, Date time) {
        super(cart);
        this.cart=cart;
        this.time = time;
    }

    public ShoppingCart getCart() {
        return cart;
    }

    public Date getTime() {
        return this.time;
    }
}
```  

## Event 발행하기
- Event 를 인스턴스화 한 다음 애플리케이션 Event 발행기에서 publishEvent() 메서드를 호출하면 Event 가 발행된다.
- Event 발행기는 아래와 같이 ApplicationEventPublisherAware 인터페이스 구현 클래스를 사용하면 된다.

```java
public class Cashier implements ApplicationEventPublisherAware {

	// @Autowired 를 사용할 수도 있다.
    private ApplicationEventPublisher applicationEventPublisher;

    @Override
    public void setApplicationEventPublisher(
            ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    public void checkout(ShoppingCart cart) throws IOException {
        CheckoutEvent event = new CheckoutEvent(cart, new Date());
        applicationEventPublisher.publishEvent(event);
    }
}
```  

## Event 리스닝 하기
- ApplicationListener 인터페이스를 구현한 애플리케이션 컨텍스트에 정의된 빈은 타입 매개변수에 매치되는 Event 를 모두 알림 받는다.
	- ApplicationContextEvent 같은 특정 그룹의 이벤트들을 리스닝한다.
	
```java
@Component
public class CheckoutListener implements ApplicationListener<CheckoutEvent> {

    @Override
    public void onApplicationEvent(CheckoutEvent event) {
        // 체크아웃 시 수행할 로직을 여기에 구현합니다.
        System.out.println("Checkout event [" + event.getTime() + "]");
    }
}
```  

- Spring 4.2 부터는 ApplicationListener 인터페이스 없이 @EventListener 를 붙여도 Event 리스터로 만들 수 있다.

```java
@Component
public class CheckoutListener {

    @EventListener
    public void onApplicationEvent(CheckoutEvent event) {
        // 체크아웃 시 수행할 로직을 여기에 구현합니다.
        System.out.println("Checkout event [" + event.getTime() + "]");
    }
}
```  

- @EventListener
	- ApplicationEvent 를 상속할 필요가 없어진다.
	- Event 를 Spring Framework 에 종속된 클래스가 아닌 POJO 로 구현할 수 있다.

```java
public class CheckoutEvent {

    private final ShoppingCart cart;
    private final Date time;
    
    public CheckoutEvent(ShoppingCart cart, Date time) {
        this.cart=cart;
        this.time = time;
    }

    public ShoppingCart getCart() {
        return this.cart;
    }

    public Date getTime() {
        return time;
    }
}
```  

## 애플리케이션 컨텍스트에 리스너 등록
- 리스너의 빈 인스턴스를 선언하거나 컴포넌트 스캐닝으로 감지하면 된다.
- 애플리케이션 컨텍스트는 ApplicationListener 인터페이스를 구현한 빈과 @EventListener 를 붙인 메서드가 위치한 빈을 인지하여 이들이 관심있는 Event 를 각각 통지한다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
