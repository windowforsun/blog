--- 
layout: single
classes: wide
title: "[풀이] 백준 1748 수 어이쓰기 1"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'N 번째 까지 수를 이어쓴 자리수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Brute Force
---  

# 문제
- 1 ~ N 까지 숫자를 이어 쓴 자리수를 구한다.
	- N = 12 라면, 123456789101112
	
## 입력
첫째 줄에 N(1≤N≤100,000,000)이 주어진다.

## 출력
첫째 줄에 새로운 수의 자릿수를 출력한다.

## 예제 입력

```
120
```  

## 예제 출력

```
252
```  

## 풀이
- 입력까지 100000000 까지 들어오기 때문에 단순한 반복문으로 풀기에는 무리가 있기 때문에 간단한 규칙을 사용한다.
- 자리수에 따른 숫자 개수와 자리수의 총 수의 개수를 활용한다.
	- 1 자리수의 숫자개수는 9개, 총 자리수는 9개
	- 2 자리수의 숫자개수는 90개, 총 자리수는 180개
	- ...

```java
public class Main {
    public TestHelper testHelper = new TestHelper();

    class TestHelper {
        private ByteArrayOutputStream out;

        public TestHelper() {
            this.out = new ByteArrayOutputStream();

            System.setOut(new PrintStream(this.out));
        }

        public String getOutput() {
            return this.out.toString().trim();
        }
    }

    private int result;
    private int num;

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
            this.num = Integer.parseInt(reader.readLine());
            this.result = 0;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 자리수에서 숫자 개수, 현재 자리수의 숫자 개수, 합산한된 수
        int eachNumCount = 1, totalNumCount = 9, maxNum = 9, remain;

        while (maxNum <= this.num) {
            this.result += eachNumCount * totalNumCount;

            eachNumCount += 1;
            totalNumCount *= 10;
            maxNum += totalNumCount;
        }

        remain = this.num - (maxNum / 10);
        this.result += eachNumCount * remain;
    }
}
```  

---
## Reference
[1748-수 이어쓰기 1](https://www.acmicpc.net/problem/1748)  
