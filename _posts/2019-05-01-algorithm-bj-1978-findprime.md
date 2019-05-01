--- 
layout: single
classes: wide
title: "[풀이] 백준 1978 소수 찾기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '소수를 찾아 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
  - Sieve of Eratosthenes
---  

# 문제
- 입력 받은 N 개의 수중 소수의 개수를 구하라.

## 입력
첫 줄에 수의 개수 N이 주어진다. N은 100이하이다. 다음으로 N개의 수가 주어지는데 수는 1,000 이하의 자연수이다.

## 출력
주어진 수들 중 소수의 개수를 출력한다.

## 예제 입력

```
4
1 3 5 7
```  

## 예제 출력

```
3
```  

## 풀이
- 입력 받은 숫자들의 소수 체크를 위해 유명한 소수관련 알고리즘인 에라토스테네스의 체 알고리즘을 사용한다.
- 에라토스테네스의 체 알고리즘은 매우 단순한 방식에 기초한다.
	- 2의 배수에서 2를 제외한 배수는 소수가 아니다.
	- 3의 배수에서 3을 제외한 배수는 소수가 아니다.
	- 4는 2의 배수이므로 소수가 아니다.
	- 5의 배수에서 5를 제이한 배수는 소수가 아니다.
	- ...
- 배열에서 위의 경우를 반복하며 소수가 아닌 배열 인덱스를 체크해 주면된다.

```java
public class Main {
    // 출력 결과
    private int result;
    // 총 수의 개수
    private int numCount;
    // 수 중 가장 큰 수
    private int maxNum;
    // 입력 받은 수 배열
    private int[] numArray;
    // 소수가 아닌지 체크
    private boolean[] isNotPrime;

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

        try{
            this.numCount = Integer.parseInt(reader.readLine());
            this.numArray = new int[this.numCount];
            this.result = 0;

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int num;

            for(int i = 0; i < this.numCount; i++) {
                num = Integer.parseInt(token.nextToken());

                if(this.maxNum < num) {
                    this.maxNum = num;
                }

                this.numArray[i] = num;
            }

            this.isNotPrime = new boolean[this.maxNum + 1];

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 가장 큰수의 제곱근 수
        int max = (int)Math.sqrt(this.maxNum);

        // 1은 소수가 아님
        this.isNotPrime[1] = true;

        // 2 부터 시작해서 가장 큰 수의 제곱근 수만큼 반복
        for(int i = 2; i <= max; i++) {

            // 소수가 아니리 경우에는 수행하지 않음
            if(!this.isNotPrime[i]) {
                // i * (2 ~) 값은 소수가 아님
                for(int j = i + i; j <= this.maxNum; j += i) {
                    this.isNotPrime[j] = true;
                }
            }
        }

        // 입력 받은 수 배열 반복
        for(int i = 0; i < this.numCount; i++) {
            // 해당 수가 소수이면 결과값 증가
            if(!this.isNotPrime[this.numArray[i]]) {
                this.result++;
            }
        }
    }
}
```  

---
## Reference
[1978-소수 찾기](https://www.acmicpc.net/problem/1978)  
