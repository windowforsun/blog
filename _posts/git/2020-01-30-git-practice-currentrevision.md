--- 
layout: single
classes: wide
title: "[Git] 현재 리비전(Revision) 알아내기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: '현재 git 저장소의 리비전을 알아내보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Revision
---  

## 현재 브랜치의 현재 리비전

```
$ git rev-parse HEAD
ebd280cf9cfddefba2b3226fb40a0a672a11d945
```  

## 특정 브랜치의 현재 리비전

```
$ git rev-parse master
d65ba7d97cf9e3965f9bc4fda8001c36af1d8b71
```  

---
 
## Reference



