--- 
layout: single
classes: wide
title: "[Algorithm 개념] 지수승, 거듭제곱 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '지수승, 거듭제곱 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Pow
---  

## 지수승 알고리즘
### Recursive
- a^n 을 정리하면 아래와 같다.
	- n = 0이면 1을 반환한다.
	- n 이 짝수 이면 (a^(n/2))^2 를 반환한다.
	- n 이 홀수 이면 ((a^(n/2))^2) * a 를 반환한다.
	
### Iterative
- 짝수일 경우

```
a^8 = (a^4)^2 
    = ((a^2)^2)^2 
    = (((a^1)^2)^2)^2 
    = a^(2*2*2)
```  

- 홀수일 경우

```
a^7 = ((a^3)^2) * a
    = ((((a^1)^2) * a)^2) *a
    = a^(2*2+2+1)
```  

### 소스코드

```java
```

---
## Reference
[Floyd-Warshall(플로이드 와샬) 알고리즘](https://dongdd.tistory.com/107)  \ 

