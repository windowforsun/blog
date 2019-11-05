--- 
layout: single
classes: wide
title: "[Web 개념] JWT(JSON Web Token)"
header:
  overlay_image: /img/web-bg.jpg
excerpt: 'JWT 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Web
tags:
    - Web
    - Token
    - Authentication
    - JWT
---  

## JWT (Json Web Token) 이란
- JSN Web Token 은 웹표준([RFC 7519](https://tools.ietf.org/html/rfc7519)) 이다.
- 두 엔드포인트에서 JSON 객체를 통해 가볍고 가자수용적인(self-contained) 방식으로 정보를 안전성 있게 전달한다.
- 인증 헤더 내에 사용되는 토큰 포멧으로 토큰은 Base64 로 인코딩한 문자열로 구성된다.
- C, C++, C#, Java, PHP, Python, R, JavaScript, Ruby, Go 등 다양한 언어에서 지원된다.
- JWT 는 필요한 기본정보, 전달할 정보, 인증 정보 등을 증명해주는 Signature 과 같은 정보를 자체적으로 지니고 있다.
- 단순한 토큰 이기때문에 헤더에 넣어 전달하거나, URL 파라미터에 넣어 전달하는 등 손쉽게 전달 가능하다.
- [JWT](https://jwt.io/#debugger) 에서 토큰 디코드하거나 인코드를 해볼 수 있다.

## JWT 토큰의 종류
### Access Token
- 자원에 접근 뿐만 아니라, 권한/인증에 대한 토큰을 뜻한다.

### Refresh Token
- Access Token 과 같은 JWT 이고, Access Token 의 탈취 문제 해결을 위해 발급되는 토큰이다.
- 초기 인증 과정에서 Access Token 과 비교적 유효기간을 가진 Refresh Token 이 발급된다.

## JWT 의 사용
### 인증
- 인증 요청을 수행하고, 인증에 대한 응답으로 토큰을 발급해 준다.
- 이후 요청에 발급된 토큰을 요청에 담아 보내면, 서버는 토큰을 통해 인증/인가를 검사하고 요청을 수행한다.
- 인증/인가를 위해 별도의 세션을 관리할 필요가 없기 때문에 자원에 대한 부담을 줄일 수 있다.

### 정보 교환
- JWT 는 두 엔드 포인트 사이에 안정성 있게 정보를 교환하기 좋은 방법이다.
- JWT 의 토큰에 Sign 이 되어 있기 때문에, 받은 정보에 대해 검증이 가능하다.

## JWT 의 구조
- JWT 는 `.` 을 구분자로 3가지 문자열로 구성된다.
- `xxxxx.yyyyy.zzzzz` 와 같이 구성되고 순서대로, `header`, `payload`, `signature` 이다.

## JWT Header(헤더)
- 헤더는 아래와 같은 두가지 정보를 포함하고 있다.
	- type : 토큰의 타입을 지정한다. (JWT)
	- alg : 해싱 알고리즘을 지정하는 필드로, `HMAC`, `SHA 256`, `RSA` 가 사용되고, 해싱 알고리즘은 토큰 검증시 사용되는 `signature` 에서 사용된다.
- 헤더 부분은 `Base64` 로 인코딩 되어 JWT 의 첫번째 부분에 위치한다. 

```json
{
	"alg" : "HS256",
	"typ" : "JWT"
}
```  

## JWT Payload(정보)
- `payload` 에는 토큰에 담는 정보를 포함하고 있다.
- `payload` 에 포함되는 정보의 한 조각을 `Claim(클레임)` 이라 하고, key-value 로 구성된다.
- 클레임은 아래와 같은 세 종류로 나눠 진다.
	- 등록된(registered) 클레임
	- 공개(public) 클레임
	- 비공개(private) 클레임
	
### 등륵된(registered) 클레임
- 등록된 클레임은 서비스에서 사용되는 정보가 아닌, 토큰에 대한 정보를 위해 이미 정해진 클레임이다.
- 등록된 클레임은 선택적이고, 그 리스트는 아래와 같다.
	- iss(Issuer) : 토큰 발급자
	- sub(Subject) : 토큰 제목
	- aud(Audience) : 토큰 대상자
	- exp(Expiration Time) : 토큰 만료시간
	- nbf(Not Before) : 토큰 활성 날짜
	- iat(Issued At) : 토큰 발급 시간
	- jti(JWT ID) : 고유 식별자
	
### 공개(public) 클레임
- 공개 클레임은 충돌이 방지된(collision-resistant) 이름을 가져야 한다.
- 이런 충돌 방지를 위해 `URI` 형식을 사용한다.

```json
{
	"http://windowforsun/auth/admin" : true
}
```  

### 비공개(private) 클레임
- 엔드 포인트들 간 협의를 통해 사용되는 클레임이다.

```json
{
	"name" : "sun",
	"age" : "28"
}
```  

### 완성된 Payload

```json
{
	"iss" : "windowforsun.com",
	"exp" : 1572966412,
	"sub" : "exam",
	"http://windowforsun/auth/admin" : true,
	"userId" : 1,
	"userName" : "sun"
}
```  

- 2개의 등록된 클레임, 공개 클레임 1개, 비공개 클레임 2개로 구성되어 있다.

## JWT Signature(서명)
- 서명은 `헤더 부분의 인코딩 값` + `정보 부분의 인코딩 값` 입력한 비밀키를 통해 해싱 한 값이다.
- 서명부분은 아래와 같이 생성할 수 있다.
	
	```
	HMACSHA256(
      base64UrlEncode(header) + "." +
      base64UrlEncode(payload),
    <your-256-bit-secret>
    )
	```  

## JWT 만들기
- [JWT](https://jwt.io/#debugger) 에서 위에서 작성했던 것들을 바탕으로 JWT 을 생성한다.

![그림 1]({{site.baseurl}}/img/web/concept-jwt-1.png)
 

## JWT 프로세스 
### Access Token 사용
1. 클라이언트 - 로그인 요청 -> 서버
1. 서버 - 인증 - 

 
---
## Reference
[Introduction to JSON Web Tokens](https://jwt.io/introduction/)
[서버 기반 인증, 토큰 기반 인증 (Session, Cookie / JSON Web Token)](https://dooopark.tistory.com/6)  
[[[JWT] JSON Web Token 소개 및 구조](https://velopert.com/2389)  
[토큰 기반 인증 간단 정리 Token based Authentication](https://blog.msalt.net/251)  
[세션 기반 인증 방식과 토큰 기반 인증(JWT)](https://yonghyunlee.gitlab.io/node/jwt/)  