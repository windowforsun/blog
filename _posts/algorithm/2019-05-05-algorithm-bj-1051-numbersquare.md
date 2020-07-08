--- 
layout: single
classes: wide
title: "[풀이] 백준 1051 숫자 정사각형"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '직사각형에서 같은 숫자 꼭지점으로 이루어진 가장 큰 정사각형을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Brute Force
---  

# 문제
- N*M 크기 한 자리 숫자가 적힌 직사각형이 주어진다.
- 주어진 직사각형에서 꼭짓점에 적힌 숫자가 모두 같은 가장 큰 정사각형을 구하라.

## 입력
첫째 줄에 N과 M이 주어진다. N과 M은 50보다 작거나 같은 자연수이다. 둘째 줄부터 N개의 줄에 수가 주어진다.

## 출력
첫째 줄에 정답 정사각형의 크기를 출력한다.

## 예제 입력

```
3 5
42101
22100
22101
```  

## 예제 출력

```
9
```  

## 풀이
- 일반 적인 브루트 포스 문제로 최대 50*50 이차원 배열을 모두 돌며 가장 큰 정사각형을 찾으면 된다.
- 최악의 경우 r=50, c=50 이고 최대의 정사각형크기가 50이면 총 `50*50*50` 으로 O(n^3) 이 될 수 있다.
- 중복 연산을 피하고 시간을 대폭적으로 줄일 수 있는 방법은 만들 수 있는 정사각형의 최대 크기가 50이라면 50부터 2까지 시작해서 최대 정사각형의 크기를 찾는 것이다.

```java
public class Main {
    // 출력 결과
    private int result;
    // 행, 렬 수
    private int r, c;
    // 직사각형에 적힌 한자리 숫자 이차원 배열
    private int[][] matrix;

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
            String tmp;

            this.r = Integer.parseInt(token.nextToken());
            this.c = Integer.parseInt(token.nextToken());
            this.matrix = new int[this.r][this.c];
            this.result = 1;

            for (int i = 0; i < this.r; i++) {
                tmp = reader.readLine();

                for (int j = 0; j < this.c; j++) {
                    this.matrix[i][j] = tmp.charAt(j) - '0';
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
        // min 값은 행, 렬 중 작은 숫자
        int num, maxR, maxC, min = this.r < this.c ? this.r : this.c;

        // 주어진 직사각형에서 만들 수 있는 정사각형의 최대 크기는 행, 렬 중 작은 것으로 만들 수 있음
        // 만들 수 있는 가장 큰 크기부터 하나씩 감소하면서 탐색
        for(int k = min - 1; k >= 1; k--) {
            // 현재 크기에서 탐색할 수 있는 행렬 최대 크기
            maxR = this.r - k;
            maxC = this.c - k;
            for(int i = 0; i < maxR; i++) {
                for(int j = 0; j < maxC; j++) {
                    num = this.matrix[i][j];

                    // 4 개의 꼭지점이 모두 같으면
                    if (this.matrix[i][j + k] == num
                            && this.matrix[i + k][j] == num
                            && this.matrix[i + k][j + k] == num) {
                        this.result = (k + 1) * (k + 1);
                        return;
                    }
                }
            }
        }
    }
}
```  

---
## Reference
[1051-숫자 정사각형](https://www.acmicpc.net/problem/1051)  
