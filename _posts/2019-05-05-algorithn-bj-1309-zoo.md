--- 
layout: single
classes: wide
title: "[풀이] 백준 1309 동물원"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '동물원 우리에서 사자를 배치하는 경우의 수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DP
---  

# 문제
- N*2 인 우리가 있을 때 사자를 배치하는 경우의 수를 구하려고 한다.
- 사자는 가로, 세로로 붙어 있게 배치할 수 없다.
- 위의 조건이 만족하는 N*2 크기의 우리에 사자를 배치하는 모든 방법을 구하라.
- 사자를 한마리도 배치하지 않는 것도 하나의 경우로 생각한다. 

## 입력
첫째 줄에 우리의 크기 N(1≤N≤100,000)이 주어진다.

## 출력
첫째 줄에 사자를 배치하는 경우의 수를 9901로 나눈 나머지를 출력하여라.

## 예제 입력

```
4
```  

## 예제 출력

```
41
```  

## 풀이
- 1*2 의 우리에서 사자를 배치하는 경우의 수는 총 3가지 이다.
	- 2칸의 우리 중 사자가 한마리도 없는 경우 1가지 = none
	- 2칸의 우리 중 사자가 한마리만 있는 경우 2가지 = one
- 2칸의 우리 중 사자가 한마리도 없는 경우에서 다음 칸이 가질 수 있는 경우의 수는 one * 2 + none 이 된다.
- 2칸의 우리 중 사자가 한마리만 있는 경우에서 다음 칸이 가질 수 있는 경우의 수는 one + none 이 된다.
- 이를 점화식으로 표현하면 one(n) = one(n-1) + none(n-1) 이되고, none(n) = (one(n-1) * 2) + none(n-1) 이 된다.

```java
public class Main {
    private int result;
    private int row;
    private final int MAX = 9901;

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
            this.row = Integer.parseInt(reader.readLine());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int useOne = 2, useNone = 1, beforeOne, beforeNone;

        // 2개 우리중 하나에만 사자가 있는 경우
        beforeOne = useOne;
        // 2개 우리 모두 사자가 없는 경우
        beforeNone = useNone;

        // 입력 받은 row 변째 까지 연산 수행
        for(int i = 1; i < this.row; i++) {
            // i 번째 하나의 우리에 사자가 있는 경우의 수 = 이전 한마리 경우의 수 + (이전 0마리 경우의 수 * 2)
            useOne = beforeOne;
            useOne += (beforeNone * 2);

            // i 번째 우리에 사자가 한마리도 없는 경우의 수 = 이전 한마리 경우의 수 + 이전 0마리 경우의 수
            useNone = beforeOne;
            useNone += beforeNone;

            useOne = useOne % MAX;
            useNone = useNone % MAX;

            beforeOne = useOne;
            beforeNone = useNone;
        }

        this.result = (useOne + useNone) % MAX;
    }
}
```  

---
## Reference
[1309-동물원](https://www.acmicpc.net/problem/1309)  
