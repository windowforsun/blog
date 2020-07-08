--- 
layout: single
classes: wide
title: "[풀이] 백준 1037 약수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '약수가 주어질 때 해당 약수의 N 을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 1과 N을 제외한 모든 N 의 모든 약수가 주어진다.
	- N = 8 일때, 주어진 약수는 2, 4
- 해당 약수만을 보고 N 값을 구하라.

## 입력
첫째 줄에 N의 진짜 약수의 개수가 주어진다. 이 개수는 50보다 작거나 같은 자연수이다. 둘째 줄에는 N의 진짜 약수가 주어진다. 1,000,000보다 작거나 같고, 2보다 크거나 같은 자연수이고, 중복되지 않는다.

## 출력
첫째 줄에 N을 출력한다. N은 항상 32비트 부호있는 정수로 표현할 수 있다.

## 예제 입력

```
2
4 2
```  

## 예제 출력

```
8
```  

## 풀이
- 특정 수 N 에 대한 약수는 크게 2가지 경우로 나눌 수 있다.
	- 약수가 홀수 일때 1 2 4 8
	- 약수가 짝수 일때 1 3 9
- 약수에서 1과 N 이 제외 되기 때문에 주어지는 약수는 다음과 같다.
	- 2 4
	- 3
- 약수가 짝수 일때는 처음값과 마지막 값을 곱하고, 홀수 일때는 중간에 위치한 값의 제곱을 하면 된다.

```java
public class Main {
    private int result;
    private int numCount;
    private int[] numArray;

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
            this.numCount = Integer.parseInt(reader.readLine());
            this.numArray = new int[this.numCount];

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            for(int i = 0; i < this.numCount; i++) {
                this.numArray[i] = Integer.parseInt(token.nextToken());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 정렬된 약수일때 로직이 정상적으로 동작하기 때문에 정렬
        Arrays.sort(this.numArray);


        if(this.numCount % 2 == 0) {
            // 약수의 개수가 짝수일 경우 처음과 끝의 곱이 정답
            this.result = this.numArray[0] * this.numArray[this.numCount - 1];
        } else {
            // 약수의 개수가 홀수일 경우 중간에 위치한 약수의 제곱이 정답
            int half = this.numCount / 2;
            this.result = this.numArray[half] * this.numArray[half];
        }
    }
}
```  

---
## Reference
[1037-약수](https://www.acmicpc.net/problem/1037)  
