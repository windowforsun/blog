--- 
layout: single
classes: wide
title: "[Spring 실습] Property 파일로 다국어 지원하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Property 파일에서 다국어 메시지를 해석하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Property
    - MessageSource
    - IoC
---  

# 목표
- Annotation 을 이용해서 다국어를 지원하는 애플리케이션을 만들어 보자.

# 방법
- MessageSource 인터페이스에는 리소스 번들(Resource Bundle) 메시지를 처리하는 메서드가 몇가지 정의되어 있다.
- ResourceBundleMessageSource 는 가장 많이 쓰는 MessageSource 구현체로, 로케일별로 분리된 리소스 번들 메시지를 해석한다.
- ResourceBundleMessageSource POJO 를 구현하고 자바 설정 파일에서 @Bean 을 붙여 선언하면 애플리케이션에서 필요한 il8n 데이터를 가져다 쓸 수 있다.

# 예제
- 미국(로케일)의 영어(언어)에 해당하는 message_en_US.properties 리소스 번들을 예로 들어 본다.
- 리소스 번들은 클래스패스 루트에서 읽으므로 이 경로에 파일이 있는지 확인하고 다음과 같이 Key-Value 를 기재한다.

```
alert.checkout=A shopping cart has been checked out.
alert.inventory.checkout=A shopping cart with {0} has been checked out at {1}.
```  

- 리소스 번들 메시지를 구분 처리하려면 ReloadableResourceBundleMessageSource 빈 인스턴스를 자바 설정 파일에 정의한다.

```java
@Configuration
public class ShopConfiguration {

    @Bean
    public ReloadableResourceBundleMessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
        messageSource.setBasenames("classpath:messages");
        messageSource.setCacheSeconds(1);
        return messageSource;
    }
}
```  

- 빈 인스턴스는 반드시 messageSource 라고 명명해야 애플리케이션 컨텍스트가 알아서 감지한다.
- setBasenames() 메서드에 가변 문자열 인수를 넘겨 ResourceBundleMessageSource 번들 위치를 지정한다.
- 예제는 기본 관례에 따라 자바 클래스패스에서 이름이 message 로 시작하는 파일들을 찾도록 설정했다. ("classpath:messages")
- 주기는 1초로 설정(setCacheSeconds(1)) 해서 쓸모없는 메시지를 다시 읽지 않도록 했다.
- 캐시를 갱신할 때는 실제로 프로퍼티 파일을 읽기 전 최종 수정 타임스탬프(timestamp) 이후의 변경 사항이 있는지 살펴보고 없으면 다시 읽지 않는다.
- 위와 같이 MessageSource 를 정의 할 경우
	- 영어가 주 언어인 미국 로케일에서 텍스트 메시지를 찾으면 messages_en_US.properties 리소스 번들 파일이 가장 먼저 탐색 된다.
	- 만일 파일이 클래스패스에 없거나, 해당 메시지를 찾지 못하면 언어에 맞는 messages_en.properties 파일을 탐색한다.
	- 언어에 맞는 파일 마저 없으면 전체 로케일의 기본 파일 messages.properties 를 선택한다.
	- 리소스 번들 로딩에 대한 자세한 정보는 java.util.ResourceBundle 의 JavaDoc 에서 확인 할 수 있다.
- 애플리케이션 컨텍스트를 구성해서 getMessage() 메서드로 메시지를 해석할 수 있다.

```java
public class Main {

    public static void main(String[] args) throws Exception {

        ApplicationContext context = new AnnotationConfigApplicationContext(ShopConfiguration.class);

        String alert = context.getMessage("alert.checkout", null, Locale.US);
        String alert_inventory = context.getMessage("alert.inventory.checkout",
            new Object[]{"[DVD-RW 3.0]", new Date()},
            Locale.US);

        System.out.println("The I18N message for alert.checkout is: " + alert);
        System.out.println("The I18N message for alert.inventory.checkout is: " + alert_inventory);
    }
}
```  


- getMessage() 메서드의 첫 번째 인수는 메시지 키, 두 번째 인수는 메시지 매개변수 배열, 세 번째 인수는 대상 로케일 이다.
	- alert 변수의 할당문에는 null 을, alert_inventory 변수 할당문에는 각각의 메시지 매개변수 자리에 끼워 넣을 값을 객체 배열 형태로 넘겼다.
- Main 클래스는 애플리케이션 컨텍스트를 직접 가져올 수 있으므로 텍스트 메시지를 해석할 수 있지만, 텍스트 메시지를 해석하려고 하는 다른 빈에는 MessageSource 선언하고 구현체를 인젝션 해야 한다.

```java
@Component
public class Cashier {
	@Autowired
	private MessageSource messageSource;
	
	public void setMessageSource(MessageSource messageSource) {
		this.messageSource = messageSource;
	}
	
	public void checkout(ShoppingChart cart) throws IOException {
		String alert = this.messageSource.getMessate("alert.inventory.checkout", new Object[] {cart.getItems(), new Date()}, Locale.US);
		System.out.println(alert);
	}
}
```
	
---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
