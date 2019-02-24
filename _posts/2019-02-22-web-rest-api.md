--- 
layout: single
classes: wide
title: "REST API"
header:
  overlay_image: /img/web-bg.jpg
subtitle: 'REST API 란 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Web
tags:
    - Web
    - REST
    - RESTful
    - REST API
    - ROA
    - Intro
---  

'REST API 란 무엇이고, 어떠한 특징을 가지고 있는지'

# REST API 란
- API (Application Programming Interface) 란
	- 데이터와 기능의 집합을 제공하여 컴퓨터 프로그램간 상호작용을 촉진하며, 서로 정보를 교환가능 하도록 하는 것
- REST API의 정의
	- REST 기반으로 서비스 API를 구현한 것
	- OPEN API(공공데이터 등), 마이크로 서비스(하나의 큰 애플리케이션을 여러 개의 작은 애플리케이션으로 쪼개어 변경과 조합이 가능 하도록 만든 아키텍쳐) 등을 제공하는 업체가 대부분 REST API를 제공한다.
	
## REST API의 특징
- 사내 시스템들도 REST 기반으로 시스템을 분산해 확장성과 재사용성을 높여 유지보수 및 운용을 편리하게 할 수 있다.
- REST는 HTTP 표준을 기반으로 구현하므로, HTTP를 지원하는 프로그램 언어로 클라이언트, 서버를 구현 할 수 있다.
- REST API를 제작하면 Java, C#, 웹 등을 이용하여 클라이언트를 제작 할 수 있다.

# REST API 디자인 가이드
REST API의 디자인(설계)에 대한 추가 적인 정보는 [RMM]({{site.baseurl}}{% link _posts/2019-02-22-web-rmm-intro.md %})에서 확인할수 있다.  
REST API 설계 시 가장 중요한 항목은 다음 2가지로 요약할 수 있다.
1. URI는 정보의 자원을 표현해야 한다.
1. 자원에 대한 행위는 HTTP Method(GET, POST, PUT, DELETE)로 표현해야 한다.

## REST API 디자인 기본 규칙
### 참고 리소스 원형
- 스토어 : 클라이언트에서 관리하는 리소스 저장소
1. URI는 정보의 자원을 표현해야 한다.
`GET /members/delete/1`  
위와 같은 방식은 REST를 제대로 적용하지 않은 URI다. URI는 자원을 표현해야 하는데 중점을 두어야한다. Delete와 같은 행위에 대한 표현이 들어가서는 안된다.
- resource는 동사보다는 명사를, 대문자 보다는 소문자를 사용한다.
- resource의 도큐먼트 이름으로는 단수 명사를 사용해야 한다.
- resource의 컬렉션 이름으로는 복수 명사를 사용해야 한다.
- resource의 스토어 이름으로는 복수 명사를 사용해야 한다.
- `GET /Memberss/1` -> `GET /members/1`

1. 자원에 대한 행위는 HTTP Method(GET, POST, PUT, DELETE)로 표현해야 한다.
- URI에 HTTP Method가 들어가면 안된다.
	- `GET /members/delet/1` -> `DELETE /members/1`
- URI 행위에 대한 동사 표현이 들어가면 안된다. (CRUD 관련)
	- `GET /members/show/` -> `GET/ members/1`
	- `GET /members/insert/2` -> `POST /members/2`
- 경로 부분 중 변하는 부분은 유일한 값으로 대체한다. (:id는 하나의 특정 resource를 나타내는 고유 값이다)
	- :id를 가진 student를 생성하는 route : `POST /students/:id`
	- :id를 가진 student를 삭제하는 route : `DELETE /students/:id`
	
## REST API 디자인 규칙 및 주의점
1. 슬래시 구분자(/)는 계층 관계를 나타내는데 사용한다.
```
http://restapi.example.com/houses/apartments
http://restapi.example.com/animals/mammels/whales
```  

1. URI 마지막 문자로 슬래시(/)를 포함하지 않는다.
	- URI에 포함되는 모든 글자는 리소스의 유일한 식별자로 사용되어야 하며 URI가 다르다는 것은 리소스가 다르다는 것이다.
	- 반대로 리소스가 다르면 URI도 달라져야 한다.
	- REST API는 분명한 URI를 만들어 통신을 해야 하기 때문에 혼동을 주지 않도록 URI 경로의 마지막에는 슬래시(/)를 사용하지 않는다.

```
http://restapi.example.com/houses/apartments/ (X)
http://restapi.example.com/houses/apartments (O)
```  

1. 하이픈(-)은 URI 가독성을 높이는데 사용
URI를 쉽게 읽고 해석하기 위해, 불가피하게 긴 URI경로를 사용하게 된다면 하이픈을 사용해 가독성을 높일 수 있다.
1. 밑줄(_)은 URI에 사용하지 않는다.
	- 글꼴에 따라 다르긴 하지만 밑줄은 보기 어렵거나 밑줄 때문에 문자가 가려지기도 한다.
	- 이런 문제를 피하기 위해 밑줄 대신 하이픈(-)을 사용하는 것이 좋다.(가독성)
1. URI 경로에는 소문자가 적합하다.
	- URI 경로에 대문자 사용은 피하도록 한다. 
	- 대소문자에 따라 다른 리소스를 인식하게 되어 복잡도가 올라갈수 있다.
	- RFC3986(URI 문법)은 URI 스키마와 호스트를 제외하고는 대소문자를 구별하도록 규정한다.
1. 파일 확장자는 URI에 포함시키지 않는다.
	- REST API에서는 메시지 바디 내용의 포맷을 나타내기 위한 파일 확장자를 URI안에 포함시키지 않는다.
	- 파일 확장자는 Accept Header를 사용한다.
`http://restapi.example.com/members/soccer/345/photo.jpg (X)`
1. 리소스 간의 연관 관계가 있는 경우
	- /리소스명/리소스ID/관계가 있는 다른 리소스명
`http://restapi.example.com/users/:id/devices` (일반적으로 소윳 'has'의 관계를 표현할 때)

## 리소스의 표현
- Collection(컬렉션)과 Document(도튜먼트)
	- Document : 문서, 객체의 인스터스나 데이터베이스 레코드와 유사한 개념
	- Collection : 문서들의 집합, 객체들의 집합
	- Collection과 Document는 모두 리소스라고 표현할 수 있고, URI에 표현된다.
	- `http://restapi.example.com/sports/soccer`
		- sports라는 Collection과 soccer라는 Document로 표현되고 있다.
	- `http://restapi.example.com/sports/soccer/players/13`
		- sports, players라는 Collection과 soccer, 13(13번 선수)를 의미하는 Document로 URI가 이루어 져있다.
		- Collection은 복수로 사용한다.
		- 직관적인 REST API를 위해서는 Collection과 Document를 사용할 때 단수, 복수를 지키면 이해하기 쉬운 URI 설계까 가능하다.

## REST API 디자인 예시
todo 기능을 구현한다고 가정했을 때 RESTful API 표준

| Endpoint| Desc | CRUD | 
|---|---|---|---|
| GET /todos | List all todos | Read
| POST /todos | Create a new todo | Create
| GET /todos/:id | Get a todo | Read
| PUT /todos/:id | Update a todo | Update
| DELETE /todos/:id | Delete a todo and its item | Delete
| PUT /todos/:id/items | Update a todo item | Update
| DELETE /todos/:id/items | Delete a todo item | Delete

## HTTP Status code
- REST API는 URI만 잘 설계된 것이 아닌 그 리소스에 대한 응답을 잘 내어주는 것까지 포함된다.
- 정확한 응답의 상태코드만으로도 많은 정보를 전달 할수가 있기 때문이다.
- 처리 상태에 대한 좀 더 명확한 상태코드 값 사용을 권장한다.

| 상태 코드 | Desc | Detail
|---|---|---|
| 2xx | Successful | 
| 200 | OK | 클라이언트의 요청을 정상적으로 수행함
| 201 | Created | 클라이언트가 리소스 생성을 요청, 해당 리소스가 성공적으로 생성됨(POST를 통한 리소스 생성 작업시)
| 3xx | Redirection | 
| 301 | Move Permanently | 클라이언트가 요청한 리소스에 대한 URI가 변경 되었을 때 사용
| 4xx | Client Error | 
| 400 | Bad Request | 클라이언트의 요청이 부적절 할 경우 사용
| 401 | Unauthorized | 클라이언트가 인증되지 않은 상태에서 보호된 리소스를 요청했을 때 사용
| 403 | Forbidden | 유저 인증 상태와 관계 없이 응답하고 싶지 않은 리소스를 클라이언트가 요청했을 때 사용
| 405 | Method Not Allowed | 클라이언트가 요청한 리소스에서는 사용 불가능한 Method를 이용했을 경우 사용
| 5xx | Server Error | 
| 500 | Internal Server Error | 서버에 문제가 있을 경우 사용

---
## Reference
[Richardson Maturity Model](https://restfulapi.net/richardson-maturity-model/)  
[REST 관련 HTTP 상태 코드](https://zetawiki.com/wiki/REST_%EA%B4%80%EB%A0%A8_HTTP_%EC%83%81%ED%83%9C_%EC%BD%94%EB%93%9C)  
[REST API 이해와 설계 - #2 API 설계 가이드](https://bcho.tistory.com/954?category=252770)  
[[웹기본개념] HTTP 그리고 REST API 다가가기](https://jinbroing.tistory.com/96)  
[RESTful API란 ?](https://brainbackdoor.tistory.com/53)  
[RestFul이란 무엇인가?](https://lalwr.blogspot.com/2016/01/restful.html)  
[[Network] REST란? REST API란? RESTful이란?](https://gmlwjd9405.github.io/2018/09/21/rest-and-restful.html)  