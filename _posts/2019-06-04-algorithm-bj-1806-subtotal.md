--- 
layout: single
classes: wide
title: "[풀이] 백준 1806 부분합"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 그래프에서 SCC 집합을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Two Pointers
  - Sliding Window
use_math : true
---  

# 문제
- 10000 이하로 이루어진 N 길이의 수열이 주어진다.
- 주어진 수열에서 연속된 수들의 부분합 중 그합이 S 이상이 되면서, 가장 짧은 수열의 길이를 구하라.

## 입력
첫째 줄에 N (10 ≤ N < 100,000)과 S (0 < S ≤ 100,000,000)가 주어진다. 둘째 줄에는 수열이 주어진다. 수열의 각 원소는 공백으로 구분되어져 있으며, 10,000이하의 자연수이다.

## 출력
첫째 줄에 구하고자 하는 최소의 길이를 출력한다. 만일 그러한 합을 만드는 것이 불가능하다면 0을 출력하면 된다.

## 예제 입력

```
10 15
5 1 3 5 10 7 4 9 2 8
```  

## 예제 출력

```
2
```  

## 풀이
- 문제는 별다른 알고리즘없이 $O(N^2)$ 의 시간복잡도로 해결할 수 있다.
- [Two Pointers]({{site.baseurl}}{% link _posts/2019-05-27-algorithm-concept-twopointers.md %}) 알고리즘을 사용하면 $O(N)$ 의 시간복잡도로 문제 해결이 가능하다.

```java
public class Main {
    private int result;
    private int[] array;
    private int arraySize;
    private int s;

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

            this.arraySize = Integer.parseInt(token.nextToken());
            this.array = new int[this.arraySize];
            this.s = Integer.parseInt(token.nextToken());
            this.result = Integer.MAX_VALUE;
            token = new StringTokenizer(reader.readLine(), " ");

            for(int i = 0; i < this.arraySize; i++) {
                this.array[i] = Integer.parseInt(token.nextToken());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int start = 0, end = 0, sum = 0;

        while(true) {
            // 만족하는 최소 길이 구하기
            if(sum >= this.s) {
                this.result = Integer.min(this.result, end - start);
            }

            if(sum > this.s) {
                sum -= this.array[start++];
            } else if(end == this.arraySize) {
                break;
            } else {
                sum += this.array[end++];
            }
        }

        // 만족하는 길이가 하나도 없을 때 처리
        if(this.result == Integer.MAX_VALUE) {
            this.result = 0;
        }
    }
}
```  

---
## Reference
[1806-부분합](https://www.acmicpc.net/problem/1806)  
