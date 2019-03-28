--- 
layout: single
classes: wide
title: "[Spring 실습] HandlerIntercept 를 사용해서 요청 가로채기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'HandlerIntercept 로 요청을 가로채고 전처리, 후처리를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-webmvc
    - HandlerIntercept
---  

# 목표
- Servlet 명세에 정의된 Servlet Filter 를 사용하면 웹 요청을 Servlet 이하기 전후에 Pre-Handle(전처리), Post-Handle(후처리) 를 할 수 있다.
- Spring Web Application Context 의 Filter 와 유사한 함수를 구성해서 Container 기능를 활요해 이를 구현해보자.
- Spring MVC Handler 로 웹 요청을 넘기기 전후에 전처리, 후처리를 하고, 핸들러가 반환한 모델을 View 로 전달하기 직전에 모델 속성을 조작해보자.

# 방법
- Spring MVC 에서 웹 요청은 HandlerIntercept 로 가로채 전처리/후처리 를 할 수 있다.
- HandlerIntercept 는 Spring 웹 애플리케이션 컨텍스트에 구성하기 때문에 Container 의 기능을 자유롭게 활용 할 수 있고, Container 내부에 선언된 빈도 참조 가능하다.
- HandlerIntercept 는 특정 요청 URL 에만 적용되도록 매핑 가능하다.
- HandlerIntercept
	- 구현을 위해 HandlerIntercept 인터페이스를 필수로 구현해야 한다.
	- preHandle(), postHandle(), afterComplete() 세 콜백 메서드도 구현해야 한다.
	- preHandle(), postHandle() 은 메서드 핸들러가 요청을 처리하기 직전과 직후에 각각 호출된다.
	- postHandle() 은 핸들러가 반환한 ModelAndView 객체에 접근할 수 있다.
	- afterComplete() 메서드는 요청 처리가 모두 끝난(View 렌더링 까지) 이후에 호출된다.
	
# 예제
- Handler Method 에서 웹요청을 처리하는 데 걸린 시간을 측정해 View 에 보여주는 예제를 만들어본다.

```java
public class MeasurementInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        long startTime = (Long) request.getAttribute("startTime");
        request.removeAttribute("startTime");
        long endTime = System.currentTimeMillis();
        modelAndView.addObject("handlingTime", endTime - startTime);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    }
}
```  

- preHandle() 메서드는 요청 처리를 시작한 시각을 재서 HttpServletRequest 에 보관한다.
- DispatcherServlet 은 preHandle() 메서드가 반드시 true 를 반환해야 요청 처리를 계속 진행한다.
	- 그외의 반환값에 대해서는 preHandle() 메서드에서 요청 처리가 끝났다고 보고 클라이언트에게 응답 객체를 반환한다.
- postHandle() 메서드는 요청 속성에 보관된 시작 시작을읽어들여 현재 시각과 비교해서 계산된 소요 시간을 Model 에 추가한 뒤 View 에 넘긴다.
- afterComplete() 메서드는 별다른 처리가 필요없어 빈상태로 남겨뒀다.
- Java 에서는 인터페이스를 구현할 때 원치않는 메서드까지 모두 구현해야 하는 규칙이 있다.
	- Java8 부터는 인터페이스의 메서드에 `default` 키워드를 붙여 구현 코드를 직접 작성할 수 있다. 이 메서드는 하위 클래스에서 오버라이드 하지 않아도 된다.
- HandlerIntercept 인터페이스를 구현하는 대신 InterceptAdapter 클래스를 상복 받아 구현 할 수 있다.
	- InterceptAdapter 는 인터페이스에 선언된 메서드를 모두 구현한 클래스라서 필요한 메서드만 오버라이드해 사용하면 된다.

```java
public class MeasurementInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        long startTime = (Long) request.getAttribute("startTime");
        request.removeAttribute("startTime");
        long endTime = System.currentTimeMillis();
        modelAndView.addObject("handlingTime", endTime - startTime);
    }
}
```  

- MeasurementInterceptor 인터셉터는 다음과 같이 WebMvcConfigurer 인터페이스를 구현한 설정 클래스 InterceptConfiguration 에서 addInterceptors() 메서드를 오버라이드해 추가할 수 있다.
- addInterceptors() 메서드는 인수로 받은 InterceptorRegistry 에 접근하여 인터셉터를 추가한다.

```java
@Configuration
public class InterceptorConfiguration implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(measurementInterceptor());
    }

    @Bean
    public MeasurementInterceptor measurementInterceptor() {
        return new MeasurementInterceptor();
    }
}

```  

- 요청 처리 응답시간을 보여주는 JSP 파일은 아래와 같다.

```
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>

<html>
<head>
    <title>Welcome</title>
</head>

<body>
<h2>Welcome to Court Reservation System</h2>
Today is <fmt:formatDate value="${today}" pattern="yyyy-MM-dd"/>.

<hr/>
Handling time : ${handlingTime} ms

</body>
</html>
```  

- HandlerInterceptors 는 기본적으로 모든 @Contorller 에 적용되지만 원하는 Controller만 선택할 수 있다.
	- Namespace 및 자바 설정 클래스를 이용하면 인터셉터를 특정 URL 에 맵핑할 수 있다.

```java
@Configuration
public class InterceptorConfiguration implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(measurementInterceptor());
        registry.addInterceptor(summaryReportInterceptor()).addPathPatterns("/reservationSummary*");

    }

    @Bean
    public MeasurementInterceptor measurementInterceptor() {
        return new MeasurementInterceptor();
    }

    @Bean
    public ExtensionInterceptor summaryReportInterceptor() {
        return new ExtensionInterceptor();
    }
}
```

```java
public class ExtensionInterceptor extends HandlerInterceptorAdapter {

    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response, Object handler,
                           //Model model) throws Exception {
                           ModelAndView modelAndView) throws Exception {

        String reportName = null;
        String reportDate = request.getQueryString().replace("date=", "").replace("-", "_");
        if (request.getServletPath().endsWith(".pdf")) {
            reportName = "ReservationSummary_" + reportDate + ".pdf";
        }
        if (request.getServletPath().endsWith(".xls")) {
            reportName = "ReservationSummary_" + reportDate + ".xls";
        }

        if (reportName != null) {
            response.setHeader("Content-Disposition", "attachment; filename=" + reportName);
        }
    }
}
```    

- 추가된 summaryReportInterceptor 는 HandlerInterceptorAdaptor 클래스를 상속한 점은 measurementInterceptor 와 비슷하다.
	- /reservationSummary URL 에 매핑된 Controller 인터셉터 로직이 적용되는 점에서는 다르다.
	- 인터셉터를 등록할 때 매핑 URL 을 지정하면 된다.
	- 기본적으로 Ant 스타일의 표현식으로 작성하며 addPathPatterns() 메서드 인수로 전달한다.
	- 제외할 URL 이 있으면 같은 표현식을 excludePathPatterns() 메서드 인수에 지정한다.

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
