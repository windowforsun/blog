--- 
layout: single
classes: wide
title: "[Spring 실습] 팩토리로 POJO 생성하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '팩토리(정적 메서드, 인스턴스 메서드, Spring FactoryBean) 을 사용해서 POJO 를 생성하자.'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - FactoryBean
    - IoC
    - spring-core
---  

# 목표
- 정적/인스턴스 팩토리 메서드를 사용해서 IoC 컨테이너에 POJO 인스턴스를 생성한다.
- 객체 생성 로직을 정적 메서드나 다른 인스턴스 메서드 내부로 캡슐화한다.
- 객체가 필요한 클라이언트는 해당 메서드를 호출해서 사용하면 된다.
- 스프링 팩토리 빈을 사용해서 IoC 컨테이너에 POJO 인스턴스를 만든다.
- IoC 컨테이너 안에서 팩토리 빈은 다른 비을 찍어내는 공장(Factory) 역할을 하며 개념은 팩토리 메서드와 비슷하지만 빈 생성 도중 IoC 컨테이너가 식별 할 수 있는 스프링 전용 빈이다.

# 방법
- 자바 설정 클래스의 @Bean 메서드는 정적 팩토리를 호출하거나 인스턴스 팩토리 메서드를 호출해서 POJO 를 생성할 수 있다.
- 스프링은 FactoryBean 인터페이스를 상속한 간편한 템플릿 클래스 AbstractFactoryBean 을 제공한다.

# 예제
- 스프링에서 팩토리 메서드를 정의, 활용하는 다양한 방법을 살펴본다.
- 팩토리 메서드 사용법, 인스턴스 팩토리 메서드와 스프링 팩토리 빈 까지 살펴본다.

## 정적 팩토리 메서드로 POJO 생성하기
- ProductCreator 클래스에서 정적 팩토리 메서드 createProduct 는 productId 에 해당하는 상품 객체를 생성한다.
- 주어진 productId 에 따라 인스터스화할 실제 상품 클래스를 내부 로직으로 결정한다.
- 해당되는 productId 의 경우 IllegalArgumentException 예외를 던진다.

```java
public class ProductCreator {

    public static Product createProduct(String productId) {
        if ("aaa".equals(productId)) {
            return new Battery("AAA", 2.5);
        } else if ("cdrw".equals(productId)) {
            return new Disc("CD-RW", 1.5);
        } else if ("dvdrw".equals(productId)) {
            return new Disc("DVD-RW", 3.0);
        }
        throw new IllegalArgumentException("Unknown product");
    }
}
```  

- 자바 설정 클래스 @Bean 메서드에서는 일반 자바 구문으로 정적 팩토리 메서드를 호출해 POJO 를 생성한다.

```java
@Configuration
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {

    @Bean
    public Product aaa() {
        return ProductCreator.createProduct("aaa");
    }

    @Bean
    public Product cdrw() {
        return ProductCreator.createProduct("cdrw");
    }

    @Bean
    public Product dvdrw() {
        return ProductCreator.createProduct("dvdrw");
    }
}
```  

## 인스턴스 팩토리 메서드로 POJO 생성하기
- 맵을 구성해서 상품 정보를 담아둘 수도 있다.
- 인스턴스 팩토리 메서드 createProduct() 는 productId 에 해당하는 상품을 맵에서 찾는다.
- 해당되는 productId 가 없을 경우 IllegalArgumentsException 예외를 던진다.

```java
public class ProductCreator {

    private Map<String, Product> products;

    public void setProducts(Map<String, Product> products) {
        this.products = products;
    }

    public Product createProduct(String productId) {
        Product product = products.get(productId);
        if (product != null) {
            return product;
        }
        throw new IllegalArgumentException("Unknown product");
    }
}
```  

- ProductCreator 에서 상품을 생성하기 위해서는 먼저 @Bea 을 선언하여 팩토리값을 인스턴스화 하고 이 팩토리 퍼사드 역할을 하는 두 번째 빈을 선언한다.
- 팩토리를 호출하고 createProduct() 메서드를 호출해서 다른 빈들을 인스턴스화 한다.

```java
@Configuration
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {

    @Bean
    public ProductCreator productCreatorFactory() {
        ProductCreator factory = new ProductCreator();
        Map<String, Product> products = new HashMap<>();
        products.put("aaa", new Battery("AAA", 2.5));
        products.put("cdrw", new Disc("CD-RW", 1.5));
        products.put("dvdrw", new Disc("DVD-RW", 3.0));
        factory.setProducts(products);
        return factory;
    }

    @Bean
    public Product aaa() {
        return productCreatorFactory().createProduct("aaa");
    }

    @Bean
    public Product cdrw() {
        return productCreatorFactory().createProduct("cdrw");
    }

    @Bean
    public Product dvdrw() {
        return productCreatorFactory().createProduct("dvdrw");
    }
}
```  

## 스프링 팩토리 빈으로 POJO 생성하기
- 할인가가 적용된 상품을 생성하는 팩토리 빈을 작성한다.
- 이 빈은 product, discount 두 프로퍼티값을 받아 주어진 상품에 할인가를 계산하여 적용하고 상품 빈을 새로 만들어 반환한다.

```java
public class DiscountFactoryBean extends AbstractFactoryBean<Product> {

    private Product product;
    private double discount;

    public void setProduct(Product product) {
        this.product = product;
    }

    public void setDiscount(double discount) {
        this.discount = discount;
    }

    @Override
    public Class<?> getObjectType() {
        return product.getClass();
    }

    @Override
    protected Product createInstance() throws Exception {
        product.setPrice(product.getPrice() * (1 - discount));
        return product;
    }
}
```  

- 팩토리 빈은 제네릭 클래스 AbstractFactoryBean<T> 를 상속하고, createInstance() 메서드를 Override(재정의)해 대상 빈 인스턴스를 생성한다.
- 자동 연결 기능이 작동하도록 getObjectType() 메서드로 대상 빈 타입을 반환한다.
- 상품 인스터스를 생성하는 팩토리 빈에 @Bean 을 붙여 DiscountFactoryBean 을 적용 한다.

```java
@Configuration
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {

    @Bean
    public Battery aaa() {
        Battery aaa = new Battery("AAA", 2.5);
        return aaa;
    }

    @Bean
    public Disc cdrw() {
        Disc aaa = new Disc("CD-RW", 1.5);
        return aaa;
    }

    @Bean
    public Disc dvdrw() {
        Disc aaa = new Disc("DVD-RW", 3.0);
        return aaa;
    }

    @Bean
    public DiscountFactoryBean discountFactoryBeanAAA() {
        DiscountFactoryBean factory = new DiscountFactoryBean();
        factory.setProduct(aaa());
        factory.setDiscount(0.2);
        return factory;
    }

    @Bean
    public DiscountFactoryBean discountFactoryBeanCDRW() {
        DiscountFactoryBean factory = new DiscountFactoryBean();
        factory.setProduct(cdrw());
        factory.setDiscount(0.1);
        return factory;
    }

    @Bean
    public DiscountFactoryBean discountFactoryBeanDVDRW() {
        DiscountFactoryBean factory = new DiscountFactoryBean();
        factory.setProduct(dvdrw());
        factory.setDiscount(0.1);
        return factory;
    }
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
