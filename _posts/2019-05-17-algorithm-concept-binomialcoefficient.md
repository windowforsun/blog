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

## 이항 계수(Binomial Coefficient) 알고리즘 (mod 연산 제외)
- 아래의 알고리즘들은 mod 연산을 제외한 즉, 아주 큰 수에 대한 고려는 하지 않은 알고리즘이다.

### 직접 계산
- nCk = n! / k!(n-k)!

```java
public class Main {
    public static void main(String[] args){
        Main main = new Main();

        System.out.println(main.solution(20, 4));
    }

    public long solution(int n, int k) {
        long result = 1;

        if(n < k) {
            result = 0;
        } else if(k == 0) {
            result = 1;
        } else {
            for(int i = n, j = 1; j <= k; i--, j++) {
                result *= i;
                result /= j;
            }
        }

        return result;
    }
}
```  

### DP
- nCk = n-1Ck + n-1Ck-1

	n\k|0|1|2|3|4|5
	---|---|---|---|---|---|---
	0|1| | | | | | 
	1|1|1| | | | | 
	2|1|2|1| | | | 
	3|1|3|3|1| | | 
	4|1|4|6|4|1| | 
	5|1|5|10|10|5|1| 
	
- n * k 이차원 배열의 공간 복잡도가 필요하고 O(n*k) 의 시간 복잡도가 소요된다.

```java
public class Main {
    public static void main(String[] args){
        Main main = new Main();

        System.out.println(main.solution(20, 2));
    }

    public long solution(int n, int k) {
        int[][] dp = new int[n + 1][k + 1];
        int min;
        dp[0][0] = 1;

        for(int i = 1; i <= n; i++) {
            min = Integer.min(i, k);

            for(int j = 0; j <= min; j++) {
                if(j == 0) {
                    dp[i][j] = 1;
                } else {
                    dp[i][j] = dp[i - 1][j - 1] + dp[i - 1][j];
                }
            }
        }

        return dp[n][k];
    }
}
```

---
## Reference
[[조합론] 이항계수 알고리즘 3가지](https://shoark7.github.io/programming/algorithm/3-ways-to-get-binomial-coefficients.html)  
[38. 이항계수 구하기 알고리즘 (재귀 프로그래밍)](https://makefortune2.tistory.com/109)  
[이항계수 (nCr) mod p 계산하기](https://koosaga.com/63)  
[페르마의 소정리](https://johngrib.github.io/wiki/Fermat-s-little-theorem/)  

