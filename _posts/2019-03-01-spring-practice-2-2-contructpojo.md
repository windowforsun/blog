--- 
layout: single
classes: wide
title: "[Spring 실습] 생성자로 POJO 구성하기 (Annotation)"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '생성자를 호출해서 POJO 생성하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Annotation
    - POJO
---  

# 목표
- IoC 컨테이너에서 생성자를 호출해서 POJO 인스턴스/빈을 생성해보자.

# 방법
- POJO 클래스에 하나 이상 생성자를 정의하고 IoC 컨터이너가 사용할 POJO 인스턴스 값을 생성자로 설정한다.
- IoC 컨테이너를 인스턴스화해서 Annotation 을 붙인 자바 클래스를 스캐닝 하도록 한다.
- POJO 인스턴스/빈을 애플리케이션의 일부처럼 액세스 할 수 있다.

# 예제
- 온라인 쇼핑몰 애플리케이션을 개발한다.
- 공통된 제품 정보를 가지고 있는 Product 추상 클래스 이다.

```java
public abstract class Product {

    private String name;
    private double price;

    public Product() {
    }

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }

    public String toString() {
        return name + " " + price;
    }
}
```  

## 생성자 있는 POJO 클래스
- Product 의 하위 클래스인 Battery, Disc 클래스이다.

```java
public class Battery extends Product {

    private boolean rechargeable;

    public Battery() {
        super();
    }

    public Battery(String name, double price) {
        super(name, price);
    }

    public boolean getRechargeable() {
        return rechargeable;
    }

    public void setRechargeable(boolean rechargeable) {
        this.rechargeable = rechargeable;
    }
}

public class Disc extends Product {

    private int capacity;

    public Disc() {
        super();
    }

    public Disc(String name, double price) {
        super(name, price);
    }

    public int getCapacity() {
        return capacity;
    }

    public void setCapacity(int capacity) {
        this.capacity = capacity;
    }
}
```  

- Battery, Disc 모두 고유한 프로퍼티를 가지고 있고 그에 대응하는 생성자도 가지고 있다.

## 자바 설정 클래스 작성하기
- IoC 컨테이너에 POJO 인스턴스 정의를 위해 생성자로 POJO 인스턴스/빈을 생성하는 설정 클래스를 작성한다.

```java
@Configuration
public class ShopConfiguration {

    @Bean
    public Product aaa() {
        Battery p1 = new Battery("AAA", 2.5);
        p1.setRechargeable(true);
        return p1;
    }

    @Bean
    public Product cdrw() {
        Disc p2 = new Disc("CD-RW", 1.5);
        p2.setCapacity(700);
        return p2;
    }
}
```  

- IoC 컨테이너에서 실제로 가져오는지 실행 시키는 Main 클래스

```java
public class Main {

    public static void main(String[] args) throws Exception {

        ApplicationContext context =
                new AnnotationConfigApplicationContext(ShopConfiguration.class);

        Product aaa = context.getBean("aaa", Product.class);
        Product cdrw = context.getBean("cdrw", Product.class);

        System.out.println(aaa);
        System.out.println(cdrw);
    }
}
```  

- 출력결과

```
AAA 2.5
CD-RW 1.5
```  


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
