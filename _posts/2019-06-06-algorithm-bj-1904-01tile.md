--- 
layout: single
classes: wide
title: "[풀이] 백준 1904 01타일"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 조건 01 타일로 만들 수 있는 타일의 경우의 수를 구하라'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DP
  - Math
use_math : true
---  

# 문제
- 00, 1 두가지 타일이 있다.
- 이진수 길이가 주어질 때 주어진 2개의 타일로 만들 수 있는 경우의 수를 구하라.
- 예시는 아래와 같다.
	- 이진 수 길이 = 1 일때, 1
	- 이진 수 길이 = 2 일때, 00, 11
	- 이진 수 길이 = 3 일때, 001, 011, 110
	- 이진 수 길이 = 4 일때, 0000, 0011, 1001, 1100, 1111

## 입력
첫 번째 줄에 자연수 N이 주어진다.(N ≤ 1,000,000)

## 출력
첫 번째 줄에 지원이가 만들 수 있는 길이가 N인 모든 2진 수열의 개수를 15746으로 나눈 나머지를 출력한다.

## 예제 입력

```
4
```  

## 예제 출력

```
5
```  

## 풀이
- N개의 길이로 만들 수 있는 타일의 경우의 수는 아래와 같다.

	N|경우의 수|개수
	---|---|---
	1|1|1
	2|11<br/>00|2
	3|111<br/>001<br/>100<br/>|3
	4|1111<br/>0011<br/>1001<br/>1100<br/>0000|5
	5|11111<br/>00111<br/>10011<br/>11001<br/>00001<br/>11100<br/>00100<br/>10000|8
	
- %d(n)% 에 대한 경우의 수는 %d(n-1)% 뒤에 1을 붙인 경우의 수 더하기 %d(n-2)% 뒤에 00을 붙인 경우의 수로 알 수 있다.
- 다시 점화식을 세우면 다음과 같다.
	- $d(n) = d(n-1) + d(n-2) (n>2)$

```java
public class Main {
    private int result;
    private int binaryCount;
    private final int MOD = 15746;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.binaryCount = Integer.parseInt(reader.readLine());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int[] dp = new int[this.binaryCount + 1];

        // 문제에서 주어진 규칙성을 보고 dp 에 이진길이에 대한 만들수 있는 경우의 수를 설정해준다.
        // 길이 1 = 1
        // 길이 2 = 2
        // 길이 3 = 3
        // 길이 4 = 5

        dp[1] = 1;

        if(this.binaryCount >= 2) {
            dp[2] = 2;
        }

        if(this.binaryCount > 2) {
            for(int i = 3; i <= this.binaryCount; i++) {
                dp[i] = dp[i - 1] + dp[i - 2];
                dp[i] = dp[i] % MOD;
            }
        }

        this.result = dp[this.binaryCount];
    }
}
```  

---
## Reference
[1904-01 타일](https://www.acmicpc.net/problem/1904)  
