--- 
layout: single
classes: wide
title: "[Spring 실습] 간단한 Spring MVC 기반 Web Application 만들기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '간단한 Spring MVC 기반 Web Application 으로 Spring MVC 의 개념과 구성에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-webmvc
---  

# 목표
- 간단한 Web Application 을 Spring MVC 로 개발하면서 Framework 의 기본 개념과 구성 방법에 대해 알아본다.

# 방법
![]()
- Front Controller 는 Spring MVC 의 중심 컴포넌트이다.
	- 간단한 Spring MVC Application 은 Java Web deployment descriptor(web.xml, ServletContainerInitializer) 에 Front Controller 의 Servlet 만 구성하면 된다.
	- Dispatcher Servlet(Spring MVC Controller) 는 코어 Java EE 디자인 패턴 중 하나인 Front Controller Pattern 을 구현한 것이다.
	- MVC Framework 에서 모든 웹 요청은 반드시 Dispatcher Servlet 을 거쳐 처리된다.
- Spring MVC Application 에 들어온 웹 요청은 먼저 Controller 가 접수하고 Spring Web Application Context 또는 Controller 자체에 붙인 Annotation 을 이용해 여러 컴포넌트를 구성한다.
- Spring Controller 클래스에는 @Controller 또는 @RestController 를 붙인다.
	- @Controller 를 붙인 클래스(컨트롤러 클래스)에 요청이 들어오면 Spring 은 적합한 Handler Method 를 찾는다.
	- Controller 에는 요청을 처리할 메서드를 @RequestMapping 을 사용해서 핸들러 메서드로 만들고 매핑해서 사용한다.
- Handler Method 의 시그니처는 자바 클래스처럼 정해진 사양은 없다.
	- 메서드명을 임의로 정의해도 되고 인수도 다양하게 정의할 수 있다.
	- Application 로직에 따라 다양한 값(String, void ..)을 반환할 수 있다.
	- @RequestMapping
		- HttpServletRequest 또는 HttpServletResponse
		- 임의형(arbitrary type) 요청 매개변수(@ResponseParam)
		- 임의형 모델 속성(@ModelAttribute)
		- 요청 내에 포함된 쿠키값(@CookieValue)
		- Handler Method 가 모델 속성을 추가하기 위해 사용하는 Map 또는 ModelMap
		- Handler Method 가 객체 바인딩/유효성을 검증한 결과를 가져올 때 필요한 Errors 또는 BindingResult
		- Handler Method 가 세션 처리를 완료했음을 알릴 때 사용하는 SessionStatus
- Controller 는 우선 적절한 Handler Method 를 선택하고 요청 객체를 전달해서 처리 로직을 실행한다.
	- Controller 는 Back-End 서비스에 요청 처리 위임하고, Handler Method 는 다양한 타입(HttpServletRequest, Map, Errors, SessionStatus) 의 Arguments 에 정보를 더하거나 삭제하여 Spring MVC 의 흐름을 이어가는 형태로 구성된다.
- Handler Method 는 요청 처리 후, 제어권을 View 로 넘긴다.
	- 제어권을 넘길 View 는 Handler Method 의 반환값으로 지정한다.
		- user.jsp 혹은 report.pdf 등
		- 직접적인 View 구현체보다(user, report) 파일 확장자가 없는 Logical View 로 나타내면 유연성이 있다.
	- Logical View 이름에 해당하는 String 형 값을 반환하는 경우가 대부분이다.
	- 반환 값을 void 로 선언하면 Handler Method 나 Controller 명에 따라 기본적인 Logical View 가 자동으로 설정된다.
- View 는 Handler Method 의 Arguments 를 얼마든지 가져올 수 있다.
    - Handler Method 가 (String, void) Logical View 이름을 반환할 경우에도 Controller -> View 로 정보를 전달이 가능하다.
    - Map, SessionStatus 형 객체를 Arguments 로 받은 Handler Method 가 수정하더라도, 이 메서드가 반환하는 View 에서 똑같이 수정된 객체를 바라볼 수 있다.
- Controller 클래스는 View 를 받고 View Resolver(View 해석기) 를 이용해 Logical View 이름을 실제 View 구현체로 해석한다.
	- ViewResolver 인터페이스를 구현한 View Resolver 는 웹 애플리케이션 컨텍스트에 빈으로 구성하며 Logical View 이름을 받아(HTML, JSP, PDF ..) 실제 뷰 구현체를 반환한다.
- Controller 클래스가 Logical View 이름을 View 구현체로 해석하면 각 View 의 로직에 따라 Handler Method 가 전달한 (HttpServletRequest, Map, Errors, SessionStatus ..) 객체를 Rendering(실제 화면에 표시할 코드 생성) 한다.
	- View 는 Handler Method 에 추가된 객체를 Client 에게 정확하게 보여주는 역할을 한다.

# 예제
- 스포츠 센터의 코트 예약 시스템을 Spring MVC 로 개발한다.
- 유저는 인터넷으로 웹 애플리케이션에 접속해 온라인으로 예약을 한다.
- domain 하위 패키지에 도메인 클래스를 작성한다.

```java
public class Reservation {
	private String courtName;
	private Date date;
	private int hour;
	private Player player;
	private SportType sportType;
	
	// 생성자, getter, setter
}
```  

```java
public class Player {
	private String name;
	private String phone;
	
	// 생성자, getter, setter
}
```  

```java
public class SportType {
	private int id;
	private String name;
	
	// 생성자, getter, setter
}
```  

- 프리젠테이션 레이어에서 예약 서비스를 제공하는 서비스 인터페이스를 service 하위 패키지에 정의한다.

```java
package ...service;

public interface ReservationService {
	public List<Reservation> query(String courtName);
}
```  

- 간단한 예제를 위해 DB 관련 로직은 제외하고, 하드코딩으로 구현한다.

```java
@Service
public class ReservationServiceImpl implements ReservationService {
	private static final SportType TENNIS = new SportType(1, "Tennis");
    private static final SportType SOCCER = new SportType(2, "Soccer");
    
    private final List<Reservation> reservations = new ArrayList<>();
    
    public ReservationServiceImpl() {

        reservations.add(new Reservation("Tennis #1", LocalDate.of(2008, 1, 14), 16,
                new Player("Roger", "N/A"), TENNIS));
        reservations.add(new Reservation("Tennis #2", LocalDate.of(2008, 1, 14), 20,
                new Player("James", "N/A"), TENNIS));
    }
    
    
    @Override
    public List<Reservation> query(String courtName) {

        return this.reservations.stream()
                .filter(reservation -> Objects.equals(reservation.getCourtName(), courtName))
                .collect(Collectors.toList());
    }
}
```  

## Spring MVC Application 설정하기
- Spring MVC 를 이용한 웹 개발이더라도 Spring MVC 와 필수 라이브러리 제외하고는 일반 Java 웹 개발과 비슷하다.
- Java EE 명세에는 WAR(웹 아카이브) 를 구성하는 자바 웹 애플리케이션의 데릭터리 구조가 명시되어 있다.
- 웹 배포 서술자(web.xml)는 WEB-INF 루트에 두거나, 하나 이상의 ServletContainerInitializer 구현 클래스로 구성해야한다.
- 웹 애플리케이션에 필요한 클래스와 각종 JAR 파일은 각각 WEB-INF/classes 와 WEB-INF/lib 에 넣어 두어야 한다.
- Spring MVC 를 이용해 웹 애플리케이션을 개발하기 위해서는 Spring MVC 의존성을 추가해야 한다.
	- Maven - pom.xml
	
		```xml
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${spring.version}</version>
		</dependency>
		```  
	
	- Gradle - build.gradle
	
		```groovy
		dependencies {
			compile "org.springframework:spring-webmvc:$springVersion"
		}
		```  
		
- CSS 파일과 이미지 파일은 WEB-INF 디렉토리 밖에 두어 유저가 URL 로 직접 접근할 수 있게 한다.
- Spring MVC 에서 JSP 파일은 일정의 템플릿 역할을 한다.
- JSP 는 Framework 가 동적 콘텐츠를 생성하기위해 읽는 파일 이므로 WEB-INF 디렉토리 안에 두고 유저의 접근을 차단해야 한다.
	- 특정 경우에 WEB-INF 내부에 파일을 두면 웹 애플리케이션이 내부적으로 읽을 수 없어 WEB-INF 밖에 두는 경우도 있다.
	
## 설정 파일 작성하기
- 웹 배포 서술자(web.xml, ServletContainerInitializer)는 Java 웹 애플리케이션의 필수 설정파일이다.
	- Application Servlet 을 정의하고 웹 요청 매핑 정보를 기술한다.
	- Spring MVC 의 가장 앞단의 Controller 에 해당하는 DispatcherServlet 인스턴스는 필요시 여러 개 정의할 수도 있다.
- 대규모 Application 에서 DispatcherServlet 인스턴스를 여러개 두면 인스턴스마다 특정 URL 을 전담하도록 설계 할 수 있어 코드 관리가 쉬워진다.
- 개발 팀원 간에 새로 반해하지 않고 각자 애플리케이션 로직에 집중할 수 있다.

```java
package ...web;

public class CourtServletContainerInitializer implements ServletContainerInitializer {

    public static final String MSG = "Starting Court Web Application";

    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();
        applicationContext.register(CourtConfiguration.class);

        DispatcherServlet dispatcherServlet = new DispatcherServlet(applicationContext);

        ServletRegistration.Dynamic courtRegistration = ctx.addServlet("court", dispatcherServlet);
        courtRegistration.setLoadOnStartup(1);
        courtRegistration.addMapping("/");
    }
}

```  

- CourtServletContainerInitializer 클래스에서 정의한 DispatcherServlet 은 Spring MVC 의 핵심 Servlet 클래스로 웹 요청을 받아 적절한 핸들러에 전달한다.
	- Servlet 이름은 court 라고 짓고 슬래시(/)(루트 디렉토리) 가 포함된 모든 URL을 매핑한다.
- URL 패턴을 더 자세히 지정할 수 있다.
	- 대규모 애플리케이션이라면 이런 Servlet 을 여러개 만들어 URL 패턴별로 위임할 수도 있다.
- CourtServletContainerInitializer 를 Spring 이 감지하려면 javax.servlet.ServletContainerInitializer 라는 파일에 추가 작업이 필요하다.
	- com.apress.
	 


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
