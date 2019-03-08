--- 
layout: single
classes: wide
title: "[Spring 실습] 외부 리소스 데이터 사용하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '외부 리소스(Text, XML, Property, Image) 의 데이터를 사용해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Property
    - Resource
    - POJO
---  

# 목표
- 여러 곳(파일 시스템, ClassPath, URL) 에 있는 외부 리소스(Text, XML, Property, Image ..)를 각자 알맞는 API 로 읽어 보자.

# 방법
- 스프링이 제공하는 @PropertySource를 이용하면 빈 프로퍼티 구성용 .properties 파일(Key-Value)을 읽어 들일 수 있다.
- Resource 라는 단일 인터페이스를 사용해 어떤 유형의 외부 리소스라도 경로만 지정하면 가져올 수 있다.
- @Value 로 접두어를 달리 하여 상이한 위치에 존재하는 리소스를 불어 올 수도 있다.
- 파일시스템 리소스는 file, ClassPath 에 있는 리로스는 classpath 접두어로 붙이는 식이다.
- 리로스 경로는 URL 로도 지정할 수 있다.

# 예제
- @PropertySource 와 PropertySourcePlaceHolderConfigurer 클래스를 이용하면 빈 프로퍼티 구성 전용 프로퍼티 파일의 내용(Key-Value)을 읽을 수 있다.
- 스프링 Resource 인터피에스 @Value 를 함께 사용하면 어느 파일이라도 읽어올 수 있다.

## 프로퍼티 파일 데이터를 이용해 POJO 초깃값 설정하기
- 프로퍼티 파일에 나열된 값을 읽어 비 프로퍼티를 설정 할 수 있다.
- Key-Value 형태로 이루어진 DB 설정 프로퍼티나 각종 애플리케이션 설정값이 대부분 해당된다.
- discounts.properties 파일에 다음과 같이 Key-Value 가 들어 있다고 하자.

```
specialcustomer.discount=0.1
summer.discount=0.15
endofyear.discount=0.2
```  

```java
@Configuration
@PropertySource("classpath:discounts.properties")
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {
	@Value("${endoftear.discount:0}")
	private double specialEndofyearDiscountField;
	
	@Bean
	public static PropertySoucesPlaceholderConfigurer propertySoucesPlaceholderConfigurer() {
		return new PropertySoucesPlaceholderConfigurer();
	}
	
	@Bean
	public Product dvdrw() {
		Disc p2 = new Disc("DVD-RW", 3.0, specialEndofyearDiscountField);
		p2.setCapacity(700);
		return p2;
	}
}
```  

- 값이 classpath:discounts.properties 인 @PropertySouce 를 자바 설정 클래스에 붙여 사용하였다.
- 스프링은 자바 ClassPath(접두어 classpath:) 에서 discounts.properties 파일을 찾는다.
- @PropertySource 를 붙여 프로퍼티 파일을 로드하려면 PropertySourcesPlaceholderConfigurer 빈을 @Bean 으로 선언해야 한다.
- 스프링은 discounts.properties 파일을 자동으로 연결하므로 이 파일에 나열된 프로퍼티를 빈 프로퍼티로 활용 할 수 있다.
- discounts.properties 파일에서 가져온 프로퍼티값을 담을 자바 변수를 정의한다.
- @value 에 placeholder(치환자) 표현식을 넣어 프로퍼티값을 자바 변수에 할당 한다.
- @Value("${key:default_value}") 구문으로 선언하면 읽어들인 애플리케이션 프로퍼티를 전부 뒤져 키를 찾는다.
- 매치되는 키가 있으면 그 값을 빈 프로퍼티값으로 할당하고 키를 찾지 못하면 기본값(default_value)을 할당한다.
- 프로퍼티 파일 데이터를 빈 프로퍼티 설정 외의 다른 용도로 사용하려면 뒤이어 설명할 스프링의 Resource 메커니즘을 이용한다.

## POJO 에서 외부 리소스 데이터를 가져와 사용하기
- 애플리케이션 구동 시 ClassPath에 위치한 banner.txt 라는 텍스트 파일 안에 넣은 문구를 배너로 보여주려고 한다.

```
This is Banner!
Awesome Banner!
```  

- 아래 BannerLoader 는 베너를 읽어 콘솔로 출력하는 POJO 클래스이다.

```java
public class BannerLoader {

    private Resource banner;

    public void setBanner(Resource banner) {
        this.banner = banner;
    }

    @PostConstruct
    public void showBanner() throws IOException {
        Files.lines(Paths.get(banner.getURI()), Charset.forName("UTF-8"))
             .forEachOrdered(System.out::println);
    }
}
```  

- banner 필드는 스프링 Resource 형으로 선언했고 그 값은 빈 인스턴스 생성 시 setter 를 통해 값이 할당 된다.
- showBanner() 메서드는 Files 클래스의 lines(), forEachOrdered() 메서드를 이용해서 배너 파일의 내용을 차례대로 읽어 콘솔에 한 줄씩 출력한다.
- 애플리케이션 구동 시 배너를 보여주기 위해 showBanner() 메서드에 @PostConstruct 를 붙여 스프링에게 빈을 생성한 후 이 메서드를 자동 실행 하도록 한다.
- BannerLoader 를 인스터스화 하고 banner 필드를 주입할 자바 설정 클래스를 작성한다.

```java
@Configuration
@PropertySource("classpath:discounts.properties")
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {
	// ...

    @Value("classpath:banner.txt")
    private Resource banner;

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    @Bean
    public BannerLoader bannerLoader() {
        BannerLoader bl = new BannerLoader();
        bl.setBanner(banner);
        return bl;
    }
    
    // ...
}
```  

- @Value("classpath:banner.txt") 를 통해 스프링은 ClassPath 에서 banner.txt 파일을 찾아 banner 프로퍼티에 주입하고 미리 등록된 프로퍼티 편집기 ResourceEditor 를 이용해 배너 파일을 빈에 주입하기 전 Resource 객체로 변환한다.



