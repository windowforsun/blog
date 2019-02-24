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

# RMM 이란
![RMM 그림]({{site.baseurl}}/img/web-rest-api-rmm.jpg)
- Leonard Richardson이라는 사람이 100가지 웹 서비스 디자인을 분석하여 REST를 얼마나 준수하는지에 따라서 네가지 범주로 나눈 것
- 이 모델은 REST 서비스가 얼마나 REST한지 구분지은 것으로 Richardson Maturity Model 의 약자로 RMM이라고 부른다.
- Richardson은  서비스의 REST 함을 판별하기 위해 URI, HTTP Method (CRUD), HATEOAS(Hypermedia)와 같은 세 가지 요소를 사용 했다.
- 서비스가 위 세가지 요소를 사용할 수록 더욱 REST 한 서비스가 된다.

## Level 0 : Single URI and a single verb

| | |
|---|---|
| URI | X 
| HTTP Method | X
| HATEOAS | X

- 웹의 기본 메커니즘을 전혀 사용하지 않는 단계로, HTTP를 통해 데이터를 주고 받지만 모든 데이터 요청이 단일 URI, 단일 HTTP Method(일반적으로 POST) 이고, 요청과 응답의 관련 정보들은 HTTP Body정보를 활용한다.
- POX(Plain Old XML)로 요청과 응답을 주고 받는 RPC 스타일 시스템으로 특정 서비스를 위해 클라이언트에서 접근할 endpoint는 하나인, 가장 원시적인 방법이다.

## Level 1 : Multiple URI based resources and single verb

| | |
|---|---|
| URI | O 
| HTTP Method | X
| HATEOAS | X

- RMM에서 REST를 위한 첫 단계는 리소스(URI)를 도입하는 것이다.
- 모든 요청을 단일 서비스 endpoint로 보내는 것이 아니라, 개별 리소스와 통신한다.
- 서비스에서는 다양한 URI를 사용하지만 하나의 HTTP Method(일반적으로 POST)를 사용한다.

## Level 2 : Multiple URI-based resources and verbs

| | |
|---|---|
| URI | O
| HTTP Method | O
| HATEOAS | X

- URI + HTTP Method 조합으로 리소스를 구분하는 것으로 응답 상태를 HTTP Status code를 활용한다.
- 현재 가장 많은 RESTful API가 이 단계를 제공한다.
- 발생한 에러의 종류를 전달하기 위해 HTTP Status code를 사용하는 것, 안전한 오퍼레이션(GET 등)과 안전하지 않은 오퍼레이션 간의 강한 분리를 제공하는 것이 이 레벨의 핵심 요소이다.
- HTTP Status code의 사용
	- 기존에는 특정 동작(Operation) 이후 다른 페이지로 이동 해야 할때 서버에서 302등의 Redirect 응답을 서버에서 내려 주는 방식
	- HTTP Status code를 사용하고 나서는 특정 동작(Operation) 이후 서버는 성공, 실패(200, 401, 403 ..)등을 알릴 뿐 페이지 이동은 Client 단에서 해결하는 방식
	- 서버는 순수하게 API로서의 기능만을 제공한다.
- Operation의 강한 분리 사용
	- HTTP GET은 멱등(Idempotent)의 방식으로 자원을 추출하는데, 이는 어떤 순서로 얼마든지 호출해도 매번 동일한 결과를 얻도록 한다.
	- 이는 요청하는 클라이언트에게 캐싱 기능을 지원할 수 있게 된다.

## Level 3 : HATEOAS(Hypermedia)

| | |
|---|---|
| URI | O
| HTTP Method | O
| HATEOAS | O

- HATEOAS(Hypertext As The Engine Of Application State)인 Hypermedia란 하나의 컨텐츠가 텍스트나 이미지, 사운드와 같은 다양한 포맷의 컨텐츠를 링크하는 개념이다.
- 관련 컨텐츠를 보기위해 링크를 따라가는 방식과 유사하다.
- 클라이언트가 다른 자원에 대한 링크를 통해 서버와 상호 작용을 한다.
- 특정 API를 요청한 후 다름 단계로 할 수 있는 작업을 알려주는 것으로, 다음 단계 작업을 위한 리소스의 URI를 알려주는 것이다.
- 이 단계를 적용하면 클라이언트에 영향을 미치지 않으면서 서버를 변경하는 것이 가능하다.
- HATEOAS의 단점
	- 클라이언트가 수행할 작업을 찾기 위해 링크를 따라가기 때문에 컨트롤 탐색에 꽤 많은 호출이 발생할 수 있다.
	- 복합성이 증가 할수 있다.
	- HTTP 요청상에 나타나는 부하로 낮은 지연시간이 요구될 때 문제가 될 수 있다.
	- HTTP 기반의 REST Payload는 Json 또는 바이너리 등의 포맷을 지원하므로 SOAP 보다 훨씬 간결할 수 있지만, Trift와 같은 바이너리 프로토콜에는 상대가 되지 못한다.
	- WebSocket의 경우 HTTP 초기 HandShake 후 클라이언트와 서버간에 TCP접속이 이루어지고 브라우저에서 스트림 데이터를 보낼 때 효율적일 수 있다.
	- HTTP가 대규모 트래픽에는 적합할 수 있지만 TCP 또는 다른 네트워크 기술 기반의 프로토콜과 비교하면 낮은 지원시간이 필요한 통신에는 그다지 좋은 선택이 아닐 수 있다.
- 위의 단점에도 HTTP 기반의 REST는 서비스 대 서비스의 상호작용을 위해 합리적이고 기본적인 선택이다.


# REST API 디자인 가이드
REST API 설계 시 가장 중요한 항목은 다음 2가지로 요약할 수 있다.
1. URI는 정보의 자원을 표현해야 한다.
1. 자원에 대한 행위는 HTTP Method(GET, POST, PUT, DELETE)로 표현해야 한다.
### 참고 리소스 원형
- 도큐먼트 : 객체 인스턴스나 데이터베이스 레코드와 유사한 개념
- 컬렉션 : 서버에서 관리하는 디렉터리 리소스
- 스토어 : 클라이언트에서 관리하는 리소스 저장소

## REST API 디자인 기본 규칙
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
	
## REST API 디자인 규칙
1. 슬래시 구분자(/)는 계층 관계를 나타내는데 사용한다.


---
## Reference
[Richardson Maturity Model](https://restfulapi.net/richardson-maturity-model/)  
[REST API 이해와 설계 - #2 API 설계 가이드](https://bcho.tistory.com/954?category=252770)  
[[웹기본개념] HTTP 그리고 REST API 다가가기](https://jinbroing.tistory.com/96)  
[RESTful API란 ?](https://brainbackdoor.tistory.com/53)  
[RestFul이란 무엇인가?](https://lalwr.blogspot.com/2016/01/restful.html)  
[[Network] REST란? REST API란? RESTful이란?](https://gmlwjd9405.github.io/2018/09/21/rest-and-restful.html)  