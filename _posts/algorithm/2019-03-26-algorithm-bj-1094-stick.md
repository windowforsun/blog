--- 
layout: single
classes: wide
title: "[풀이] 백준 1094 막대기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '고정된 길이의 막대기를 일정한 연산을 통해 특정 길이로 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
  - Simulation
---  

# 문제
- 64cm 막대기가 있는데, 이 막대기를 Xcm 길이로 만들려고 한다.
- Xcm 길이로 만들기 위해서 아래와 같은 동작을 수행할 수 있다.
	1. 현재 가지고 있는 막대기들 중에(맨 처음에는 64cm 한개로 시작) 가장 짧은 막대기를 절반으로 자른다.
	1. 그전에 원래 가지고 있던 막대기들과 절반으로 자른 막대기 2개 중 하나만 더한다.
	1. 막대기 길이를 합산 했을 때 Xcm 보다 크거나 같다면, 위에서 자르고 더하지 않은 2개 중 하나의 막대기를 버린다.
	1. 남은 막대기들을 붙인다.
	1. 남은 막대기를 더해 Xcm 가 될때까지 위 과정을 반복한다.
- 위와 같은 과정을 통해 Xcm 막대기를 구했을 때 몇개의 막대기로 Xcm 를 만들었는지 구하라.

## 입력
첫째 줄에 X가 주어진다. X는 64보다 작거나 같은 자연수이다.

## 출력
문제의 과정을 거친다면, 몇 개의 막대를 풀로 붙여서 Xcm를 만들 수 있는지 출력한다.

## 예제 입력

```
23
```  

## 예제 출력

```
4
```  

## 예제 입력

```
32
```  

## 예제 출력

```
1
```  

## 예제 입력

```
64
```  

## 예제 출력

```
1
```  

## 예제 입력

```
48
```  

## 예제 출력

```
2
```  

# 풀이
- 23cm 의 길이를 만든다면 사용하는 막대기를 트리로 표현하면 아래와 같다.

	![23cm 트리]({{site.baseurl}}/img/algorithm/algorithm-bj-1094-tree.png)

	- 트리에서 주목해야 할 부분은 리프 노드 부분이다.
	- X 가 있는 숫자는 연산에서 버리는 숫자이다.
	- O 가 있는 숫자는 막대기에서 사용하는 숫자이다. 
- 가지고 있는 막대의 길이 중 최소를 M, 선택된 막대기들의 길이 합을 S 라고 한다면 연산 과정은 아래와 같다.
	1. S + (M / 2) < X 일 경우 S = S + (M / 2) 로 S 값에 더 해주고, 사용한 막대기 개수를 1 증가시킨다.
	1. S == X 가 될때 까지 위 연산을 반복해 준다.

```java
public class Main {
    // 막대기 길이
    final int MAX = 64;
    // 출력 결과값 (사용한 막대기 개수)
    private int result;
    // 만들어야 하는 막대기 길이
    private int stickLen;

    public Main() {
        this.result = 0;
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.solution();
        main.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.stickLen = Integer.parseInt(reader.readLine());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int totalLen = 0, currentLen = MAX, stickSum;

        // 만들어야 하는 막대기 길이가 처음 막대기 길이와 같다면 결과 값 설정 후 리턴
        if(this.stickLen == MAX) {
            this.result = 1;
            return;
        }

        // 현재 막대기들의 길이 합이 만들어야 하는 길이와 다를 때까지 반복
        while(totalLen != this.stickLen) {
            // 현재 막대기 길이의 절반
            currentLen /= 2;
            // 전에 가지고 있던 막대기와 자른 막대기 2개중 한개만 합산
            stickSum = currentLen + totalLen;

            // 만들어야 하는 막대기보다 같거나 작으면
            if(stickSum <= this.stickLen) {
                // 전에 선택된 막대기 길이에 더해주기
                totalLen += currentLen;
                // 막대기 개수 카운트 증가
                this.result++;
            }
        }
    }
}
```  

---
## Reference
[1094-막대기](https://www.acmicpc.net/problem/1094)  
