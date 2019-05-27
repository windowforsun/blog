--- 
layout: single
classes: wide
title: "[Algorithm 개념] 이항 계수(Binomial Coefficient), 조합 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '이항 계수, 조합을 구하는 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Binomial Coefficient
  - Combination
---  

## 이항 계수(Binomial Coefficient) 이란
- 이항 계수란 주어진 크기의(순서 없는) 조합의 가짓수 이다.
- 주어진 크기의 집합에서 원하는 개수만큼 순서없이 뽑느 조합의 가짓수를 말한다.
- 2를 상징하는 이항이라는 말이 붙은 이유는 하나의 원소에 대해 뽑거나, 안뽑거나 두 가지 선택만 있기 때문이다.
- N 개의 공에서 K개의 공을 뽑은 경우의 수
	- nCk

## 이항 계수(Binomial Coefficient) 알고리즘
- 직접 계산 : nCk = n! / k!(n-k)!)
- DP : nCk = n-1Ck-1 + nCk-1
- 페르마의 소정리 : 
	
### 예시

### 소스코드

```java
```

---
## Reference
[Floyd-Warshall(플로이드 와샬) 알고리즘](https://dongdd.tistory.com/107)  \ 

