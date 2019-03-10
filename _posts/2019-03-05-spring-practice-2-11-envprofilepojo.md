--- 
layout: single
classes: wide
title: "[Spring 실습] 환경 및 프로파일마다 다른 POJO 로드"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '스프링 환경 및 프로파일마다 다른 POJO 를 로드해보자.'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - A-Profile
---  

# 목표
- 동일한 POJO 빈 인스턴스를 여러 애플리케이션 시나리오(개발, 테스트, 운영)별로 초깃값을 달리하여 구성한다.

# 방법
- 자바 설정 클래스를 어러 개 만들고 각 클래스마다 POJO 빈 인스턴스를 묶는다.
- 묶은 의도 파악을 위해 프로파일을 명명하고 자바 설정 클래스에 @Profile 을 붙인다.
- 애플리케이션 컨텍스트 환경을 가져와 프로파일을 설정하여 해당 POJO 들을 가져온다.

# 예제
- POJO 초깃값은 애플리케이션 시나리오마다 달라질 수 있다.
	- 애플리케이션 제작 단계가 개발 -> 테스트 -> 운영이기 때문에
- 자바 설정 클래스를 여러 개 두고 각각 다른 POJO 를 구성한다.
	- ShopConfigurationGlobal, ShopConfigurationStr, ShopConfigurationSumWin
- 애플리케이션 컨텍스트가 시나리오에 맞는 설정 클래스 파일을 읽어 실행하도록 한다.

## @Profile 로 자바 설정 클래스를 프로파일별로 작성하기
- 자바 설정 클래스에 @Profile Annotation 을 붙여 여러가지 버전으로 만든다.

```java
@Configuration
@Profile("global")
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfigurationGlobal {

    @Bean(initMethod = "openFile", destroyMethod = "closeFile")
    public Cashier cashier() {
        final String path = System.getProperty("java.io.tmpdir") + "cashier";
        Cashier c1 = new Cashier();
        c1.setFileName("checkout");
        c1.setPath(path);
        return c1;
    }
}
```  

```java
@Configuration
@Profile({"summer", "winter"})
public class ShopConfigurationSumWin {

    @Bean
    public Product aaa() {
        Battery p1 = new Battery();
        p1.setName("AAA");
        p1.setPrice(2.0);
        p1.setRechargeable(true);
        return p1;
    }

    @Bean
    public Product cdrw() {
        Disc p2 = new Disc("CD-RW", 1.0);
        p2.setCapacity(700);
        return p2;
    }

    @Bean
    public Product dvdrw() {
        Disc p2 = new Disc("DVD-RW", 2.5);
        p2.setCapacity(700);
        return p2;
    }
}
```  

- @Profile 은 클래스 레벨로 붙였기 때문에 자바 설정 클래스에 속한 모든 @Bean 인스턴스는 해당 프로파일에 편입된다.
- 프로파일명은 @Profile 속성값으로 "" 안에 적는다.
	- 이름이 여러 개일 경우 cSV 형식으로 {}로 감싸 적는다.
	- @Profile({"summer", "winter"})

## 프로파일을 환경에 로드하기
- 프로파일에 속한 빈을 애플리케이션에 로그하려면 일단 프로파일을 활성화 한다.
- 여러 프로파일을 한 번에 로드하는 것도 가능하며 자바 런타임 플래그나 WAR 파일 초기화 매개변수를 지정해 프로그램 방식(프로그램 직접 코딩)으로 프로파일을 로드 할 수도 있다.
- 애플리케이션 컨텍스트를 사용해 프로그램 방식으로 프로파일을 로드하려면 먼저 컨텍스트 환경을 가져와 setActiveProfiles() 메서드를 호출한다.

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.getEnvironment().setActiveProfiles("global", "winter");
context.scan("com.apress.springrecipes.shop");
context.refresh();
```  

- 자바 런타임 플래그로 로드할 프로파일을 명시하는 방법도 있다.
	- global, winter 프로파일에 속한 빈을 로드하려면 아래외 같다.
	
```
-Dspring.profile.active=global,winter
```  

## 기본 프로파일 지정하기
- 어떤 프로파일도 애플리케이션에 로드되지 않는 상태를 위해 기본 프로파일을 지정한다.
- 기본 프로파일은 스프링이 활성 프로파일을 하나도 찾지 못할 경우 적용되며 프로그램, 자바 런타임 플래그, 웹 애플리케이션 초기화 매개변수를 사용해 지정한다.
	- 프로그램 방식
		- setActiveProfiles() 대신 setDefaultProfiles() 메서드를 사용한다.
	- 나머지 두 방식
		- spring.profiles.active 를 spring.profile.default 로 대신한다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
