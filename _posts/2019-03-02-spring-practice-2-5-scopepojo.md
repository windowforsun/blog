--- 
layout: single
classes: wide
title: "[Spring 실습] @Scope 를 이용하여 POJO 스코프 지정하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '@Scope 를 이용해서 POJO 에 스코프를 지정하여 POJO 의 생명주기를 관리하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - A-Scope
    - POJO
    - IoC
---  

# 목표
- @Component 같은 Annotation POJO 인스턴스에 붙이는 것은 빈 생성에 관한 템플릿을 정의하는 것뿐, 실제 빈 인스턴스를 정의하는 것이 아니다.
- getBean() 메서드로 빈을 요청하거나 다른 빈에서 참조할 때 스프링은 빈 스코프에 따라 어느 빈 인스턴스를 반환 할지 결정해야 한다.
- 이때 기본 스코프 이외의 다른 빈 스코프를 지정할 경우가 있다.

# 방법
- @Scope 는 빈 스코프를 지정하는 Annotation 이다.
- 스프링은 IoC 컨테이너에 선언한 빈 마다 정확한 인스턴스 하나를 생성하고 이렇게 만들어진 인스턴스는 전체 컨테이너 스코프에 공유 된다.
- getBean() 메서드를 호출하거나 빈을 참조하면 이러한 유일한 인스턴스가 반환된다.
- 이런 유일한 인스턴스가 바로 빈의 기본 스코프인 Singleton 이다.

| Scope | 설명 |
|---|---|
| singleton | IoC 컨테이너 당 빈 인스턴스 하나를 생성한다. 
| prototype | 요청할 대마다 빈 인스턴스를 새로 만든다.
| request | HTTP 요청당 하나의 빈 인스턴스를 생성한다. (웹 애플리케이션 Context)
| session | HTTP 세션당 빈 인스턴스 하나를 생성한다. (웹 애플리케이션 Context)
| globalSession | 전역 HTTP 세션당 빈 인스턴스 하나를 생성한다. (포털 애플리케이션 Context)

# 예제
- 쇼핑몰 애플리케이션의 카트를 예로 들며 빈 스코프에 대해 설명한다.
- 카트를 나타내는 ShoppingCart 클래스를 작성한다.

```java
@Component
public class ShoppingCart {

    private List<Product> items = new ArrayList<>();

    public void addItem(Product item) {
        items.add(item);
    }

    public List<Product> getItems() {
        return items;
    }
}
```  

- 컴퓨터 Device 에 해당하는 빈 인스턴스를 Computer 에 추가할 수 있도록 자바 설정 파일을 작성한다.

```java
@Configuration
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {

    @Bean
    public Product aaa() {
        Battery p1 = new Battery();
        p1.setName("AAA");
        p1.setPrice(2.5);
        p1.setRechargeable(true);
        return p1;
    }

    @Bean
    public Product cdrw() {
        Disc p2 = new Disc("CD-RW", 1.5);
        p2.setCapacity(700);
        return p2;
    }

    @Bean
    public Product dvdrw() {
        Disc p2 = new Disc("DVD-RW", 3.0);
        p2.setCapacity(700);
        return p2;
    }
}

```  

- 2 명의 사람이 동시에 소핑몰에 접속해 했다고 가정한다.
- 1번 고객이 getBean 메서드로 카트를 가져와 상품 2개를 담고 그 다음 2번 고객 역시 같은 방법으로 카트를 가져와 다른 상품을 넣는다.

```java
public class Main {

    public static void main(String[] args) throws Exception {
        ApplicationContext context = new AnnotationConfigApplicationContext(ShopConfiguration.class);

        Product aaa = context.getBean("aaa", Product.class);
        Product cdrw = context.getBean("cdrw", Product.class);
        Product dvdrw = context.getBean("dvdrw", Product.class);

        ShoppingCart cart1 = context.getBean("shoppingCart", ShoppingCart.class);
        cart1.addItem(aaa);
        cart1.addItem(cdrw);
        System.out.println("Shopping cart 1 contains " + cart1.getItems());

        ShoppingCart cart2 = context.getBean("shoppingCart", ShoppingCart.class);
        cart2.addItem(dvdrw);
        System.out.println("Shopping cart 2 contains " + cart2.getItems());

    }
}
```  

- 현재 빈 구성으로는 두 고객이 하나의 카트 인스턴스를 공유한다.
- 출력결과

```
Shopping cart 1 contains [AAA 2.5, CD-RW 1.5]
Shopping cart 2 contains [AAA 2.5, CD-RW 1.5, DVD-RW 3.0]
```  

- 스프링 기본 스코프가 singleton 이라서 IoC 컨테이너당 카트 인스턴스가 한개만 생성 되었기 때문에 위와 같은 결과가 출력된다.
- shoppingCart 빈 스코프를 prototype 으로 설정하면 스프링은 getBean() 메서드를 호출 할 때마다 빈 인스턴스를 새로 만든다.

```java
@Component
@Scope("prototype")
public class ShoppingCart {
	// ...
}
```  

- Main 클래스를 다시 실행하면 결과는 다음과 같다.

```
Shopping cart 1 contains [AAA 2.5, CD-RW 1.5]
Shopping cart 2 contains [DVD-RW 3.0]
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  

