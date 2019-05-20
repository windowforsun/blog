--- 
layout: single
classes: wide
title: "[풀이] 백준 1138 한 줄로 서기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 특정 정보로 사람을 한줄로 세워 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Greedy
  - Math
---  

# 문제
- N 명의 사람이 왼쪽부터 한줄로 선다.
- N 명의 사람은 자신의 왼쪽에 자신보다 키가 큰 사람의 수를 기억한다.
- 각 사람들이 기억하는 정보가 주어질 때, 줄을 어떻게 서야 하는지 출력하자.

## 입력
첫째 줄에 사람의 수 N이 주어진다. N은 10보다 작거나 같은 자연수이다. 둘째 줄에는 키가 1인 사람부터 차례대로 자기보다 키가 큰 사람이 왼쪽에 몇 명이 있었는지 주어진다. i번째 수는 0보다 크거나 같고, N-i보다 작거나 같다.

## 출력
첫째 줄에 줄을 선 순서대로 키를 출력한다.

## 예제 입력

```
4
2 1 1 0
```  

## 예제 출력

```
4 2 1 3
```  

## 풀이
- 그리디 알고리즘을 활용하면 간단하게 문제를 해결 할 수 있다.
- 2 명의 사람이 있고 주어진 정보는 0 0 이라고 가정하자.
	- 1번과 2번중 1번을 먼저 줄을 세워야 한다.
- 위의 예시에서 알수 있는 것은 주어진 정보에서 0이 한개 이상 있을 경우 작은 번호를 먼저 선택해야 한다는 것이다.
- 특정 번호를 줄에 세웠으면 그 번호보다 작은 번호들의 정보를 1씩 감소 시켜 정보를 갱신해 주고 이를 반복한다.
	
```java
public class Main {
    // 출력 결과
    private StringBuilder result;
    // 총 인원
    private int count;
    // 자신보다 큰 수에 대한 배열
    private int[] biggerCount;
    // 배치할 인덱스
    private int placeIndex;

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
            this.count = Integer.parseInt(reader.readLine());
            this.biggerCount = new int[this.count + 1];
            this.result = new StringBuilder();

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            for (int i = 1; i <= this.count; i++) {
                this.biggerCount[i] = Integer.parseInt(token.nextToken());

                // 처음에 배치할 인덱스 설정
                if (this.placeIndex == 0 && this.biggerCount[i] == 0) {
                    this.placeIndex = i;
                    this.biggerCount[i] = -1;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int placeCount = 0;

        // 배치한 수가 총 인원수보다 작을 때 까지 반복
        while (placeCount < this.count) {
            placeCount++;

            // 배치할 인덱스 배치 하기
            this.result.append(this.placeIndex).append(" ");

            // 배치한 인덱스보다 작은 숫자에 대해 큰 숫자 수 갱신
            for (int i = 1; i < this.placeIndex; i++) {
                this.biggerCount[i]--;
            }

            // 다음에 배치할 인덱스
            for (int i = 1; i <= this.count; i++) {
                // -1이 아니면서 0인 가장 작은 인덱스
                if (this.biggerCount[i] != -1 && 0 == this.biggerCount[i]) {
                    this.placeIndex = i;
                    this.biggerCount[i] = -1;
                    break;
                }
            }
        }
    }
}
```  

---
## Reference
[1138-한 줄로 서기](https://www.acmicpc.net/problem/1138)  
