--- 
layout: single
classes: wide
title: "[Algorithm 개념] 최소공배수(GCM), 최대공약수(LCM) 구하기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '최대공약수와 최소공배수를 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - GCM
  - LCM
  - Euclidean
---  

## Euclidean Algorithm
- 유클리드 호제법은 자연수에서 최대공약수를 구하는 알고리즘 중 하나이다.
- 호제법은 두 수가 서로 상대방 수를 나누면서 기대하는 수를 얻는 알고리즘을 뜻한다.
- a(큰수), b(작은수) 두 수가 있을 때 유클리드 호제법 적용하기
	- r = a % b(a를 b로 나누었을 때 나머지) 이라고 한다.
	- 다시 b % r 을 통해 계속해서 큰수를 작은수로 나눈 나머지를 r에 넣어주다보면 r 이 어느순간 0이 된다.
	- r == 0 일때가 바로 a, b 의 최대공약수를 구한 시점이 된다.

## 최대공약수 GCM(Greatest Common Divisor)
- 최대공약수는 위에서 언급한 것과 같이 유클리드 호제법을 통해 구할 수 있다.

```java
public int getGcd(int a, int b) {
	int tmp, small, big;

	if(a > b) {
		big = a;
		small = b;
	} else {
		big = b;
		small = a;
	}

	while(small != 0) {
		tmp = big % small;
		big = small;
		small = tmp;
	}

	return big;
}
```  

- a = 8, b = 3 일때
	- big = 8, small = 3, tmp = 2
	- big = 3, small = 2, tmp = 1
	- big = 2, small = 1, tmp = 0
	- big = 1, small = 0
	- 8과 3의 최대공약수는 1이 된다.

## 최소공배수 LCM(Least Common Multiple)
- 최소공배수는 최대공약수를 활용해서 구할수 있다.

```java
public int getLcm(int a, int b) {
	return (a * b) / getGcd(a, b);
}
```  

- a = 8, b = 3 일때 GCD = 1 이므로 최소공배수는 `8*3*1 = 24` 가 된다.

---
## Reference
[[알고리즘] 최대공약수, 최소공배수](https://m.blog.naver.com/PostView.nhn?blogId=writer0713&logNo=221133124302&proxyReferer=https%3A%2F%2Fwww.google.com%2F)  
[[Algorithm] 코드로 최대공약수(GCD)와 최소공배수(LCM) 구하기](https://twpower.github.io/69-how-to-get-gcd-and-lcm)  
[유클리드 호제법이란?](http://lonpeach.com/2017/11/12/Euclidean-algorithm/)  
