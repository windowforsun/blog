--- 
layout: single
classes: wide
title: "ROA 개요"
header:
  overlay_image: /img/web-bg.jpg
subtitle: 'ROA 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Web
tags:
    - Web
    - REST
    - ROA
    - Intro
---  

'ROA 란 무엇이고, 어떠한 특징을 가지고 있는지'

# ROA 란
Resource Oriented Architecture의 약자로, 웹의 모든 리소스를 URI로 표현하고 구조적이고 유기적으로 연결하여 비 상태 자향적인 방법으로 정해진 Mehtod만을 사용하여 리소스를 사용하는 아키텍쳐. 4가지의 고유한 속성(Addressability, Connectedness, Statelessness, Homogeneous Interface)과 연관되어 진다.([REST]({{site.baseurl}}{% link _posts/2019-02-21-web-rest-intro.md %})는 이 속성들을 지향한다.)
- Addressability (주소로 표현 가능함)
	- 제공하는 모든 정보를 URI로 표시할 수 있어야 한다.
	- 직접 URI로 접근할 수 없고 HyperLink를 따라서만 해당 리소스에 접근할 수 있다면 이는 RESTful하지 않은 웹서비스이다.
- Connectedness (연결됨)
	- 일반 웹 페이지처럼 하나의 리소스들은 서로 주변의 연관 리소스들과 연결되어 표현(Presentation)되어야 한다.
	- 연결되지 않은 독립적인 리소스
	
	```
	{
		"user" : {
			"name" : "CS"
		}
	}
	```  
	
	- 관련 리소스(address, phone)가 잘 연결된 리소스
	
	```
	{
		"user" : {
			"name" : "CS",
			"address" : "CS/seoul",
			"phone" : "CS/010"
		}
	}
	```  
	
- Statelessness (상태 없음)
	- 현재 클라이언트의 상태를 절대로 서버에서 관리하지 않아야 한다.
	- 모든 요청은 일회성의 성격을 가지며 이전의 요청에 영향을 받지 말아야 한다.
	- ex) 메일을 확안하기 위해 꼭 /mail/mailView.jsp 에 접근하여 해당 세션을 유지한 상태에서 메일 리소스에 접근해야 한다. 이는 Statelessness 가 없는 예이다.
	- 세션을 유지 하지 않기 때문에 서버 로드 밸런싱(Load Balancing)이 매우 유리하다.
	- URI에 현재 State를 표현할 수 있어야 한다.(권장 사항)
- Homogeneous Interface (동일한 인터페이스)
	- HTTP 에서 제공하는 기본적인 4가지의 Method와 추가적인 2가지의 Method를 이용해서 리소스의 모든 동작을 정의한다.
	- 리소스 조회 : GET
	- 새로운 리소스 생성 : PUT, POST (새로운 리로스의 URI를 생성하는 주체가 서버이면 POST 사용)
	- 존재하는 리소스 변경 : PUT
	- 존재하는 리소스 삭제 : DELETE
	- 존재하는 메타데이터 보기 : HEAD
	- 대부분의 리소스 조작은 위의 Method를 이용하여 처리 가능하다. 만일 이것들로 처리 불가능한 액션이 필요할 경우에는 POST를 이용하여 추가 액션을 정의할 수 있다. (되도록 지양)


---
## Reference
[RestFul이란 무엇인가?](https://lalwr.blogspot.com/2016/01/restful.html)  