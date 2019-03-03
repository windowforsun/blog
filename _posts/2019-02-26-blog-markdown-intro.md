--- 
layout: single
classes: wide
title: "마크다운(Markdown) 작성법"
header:
  overlay_image: /img/blog-bg.jpg
subtitle: 'Markdown 이란 무엇이고, 어떻게 작성하는 지'
author: "window_for_sun"
header-style: text
categories :
  - Blog
tags:
    - Blog
    - Markdown
    - Intro
---  

'Markdown 이란 무엇이고, 어떻게 작성하는 지'

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
===
```  

This is SubTitle H2
===

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

- 어떤 번호를 입력하더라도 순서는 내림차순으로 정의된다.  
	
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
- 참조링크
```markdown
[Google](https://www.google.com)
[Google 링크설명](https://www.google.com "링크 설명")
[상대참조링크](/_posts/post)
[Github][1]
문서안의 [참조링크] 사용하기
구글 바로가기: <https://www.google.com>
내 jekyll 블로그 걸기 [Design Pattern]({{site.baseurl}} {% link _posts/2019-02-08-designpattern-intro.md %})

[1]: https://github.com
[참조링크]: https://www.google.com
```

---
## Reference
[존 그루버 마크다운 페이지 번역](https://nolboo.kim/blog/2013/09/07/john-gruber-markdown/)  
[마크다운 markdown 작성법](https://gist.github.com/ihoneymon/652be052a0727ad59601)  
[MarkDown 사용법 총정리](https://heropy.blog/2017/09/30/markdown/)  