--- 
layout: single
classes: wide
title: "[Linux 실습] Shell 변수 문자열 특정 문자열로 자르기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Shell 변수의 문자열을 특정 문자열을 기준으로 잘라보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
---  

## 환경
- CentOS 7

## 특정 문자열 기준 앞 문자열 가져오기
- `$$<특정 문자열>*` 을 사용해 특정 문자열 기준 앞 문자열을 가져 올 수 있다.

```
$ echo $BASE_STR
My URL is http://my.url.com
$ DESC_STR=${BASE_STR%%http://*}
$ echo $DESC_STR
My URL is
```  

## 특정 문자열 기준 뒷 문자열 가져오기
- `#*<특정 문자열>` 을 통해 특정 문자열 기준 앞 문자열을 가져 올 수 있다.
	- `##*<특정 문자열>` 도 가능하다.

```
$ echo $BASE_STR
My URL is http://my.url.com
$ URL_STR=${BASE_STR#*http://}
$ echo $URL_STR
my.url.com
$ URL_STR_2=http://${BASE_STR#*http://}
$ echo $URL_STR_2
http://my.url.com
```  

	
---
## Reference