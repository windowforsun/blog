--- 
layout: single
classes: wide
title: "[풀이] 백준 1182 부분수열의 합"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '수열이 주어질 때 부분수열이 특정 정수를 만족하는 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Brute Force
---  

# 문제
- N 개의 정수로 이루어진 수열이 주어진다.
- 위의 수열의 부분 수열의 합이 특정 정수 S 를 만족시키는 부분수열의 경우의 수를 구하라.

## 입력
첫째 줄에 정수의 개수를 나타내는 N과 정수 S가 주어진다. (1 ≤ N ≤ 20, |S| ≤ 1,000,000) 둘째 줄에 N개의 정수가 빈 칸을 사이에 두고 주어진다. 주어지는 정수의 절댓값은 100,000을 넘지 않는다.

## 출력
첫째 줄에 합이 S가 되는 부분수열의 개수를 출력한다.

## 예제 입력

```
5 0
-7 -3 -2 5 8
```  

## 예제 출력

```
1
```  

## 풀이
- `수열 = [-7, -3, -2, 2 8]` 이 있다고 하자.
- 위 수열에서 만들수 있는 부분수열은 -7, -3 ,-2 등 모든 수열의 원소로 시작 할 수 있다.
- 각 원소로 시작된 부분 수열은 자신으 인덱스 보다 큰 인덱스를 선택하며 부분 수열의 개수를 늘려 나갈 수 있다.
	- 자신의 인덱스보다 작은 원소를 선택하여 부분 수열을 늘릴경우 이는 중복 되는 부분 수열이 된다.
- 첫번째 인덱스인 -7 로 시작하는 부분 수열은 아래와 같다.


|-7|-3|-2|2|8|부분수열의 합
|---|---|---|---|---|:---:|
|o| | | | |-7
|o|o| | | |-10
|o|o|o| | |-12
|o|o|o|o| |-10
|o|o|o|o|o|-2
|o| |o| | |-9
|o| |o|o| |-7
|o| |o|o|o|1
|o| | |o| |-5
|o| | |o|o|3
|o| | | |o|1 

- 재귀함수를 사용해서 위와 같은 탐색을 구현하여 결과값을 도출해낼 수 있다.

```java
public class Main {
    // 출력 결과 저장
    private int result;
    // 입력 받는 정수의 개수
    private int count;
    // 입력 받은 정수 배열
    private int[] numArray;
    // 부분수열의 합이 만족시켜야 하는 합
    private int sum;

    public Main() {
        this.result = 0;
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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            this.count = Integer.parseInt(token.nextToken());
            this.sum = Integer.parseInt(token.nextToken());
            this.numArray = new int[this.count];

            token = new StringTokenizer(reader.readLine(), " ");
            for(int i = 0; i < this.count; i++) {
                this.numArray[i] = Integer.parseInt(token.nextToken());
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 0 번째 인덱스 부터 순자척으로 dfs 호출해서 부분 수열을 만든다.
        for(int i = 0; i < this.count; i++){
            dfs(i, this.numArray[i]);
        }
    }

    public void dfs(int index, int currentSum) {
        // 현재 까지의 부분수열의 합이 만족되면 결과값 카운트
        if(currentSum == this.sum) {
            this.result++;
        }

        // 다음 인덱스로 시작해서 부분 수열을 만든다.
        for(int i = index + 1; i < this.count; i++) {
            dfs(i, currentSum + this.numArray[i]);
        }
    }
}
```  

---
## Reference
[1182-부분수열의 합](https://www.acmicpc.net/problem/1182)  
