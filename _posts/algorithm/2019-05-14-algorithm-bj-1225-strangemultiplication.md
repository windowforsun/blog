--- 
layout: single
classes: wide
title: "[풀이] 백준 1225 이상한 곱셈"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '두 수를 다른 방식의 곱셈으로 연산하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- A*B 곱셈 연산을 다른 방식으로 연산한다.
- A에서 한자리, B에서 한자리 수를 뽑아 곱한다.
	- 조합 가능한 모든 경우의 수는 A의 자리수 * B의 자리수가 된다.
- 121 * 34 라는 수식이 있다면 아래와 같이 연산한다.
	- `1*3 + 1*4 + 2*3 + 2*4 + 1*3 + 1*4 = 28`

## 입력
첫째 줄에 A와 B가 주어진다. 주어지는 두 수는 모두 10,000자리를 넘지 않는다.

## 출력
첫째 줄에 형택이의 곱셈 결과를 출력한다.

## 예제 입력

```
123 45
```  

## 예제 출력

```
54
```  

## 풀이
- 이상한 곱셈의 수식을 문제에서 설명한 것과 같이 하게 된다면 10000자리수 * 10000자리수 는 10000*10000 = 100000000 O(n^2) 의 연산을 통해 답을 도출할 수 있다.
- 하지만 수식을 다른 방식으로 묶는다면 최대 O(n) 의 시간복잡도에 정답을 도출해 낼 수 있다.
- `(1 + 2 + 1) (3 + 4) = 1*3 + 1*4 + 2*3 + 2*4 + 1*3 + 1*4 = 28`

```java
public class Main {
    // 출력 결과
    private long result;
    // 입력 받은 수 1
    private String num1Str;
    // 입력 받은 수 2
    private String num2Str;

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.num1Str = token.nextToken();
            this.num2Str = token.nextToken();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int num1Len = this.num1Str.length(), num2Len = this.num2Str.length();
        long num1Sum = 0, num2Sum = 0;
        int tmp;

        // 입력 받은 수를 구성하고 있는 숫자 합산
        for(int i = 0; i < num1Len; i++) {
            tmp = this.num1Str.charAt(i) - '0';
            num1Sum += tmp;
        }

        for(int i = 0; i < num2Len; i++) {
            tmp = this.num2Str.charAt(i) - '0';
            num2Sum += tmp;
        }

        // aab * cc = (a + a + b) * (c + c) = (a * c) + (a * c) + (a * c) + (a * c) + (b * c) + (b * c)
        this.result = num1Sum * num2Sum;
    }
}
```  

---
## Reference
[1225-이상한 곱셈](https://www.acmicpc.net/problem/1225)  
