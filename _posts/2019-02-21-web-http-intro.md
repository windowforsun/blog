--- 
layout: single
classes: wide
title: "HTTP 개요"
header:
  overlay_image: /img/web-bg.jpg
subtitle: 'HTTP 무엇이고, 어떠한 특징을 가지고 있는지'
author: "window_for_sun"
header-style: text
categories :
  - Web
tags:
    - Web
    - HTTP
    - Intro
---  

'HTTP란 무엇이고, 어떠한 특징을 가지고 있는지'

---
# HTTP 란
- HTTP(Hyper Text Transport Protocol)은 웹서버와 클라이언트간의 문서를 교환하기 귀한 통신 규약
- WWW(World Wide Web)의 문산되어 있는 서버와 클라이언트 간에 Hypertext를 이용한 정보 교환이 가능하도록 하는 통신 규약
- HTTP는 TCP/IP 기반으로 한지점에서 다른 지점(서버-클라이언트)으로 요청과 응답을 전송한다.

## More
- HTTP는 HTML과 같은, 하이퍼미디어 문서 전송을 위한 응용계층 프로토톨이다.
- HTTP는 웹 브라우저와 웹 서버 사이의 통신을 위해 설계되었지만, 다른 목적을 위해서도 사용 가능하다.
- 연결을 열고, 요청을 보낸 뒤 응답을 받을 때까지 대기하는 클라이언트를 이용하는 고전적인 클라이언트-서버 모델이다.
- 무상태 프로토콜로 두 개의 요청 사이에 어떤 데이터(상태)도 서버가 유지하지 않는다.

---
# HTTP 동작 방식
![http 개요 기본 동작 방식1]({{site.baseurl}}/img/http-intro-baseoperation-1-sequencediagram.png)
- 클라이언트 : 웹 애플리케이션의 경우, 크롬, 파폭, IE 등 브라우저를 통해서 서버에 요청(프로토콜 + 도메인 + URI)
- 서버 : 클라이언트로 부터 받은 요청을 내부적으로 처리하여 그에 대한 결과를 응답

---
# HTTP 특징 및 기능
1. Connectionless
	- 요청-응답 후 연결을 끊음
	- 클라이언트의 이전 상태를 알수 없음, 그래서 쿠키와 세션(클라이언트와 서버의 이전 상태정보)이 필요함
	- 수십만명이 웹서비스를 사용(요청)하더라도 유저의 요청을 처리할 수 있음
1. Stateless
	- 독립적인 트랜잭션으로 취급하는 통신을 의미
	- 이전 요청과 다음 요청간에 관계가 없기 때문에 저장공간을 동적으로 따로 할당할 필요가 없어, 서버에서는 요청간의 관계에 대해 고려 할 필요 없음
	- 클라이언트가 트랜잭션 도중 다운 되더라도, 서버에서는 이를 처리할 필요가 없음
	- 요청마다 추가 정보를 포함해야 하기 때문에, 요청시에 항상 해석해야한다는 단점이 있음
1. Request-Response(Request-Reply)
	- 한 컴퓨터에서 어떤 데이터를 요청하면, 다른 컴퓨터에서는 그 요청에 대한 응답을 보내는 방식
	- 웹 페이지는 Request-Response의 한 예
1. 그외
	- HTTP 메세지는 HTTP Server와 HTTP Client에 의해서 해석
	- TCP/IP 프로토콜의 Application 계층에 위치
	- TCP Protocol을 이용(Default Port 80)
	
---
# HTTP 1.1
- HTTP 1.0의 성능 개선에 중점을 둠

## HTTP 1.0의 문제점
- 단순한 OPEN, OPERATION, CLOSE
- 매번 필요할 때마다 연결(비 지속성 연결방식) -> 성능 저하
- 한번에 얻어서 가져올 수 있는 데이터의 양이 제한
- URL의 크기도 작으며, 캐시 기능이 이흡(Last-Modified에 의존)
- GET/HEAD/POST methdo만 허용

## HTTP 1.1의 개선
- 지속적인 연결을 해 주는 persistent connection 지원
- multiple request 처리 가능
- request/response 가 pipeline 방식으로 진행
- proxy server와 캐시 기능 향상(Cache-Control)
- GET, HEAD, POST, OPTIONS, DELETE, TRACE, CONNET, PUT 메소드 허용

### 파이프라이닝(Pipe Lining) 이란 ?
![http 개요 파이프라이닝1]({{site.baseurl}}/img/http-intro-pipelining-1-sequencediagram.png)
- 응답 메시지가 도착하지 않은 상태에서 연속적인 요구 메시지를 서버에 전달
- 이때 서버는 요구메시지를 수신한 순서대로 응답메시지를 클라이언트에 전달
- 연결과 종료횟수를 줄임으로서 네트워크 자원의 절약
- 발생하는 패킷의 숫자를 감소, 네트워크 트래픽 감소

---
# HTTP Request 구조
- 사용자(웹 애플리케이션)가 서버에 요청을 보낼 때 HTTP Request 구조  

|  | Division | Example |
|---|---|---|
| Header | Request Line | GET /index.html HTTP/1.1
| Header | Request Header | Host:www.example.com:80 User-Agent:Mozilla/5.0 ...
|	| An Empty Line | <carriage return>
| Message | Optional Message Body | POST Data

- Request Line(Header) : 사용자가 서버에 요청하는 메소드, HTTP 버전 확인 가능
- Request Header(Header) : 서버에 전달되는 사용자 정보(웹 애플리케이션 정보: 문자 코드, 언어, 파이 종류)
- Optional Message Body(Message) : HTTP Request 요청 메시지를 담고 있는 부분, GET 메소드의 경우 요청정보가 주소에 담겨져 있어 본문은 빈상태

---
# Request Method

|Request Method|Description|
|---|---|
| GET | 지정된 URL의 정보를 가져온다.
| POST | 지정된 URL로 Body에 포함된 정보를 제출한다.
| PUT | 지정된 URL에 저장될 정보를 전달한다.
| DELETE | 지정된 Resource(정보)를 삭제한다.
| HEAD | GET과 유사하나 헤더 정보외에는 어떤 데이터도 보내지 않는다.
| OPTIONS | 해당 메소드를 통해 시스템에서 지원되는 메소드 종류를 확인한다.
| TRACE | 원격지 서버에 Loopback(루프백)메시지를 호출하기 위해 사용된다.
| CONNET | 웹 서버에 프록시 기능을 요청할 때 사용된다.

---
# HTTP Response 구조
- 서버가 사용자(웹 애플리케이션) 요청에 응답을 할때 HTTP Response 구조

| | Devision | Example |
|---|---|---|
| Header | Response line | HTTP/1.1 200OK
| Header | Response Header | Host:www.example.com:80 ...
| | An Empty Line | <CR><LF>, carriage return
| Message | Message Body | HTML Contents

- Response line(Header) : 사용자의 요청에 대한 서버 처리결과를 나타냄(200, 404, 500...)
- Response Header(Header) : 사용자에게 전달한 데이터 정보를 나타냄
- Message Body(Message) : 사용자에게 전달한 데이터 내용을 담고 있음(요청에 대한 응답 데이터)

---
# Response Status Code

| Range | Status Code | Description |
|---|---|---|
| 1xx(Information) | | 
| 2xx(Success | 200 | OK 
| 3xx(Redirection) | 301 | Moved Permanently
| | 302 | Found
| | 304 | Not Modified
| 4xx(Client Error) | 400 | Bad Request
| | 401 | Unauthorized
| | 403 | Forbidden
| | 404 | Not Found
| 5xx(Server Error) | 500 | Internal Server Error
| | 503 | Service Unavailable
	 

---
# After
- HTTP 2.0
- cookie
- session
- url rewriting
- hidden form field

---
 
## Reference
[HTTP](https://developer.mozilla.org/ko/docs/Web/HTTP)  
[HTTP](https://ko.wikipedia.org/wiki/HTTP)  
[[웹기본개념] HTTP 그리고 REST API 다가가기](https://jinbroing.tistory.com/96)  
[[HTTP]HTTP란? 특징 및 구성요소](https://ourcstory.tistory.com/71)  
[HTTP (HyperText Transport Protocol)의 이해](http://wiki.gurubee.net/pages/viewpage.action?pageId=26739929)  


