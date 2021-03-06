--- 
layout: single
classes: wide
title: "마크다운(Markdown) 작성법"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: 'Markdown 이란 무엇이고, 어떻게 작성하는 지'
author: "window_for_sun"
header-style: text
categories :
  - Blog
tags:
    - Blog
    - Markdown
    - Intro
toc: true
---  

# 마크다운(Markdown) 이란
- Markdown은 텍스트 기반의 마크업 언어로 2004년 존그루버에 의해 만들어졌다.
- 쉽게 쓰고 읽을 수 있으며 HTML로 변환이 가능하다.
- 특수기호와 문자열을 이용한 매우 간단한 구조의 문법을 사용하여 웹에서도 보다 빠르게 컨텐츠를 작성하고 보다 직관적으로 인식할 수 있다.

# 마크다운의 장점
- 간결하다.
- 별도의 도구없이 작성가능하다.
- 다양한 형태로 변환이 가능하다.
- 텍스트로 저장되기 때문에 용량이 적어 보관이 용이하다.
- 텍스트 파일이기 때문에 버전관리 시스템을 이용하여 변경이력을 관리할 수 있다.
- 지원하는 프로그램과 플랫폼이 다양하다.

# 마크다운 단점
- 표준이 없다.
- 표준이 없기 때문에 도구에 따라 변환방식이나 생성물이 다르다.
- 모든 HTML 마크업을 대신하지 못한다.

# 마크다운 사용법
## #해더 (Header)
1. 큰 제목 (문서 제목)

```markdown
This is Title H1
====
```   
	  
This is Title H1
===

1. 작은 제목 (문서 부제목)

```markdown
This is SubTitle H2
---
```  

This is SubTitle H2
---

## #글머리(1 ~ 6까지 지원)

```markdown
# This is H1
## This is H2
### This is H3
#### This is H4
##### This is H5
###### This is H6
```  

# This is H1
## This is H2
### This is H3
#### This is H4
##### This is H5
###### This is H6
		
## #인용문 (BlockQuote)

```markdown
> This is a blockquote 1
>> This is a blockquote 2
>>> This is a Blockquote 3
```  

> This is a blockquote 1
>> This is a blockquote 2
>>> This is a Blockquote 3  

- 이 안에서는 다른 마크다운 요소를 포함할 수 있다.  

```
> # This is Blockquote Title
> - List
> 	1. element
>		```markdown
>			code
>		```
```  

> # This is Blockquote Title
> - List
> 	1. element
>		```markdown
>			code
>		```  

## #목록 (List)
1. 순서 있는 목록(번호)

	```markdown
	1. 첫번째
	1. 두번째
	1. 세번째
	3. 테스트 1
	2. 테스트 2
	```  
	
	1. 첫번째
	1. 두번째
	1. 세번째
	3. 테스트 1
	2. 테스트 2
  
 어떤 번호를 입력하더라도 순서는 내림차순으로 정의된다.  
	
1. 순서 없는 목록(글머리 기호)

	```markdown
	- 첫번째
		- 두번째
			- 세번째
	* 첫번째
		* 두번째
			* 세번째
	+ 첫번째
		+ 두번째
			+ 세번째
	```  
	
	- 첫번째
		- 두번째
			- 세번째
	* 첫번째
		* 두번째
			* 세번째
	+ 첫번째
		+ 두번째
			+ 세번째  
			
- 혼합해서 사용하는 것도 가능하다.  
	
	```markdown
	- 첫번째
		* 두번째
			+ 세번째
	```  
		
	- 첫번째
		* 두번째
			+ 세번째
		
## #코드 (Code) 강조
- 인라인 (inline) 코드 강조

```markdown
`inline` 코드 강조 `강조 할 부분`
```  

`inline` 코드 강조 `강조 할 부분`
	
- 블록 (block) 코드 강조
	`를 세번이상 입력하고 코드 종류도 적는다.
	
```markdown
```markdown
마크다운 코드 작성
(```) 괄호 안에것만
```  

```markdown
```java
public class Main {
	// ...
}
(```) 괄호 안에것만
```   

```markdown
마크다운 코드 작성
```  

```java
public class Main {
	// ...
}
```  

`Github Page` 에서 코드블럭에 언어를 지정할 수 있는데 그 리스트는 [여기](http://www.rubycoloredglasses.com/2013/04/languages-supported-by-github-flavored-markdown/)
에서 확인 할 수 있다. 
		
## #수평선 (Horizontal Rule)
수평선을 만들어 페이지 나누기 용도로 많이 사용한다. 각 기호를 3개이상 입력한다.  

```markdown
---
(Hyphens)

***
(Asterisks)

___
(Underscores)
```  

--- 
(Hyphens)

***
(Asterisks)

___
(Underscores)

## #링크 (Links)

```markdown
[Google](https://www.google.com)
[Google 링크설명](https://www.google.com "링크 설명")
[상대참조링크](/_posts/post)
구글 바로가기: <https://www.google.com>
내 jekyll 블로그 포스트 링크 걸기 [Design Pattern]({{site.baseurl}} {% link _posts/designpattern/2019-02-08-designpattern-intro.md %})

```  


[Google](https://www.google.com)  
[Google 링크설명](https://www.google.com "링크 설명")  
[상대참조링크]({{site.baseurl}}{% link _posts/designpattern/2019-02-08-designpattern-intro.md %})  
구글 바로가기: <https://www.google.com>  
내 jekyll 블로그 포스트 링크 걸기 [Design Pattern]({{site.baseurl}}{% link _posts/designpattern/2019-02-08-designpattern-intro.md %})  

## #강조 (Emphasis)

```markdown
이텔릭체는 *별표(asterisks)* 혹은 _언더바(underscore)_를 사용하세요.
두껍게는 **별표(asterisks)2개** 혹은 __언더바(underscore)2개__를 사용하세요.
**_이텔릭체_와 두껍게**를 같이 사용할 수 있습니다.
취소선은 ~~물결표시(tilde)~~를 사용하세요.
<u>밑줄</u>은 `<u></u>`를 사용하세요.
```  

이텔릭체는 *별표(asterisks)* 혹은 _언더바(underscore)_를 사용하세요.  
두껍게는 **별표(asterisks)2개** 혹은 __언더바(underscore)2개__를 사용하세요.  
**_이텔릭체_와 두껍게**를 같이 사용할 수 있습니다.  
취소선은 ~~물결표시(tilde)~~를 사용하세요.  
<u>밑줄</u>은 `<u></u>`를 사용하세요.  

_언더바(underscore)은 적용되지 않네요 ..  

## #이미지 (Images)<a id="image-anchor"></a>
- 이미지

```markdown
![대체 텍스트(alternative text)를 입력하세요](/img/image.jpg "이미지 설명")
![이미지1](/img/image.jpg)
![jekyll에서 이미지]({{site.baseurl}}/img/image.jpg)
```  


![대체 텍스트(alternative text)를 입력하세요]({{site.baseurl}}/img/home-bg-network_2.jpg "이미지 설명")
![이미지1]({{site.baseurl}}/img/home-bg-network_2.jpg)
![jekyll에서 이미지]({{site.baseurl}}/img/home-bg-network_2.jpg)

- 이미지에 링크

```markdown
[![WifoSun's Blog](/img/image.jpg)](https://windowforsun.github.io/blog/)
```  

[![WifoSun's Blog]({{site.baseurl}}/img/home-bg-network_2.jpg)](https://windowforsun.github.io/blog/)

## #표(Table)<a id="table-anchor"></a>
헤더 셀을 구분할 때 3개 이상의 -(hyphen/dash) 기호가 필요하다.  
헤더 셀 구분하면서 :(Colons)기호로 셀(열/칸) 안에 내용을 정렬할 수 있다.  
가장 좌측과 가장 우측에 있는 |(Vertical bar)기호는 생략 가능하다.

```markdown
| 값 | 의미 | 기본값 | 설명 |
|---|:---:|---:|:---|
| 값11111 | 의미11111111111 | 기본값11111111111 | 설명11111111 |
| 값2 | 의미2 | 기본값2 | 설명2 |
| 값3 | 의미3 | 기본값3 | 설명3 |
| 값4 | 의미4 | 기본값4 | 설명 4|

 값 | 의미 | 기본값 | 설명
---|:---:|---:|:---
 값11111 | 의미11111111111 | 기본값11111111111 | 설명11111111 
 값2 | 의미2 | 기본값2 | 설명2 
 값3 | 의미3 | 기본값3 | 설명3 
 값4 | 의미4 | 기본값4 | 설명 4

```  

| 값 | 의미 | 기본값 | 설명 |
|---|:---:|---:|:---|
| 값11111 | 의미11111111111 | 기본값11111111111 | 설명11111111 |
| 값2 | 의미2 | 기본값2 | 설명2 |
| 값3 | 의미3 | 기본값3 | 설명3 |
| 값4 | 의미4 | 기본값4 | 설명 4|

- 표 안에서 줄넘김은 HTML 코드 `<br/>` 을 통해 할 수 있다.

```markdown
값|의미|기본값|설명
---|---|---|---
값1|의미11|기본값1|설명1
값2|의미21<br/>의미22|기본값2|설명2
값3|의미31<br/>의미32<br/>의미33|기본값3|설명3
```

값|의미|기본값|설명
---|---|---|---
값1|의미11|기본값1|설명1
값2|의미21<br/>의미22|기본값2|설명2
값3|의미31<br/>의미32<br/>의미33|기본값3|설명3

## #원시 HTML (Raw HTML)
마트다운 문법이 아닌 원시 HTML 문법을 사용할 수 있다.  

```markdown
<u>마크다운에서 지원하지 않는 기능</u>을 사용할 때 유용하며 대부분 잘 동작한다.

<img width="150" src="/img/image.jpg" alt="Prunus" title="A Wild Cherry (Prunus avium) in flower">

![Prunus](/img/image.jpg)
```  

<u>마크다운에서 지원하지 않는 기능</u>을 사용할 때 유용하며 대부분 잘 동작한다. 

<img width="150" src="{{site.baseurl}}/img/home-bg-network_2.jpg" alt="Prunus" title="A Wild Cherry (Prunus avium) in flower">  

![Prunus]({{site.baseurl}}/img/home-bg-network_2.jpg)  

## #줄바꿈 (Line Breaks)

```markdown
가나다라마바사  <!-- 띄어쓰기 2번  -->
가나다라마바사<br>
가나다라마바사
```  

가나다라마바사  
가나다라마바사<br>
가나다라마바사

## #네임드 앵커 (Name Anchor)
네임드 앵커는 특정 앵커로 점프하는 용도로 사용된다.

```markdown
- 앵커 리스트
	- [테스트 앵커1](#test-1)
	- [테스트 앵커2](#test-2)
	- [이미지](#image-anchor)
	- [표](#table-anchor)

테스트 앵커1<a id="test-1"></a>  
테스트 앵커2<a id="test-2"></a>  
```  

- 앵커 리스트
	- [테스트 앵커1](#test-1)
	- [테스트 앵커2](#test-2)
	- [이미지](#image-anchor)
	- [표](#table-anchor)
	
테스트 앵커1<a id="test-1"></a>  
테스트 앵커2<a id="test-2"></a>  

## #주석

```markdown
<!--
마크다운에서 주석으로 처리되는 부분
-->
```  

## #각주
본문의 어떤 부분을 설명하거나 보충하기 위해 본문 아래쪽에 별도로 작성하는 간단한 설명문으로서 주로 내용의 출처를 밝힐 때 사용된다.

```markdown
이러쿵 저러쿵 이야기들 어이쿵 요이쿵[^각주]
\[^각주]: 이러쿵 저러쿵 각주
```  

이러쿵 저러쿵 이야기들 어이쿵 요이쿵[^각주]  
\[^각주]: 이러쿵 저러쿵 각주  

각주가 안되네.. ㅠㅠ  

## Expander control(접기/펼치기)

```markdown
<details>
<summary>접기/펼치기</summary>
<div markdown="1">

## 제목
- 여기에 이러쿵 저러쿵

|type|value|
|--|--|
|1|1|
|2|10|

</div>
</details>
```  

<details>
<summary>접기/펼치기</summary>
<div markdown="1">

## 제목
- 여기에 이러쿵 저러쿵

|type|value|
|--|--|
|1|1|
|2|10|

</div>
</details>

## #특수문자
마크다운은 `HTML` 으로 변환되기 때문에 특수문자또한 `HTML` 의 방식으로 사용할 수 있다. 

```
&<name>;
&#<decimal>;
&#<hexadecimal>;
```  

관련 문자 리스트는 [여기](https://en.wikipedia.org/wiki/List_of_XML_and_HTML_character_entity_references)
에서 확인 할 수 있다. 

---
## Reference
[Basic Syntax](https://www.markdownguide.org/basic-syntax)  
[Extended Syntax](http://www.rubycoloredglasses.com/2013/04/languages-supported-by-github-flavored-markdown/)  
[존 그루버 마크다운 페이지 번역](https://nolboo.kim/blog/2013/09/07/john-gruber-markdown/)  
[마크다운 사용법 - Quick Start](http://taewan.kim/post/markdown/#chapter-2)  
[마크다운 markdown 작성법](https://gist.github.com/ihoneymon/652be052a0727ad59601)  
[MarkDown 사용법 총정리](https://heropy.blog/2017/09/30/markdown/)  