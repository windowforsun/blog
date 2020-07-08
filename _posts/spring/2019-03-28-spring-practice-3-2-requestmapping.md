--- 
layout: single
classes: wide
title: "[Spring 실습] @RequestMapping 을 통해 요청 매핑하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '@RequestMapping Annotation 을 이용해서 요청을 매핑하는 전략을 구성하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-webmvc
    - A-RequestMapping
---  

# 목표
- DispatcherServlet 은 웹 요청을 받는 즉시 @Controller 가 달린 적합한 Controller 클래스에 처리를 위임한다.
- 처리을 위임 받은 Controller 클래스에 선언된 다양한 @RequestMapping 설정 내용들을 바탕으로 적합한 Handler Method 에게 다시 처리를 위임한다.
- 이러한 @RequestMapping 을 사용해서 요청을 매핑하는 전략을 구성해 보자.

# 방법
- Spring MVC Application 에서 웹 요청은 Controller 클래스에 선언된 하나 이상의 @RequestMapping 에 따라 담당 Handler Method 에 매핑된다.
- 핸들러는 컨텍스트 경로(웹 애플리케이션 켄텍스트의 배포 경로)에 대한 상대 경로 및 서블릿 경로(DispatcherServlet 매핑 경로)에 맞게 연결된다.
	- `http://localhost:8080/court/welcome` URL 로 요청이 들어오면, 컨텍스트 경로는 /court, 서블릿 경로는 없기 때문에 /welcome 에 맞는 핸들러를 찾는다.
		- Serlvet 경로는 CourtWebApplicationInitializer 에서 Servlet 경로를 / 로 설정했다.

# 예제
## 메서드에 따라 요청 매핑하기
- @RequestMapping 을 가장 쉽게 사용하는 방법은 Handler Method 에 직접 붙여 URL 패턴을 속성 값으로 정의하는 것이다.
- DispatcherServlet 은 Annotation 에 기재한 요청 URL 과 가장 잘 맞는 핸들러에 처리를 위함한다.

```java
@Controller
public class MemberController {

    private MemberService memberService;

    // 애플리케이션 컨텍스트에서 사용 가능한 서비스를 생성자에 연결합니다.
    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }

    @RequestMapping("/member/add")
    public String addMember(Model model) {
        model.addAttribute("member", new Member());
        model.addAttribute("guests", memberService.list());
        return "memberList";
    }


    // 메서드를 /member/remove, /member/delete 두 URL에 매핑합니다.
    @RequestMapping(value = {"/member/remove", "/member/delete"}, method = RequestMethod.GET)
    public String removeMember(@RequestParam("memberName")
                                       String memberName) {
        memberService.remove(memberName);

        // 리스트를 갱신하기 위해 리다이렉트합니다.
        return "redirect:";
    }
}
```  

- removeMember() 메서드 처럼 URL 을 여러개 할당
	- /member/remove, /member/delete 둘 다 이 메서드에 매핑 된다.
- 별다른 설정이 없을 경우 수신된 요청은 HTTP GET 방식으로 인지한다.

## 클래스에 따라 요청 매핑하기
- @RequestMapping 은 Controller 클래스에도 붙여 사용할 수 있다.
- 클래스 레벨에 @ReuqestMapping 을 붙이면 클래스에 속한 전체 메서드에 @RequestMapping 을 하나씩 붙이지 않아도 된다.
- 메서드 마다 @RequestMapping 으로 URL 을 좀 더 세밀하게 컨트롤 할 수 있다.
- 와일드카드(*) 를 사용해서 URL 를 폭넓게 매치할 수 있다.

```java
@Controller
@RequestMapping("/member/*")
public class MemberController {
	private final MemberService memberService;
	
	public MemberController(MemberService memberService) {
		this.memberService = memberService;
	}
	
	@RequestMapping("add")
	public String addMember(Model model) {
		// ...
		return "memberList";
	}
	
	@RequestMapping(value={"remove", "delete"}, method=RequestMethod.GET)
	public String removeMember(@RequestParam("memberName") String memberName) {
		// ...
		return "redirect";
	}
	
	@RequestMapping("display/{member}")
	public String displayMember(@PathVarliable("member") String member, Model model) {
		// ...
		return member;
	}
	
	@RequestMapping
	public void memberList() {
	
	}
	
	public void memberLogic(String memberName) {
	
	}
}
```  

- 클래스 레벨의 @RequestMapping 에 와일드카드(*) 가 포함된 `URL(/member/*)`이 있으므로 /member/ 로 시작하는 URL 은 모두 이 컨트롤러의 Handler Method 중 하나와 매핑된다.
- HTTP GET /member/add 요청하면 addMember() 메서드 호출된다.
- HTTP GET /member/remove, /member/delete 요청하면 removeMember() 메서드가 호출 된다.
- displayMember() 메서드의 @RequestMapping 속성 값에 있는 `{경로변수}` 는 URL 안에 포함된 값을 Handler Method 의 입력 매개변수값으로 전달 하겠다는 의미이다.
	- Handler Method 매개변수 부분에도 @PathVariable("member") String member 라고 선언되어 있다.
	- 요청 URL 이 member/display/jdeo 이면 Handler Method 의 member 변수값이 jdoe 로 설정된다.
	- 햄들러에서 요청 객체를 참조할 필요가 없어서 편리한 기능이고, REST 형 웹 서비스를 설계할 때 유용한 기법이다.
- memberList() 의 @RequestMapping 이 선언되었지만 아무런 속성 값이 없다.
	- 클래스 레벨에 이미 와일드카드를 쓴 URL(/member/*) 이 있으므로 다른 메서드에 걸리지 않는 모든 요청들과 매핑될 수 있다.
	- /member/abd, /member/randomroute 등 다양한 URL 이 포함될 수 있다.
	- 반환값은 void 라서 Handler Method 는 자신의 이름과 같은 뷰 (memgerList) 로 제어권을 넘기게 된다.
- memberLogic() 에는 @RequestMapping 이 달려 있지 않다.
	- Spring MVC 와 무관한 해당 클래스 내부에서만 사용되는 유틸리티 클래스 이다.
	
## HTTP 요청 메서드에 따라 요청 매핑하기
- @RequestMapping 은 모든 HTTP 요청 메서드를 처리할 수 있다. 하지만 한 Handler Method 에서 GET, POST 요청을 모두 처리하지는 않는다.
- HTTP 메서드 별로 요청을 따로 처리하기 위해서는 @RequestMapping 에 HTTP 요청 메서드를 명시한다.

```java
@RequestMapping(value="processUser", method=RequestMethod.POST)
public String submitForm(@ModelAttribute("member") Member member, BindingResult result, Model model) {
	// ..
}
```  

- 특정한 Web Application 에서는 (REST 웹 서비스) GET/POST 를 포함한 다양한 HTTP METHOD 를 처리할 수 있다.
	- HTTP 요청 메서드의 종류는 HEAD, ,GET, POST, PUT, DELETE, PATCH, TRACE, OPTIONS, CONNECT 모두 8가지이다.
- Spring MVC 에서 지원하는 HTTP METHOD Annotation 은 아래와 같다.

	요청 메서드 | Annotation
	---|---
	POST | @PostMapping
	GET | @GetMapping
	DELETE | @DeleteMappping
	PUT | @PutMapping

- 위 Annotation 들은 @RequestMapping 을 특정 짓는 Annotation 이다.

```java
@PostMapping("processUser")
public String submitForm(@ModelAttribute("member") Member member, BindingResult result, Model model) {
	// ...
}
```  

- @RequestMapping 에서 지정한 URL 을 보면 .html, .jsp 와 같은 파일 확장자가 존재하지 않는다.
	- 이는 MVC 설계 사상이 반영된 부분이다.
	- Controller 는 HTML, JSP 와 같은 특정 View 구현 기술에 얽매이지 않아야 한다.
	- Controller 에서 Logical View 를 반환할 때 URL 에 확장자를 넣지 않는 것도 이러한 이유 때문이다.
	
---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
