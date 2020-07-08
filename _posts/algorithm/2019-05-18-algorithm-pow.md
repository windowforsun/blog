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
use_math : true
---  

## 지수승 알고리즘
### Recursive
- a^n 을 정리하면 아래와 같다.
	- n = 0이면 1을 반환한다.
	- n 이 짝수 이면 (a^(n/2))^2 를 반환한다.
	- n 이 홀수 이면 ((a^(n/2))^2) * a 를 반환한다.
	
### Iterative

```
a^8 = (a^4)^2 
    = ((a^2)^2)^2 
    = (((a^1)^2)^2)^2 
    = a^(2*2*2)
```  

```
a^24 = (a^12)^2
	 = ((a^6)^2)^2
	 = (((a^3)^2)^2)^2
	 = (((((a^1)^2) * a)^2)^2)^2
	 = a^(2*2*2*2 + 2*2*2)
```  

```
a^7 = ((a^3)^2) * a
    = ((((a^1)^2) * a)^2) * a
    = a^(2*2+2+1)
```  


### 소스코드
- Recursive, Iterative 모두 지수승의 절반씩 구하며 답을 도출하고 있다. 시간 복잡도 O(logn)
	- 2^1024 을 구해야 한다면 지수승인 1024는 2^7 이므로 총 7번의 연산으로 답을 도출 할 수 있다.

```java
public class Main {
    public static void main(String[] args) {
        Main main = new Main();

        System.out.println(main.recursive(2, 5));
        System.out.println(main.iterative(2, 5));
        System.out.println(main.recursive(2, 31));
        System.out.println(main.iterative(2, 31));
        System.out.println(main.recursive(2, 32));
        System.out.println(main.iterative(2, 32));
    }

    public long recursive(long a, long n) {
        long result = 1;

        if(n == 0) {
            result = 1;
        } else {
            result = this.recursive(a, n / 2);

            result *= result;

            if(n % 2 == 1) {
                result *= a;
            }
        }

        return result;
    }

    public long iterative(long a, long n) {
        long result = 1, multiply = a;

        while(n > 0) {
            if(n % 2 == 1) {
                result *= multiply;
            }

            multiply *= multiply;

            n /= 2;
        }

        return result;
    }
}
```  

---
## Reference

