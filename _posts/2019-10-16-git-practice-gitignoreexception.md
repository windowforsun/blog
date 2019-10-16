--- 
layout: single
classes: wide
title: "[Git] .gitignore 예외 등록"
header:
  overlay_image: /img/git-bg.jpg
excerpt: '.gitignore 에 예외를 등록해서, 제외된 디렉토리에서 특정 파일만 git 에 추가하는 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - gitignore
---  

## .gitignore 사용하기

```
# log/test.log 파일만 git 에 추가하기
log/**
!log/test.log

# log/local/ 디렉토리 전체를 git 에 추가하기
log/*
!log/local/
```   

## Git 명령어 사용하기
- `git add -f` 명령어를 통해 강제로 git 에 파일을 추가할 수 있다.

	```
	git add -f <추가할 예외파일>
	```  

---
 
## Reference






